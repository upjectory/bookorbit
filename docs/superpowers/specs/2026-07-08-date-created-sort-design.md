# Design: "Date Created" sort for BookOrbit

**Date:** 2026-07-08
**Status:** Approved (design), pending implementation plan
**Author:** Alex Labbett (with Claude)

## Problem

BookOrbit's "Recently Added" sort orders books by `books.added_at`, which is
stamped via `.defaultNow()` at the moment the scanner first ingests a file — i.e.
when the library was set up on the server, **not** when the book was originally
acquired. For a library bulk-imported to a new server, every book gets roughly the
same `added_at`, so "Recently Added" carries almost no signal and cannot reproduce
the true chronology of the collection.

Concrete example: an epub whose real file date is 2010-06-04 displays as added
"Jul 4, 2026" in BookOrbit — the day it landed on the VPS.

## Key findings (from source + a live install)

- `books.added_at` — `defaultNow()` at first scan. Powers the existing "Date Added"
  sort (`books_library_added_at_idx`). Immutable after insert (file writes only
  touch `fileHash`/`updatedAt`/`mtime`), but records ingest time, not file age.
- `book_files.mtime` — the real filesystem modify time, captured during scanning
  (`file-event-processor.ts` `statToFileInfo` → `s.mtime`). On the target install
  this correctly holds the original 2010-era dates because the Dropbox→VPS transfer
  preserved `mtime`.
- `birthtime` (true creation time) is **not** a viable source: unsupported/unreported
  on the target filesystem, and reset to copy-time on transfer. Discarded.
- Embedded dates (`calibre:timestamp`, `dc:date`) are the most durable *in principle*
  (survive copies) but BookOrbit only extracts `publishedYear`, and very few of the
  target library's files passed through Calibre. Discarded for this feature.
- **Fragility of `mtime`:** with `fileWriteEpubEnabled` (default `true`), in-app
  metadata edits rewrite the epub (temp file + `rename`, no `utimes` restore); the
  next scan overwrites `book_files.mtime`. The original date would be lost. The
  target install has write-back **disabled**, so `mtime` is currently pristine — but
  we must not depend on it staying that way.

## Decision

Capture the currently-pristine `mtime` into a **new immutable snapshot column**
and sort on that. This freezes the true dates now, independent of any future
write-back or rescan.

- **Source:** `book_files.mtime` at first scan.
- **Storage:** new nullable column `book_files.original_date` (timestamptz),
  written once at insert, never updated.
- **Sort:** new `dateCreated` sort field, `COALESCE(original_date, added_at)`.
- **Naming:** honest column name (`original_date`); user-facing label
  **"Date Created"**. The PR description will state the source is file `mtime` at
  ingest, so a maintainer can adjust only the label without touching schema/logic.

**Intent:** one-off personal build for the author's VPS, but kept upstream-clean
(honest schema, migration, tests) so it can be proposed as a PR later without rework.

## Design

### 1. Data model + migration

New column on `book_files`:

```
original_date  timestamptz  NULL
```

Drizzle migration:

1. `ALTER TABLE book_files ADD COLUMN original_date timestamptz;`
2. Backfill from the currently-pristine mtime:
   `UPDATE book_files SET original_date = mtime WHERE original_date IS NULL;`

The backfill freezes existing 2010-era dates on first boot of the new image.

### 2. Scanner change (immutability by construction)

Set the snapshot only at insert, in `ScannerRepository.createBookFile`'s insert
values: `originalDate: data.mtime`. `updateBookFile` takes a partial and is never
passed `originalDate`, so rescans and write-backs update `mtime` but never
`original_date`. New files added after the snapshot get `original_date = mtime` at
their first scan, which is correct for them.

### 3. Backend sort

- Add `"dateCreated"` to the `SortField` union and `SORT_FIELDS` array in
  `packages/types/src/query.ts`.
- Add a case to `BookSortBuilder.appendField`
  (`server/src/modules/book/book-sort-builder.service.ts`), mirroring the existing
  `fileSize`/`format` correlated-subquery pattern, with fallback so null snapshots
  still sort sanely:

```ts
case 'dateCreated':
  result.push(sql.raw(
    `(SELECT COALESCE(bf.original_date, books.added_at)
        FROM book_files bf WHERE bf.id = books.primary_file_id) ${D} NULLS LAST`));
  break;
```

### 4. Frontend

Add "Date Created" to the sort dropdown options + label map in the Vue client,
alongside the existing sorts (Title, Date Added, etc.). Jump-bucket (year grouping
in the fast-scroll bar) is **out of scope for v1** (YAGNI).

### 5. Deployment (author's VPS)

Clone fork → this branch → build a custom Docker image → point `docker-compose.yml`
at it. Drizzle migrations run on startup; the backfill captures the dates on first
boot of the new image.

### 6. Tests

- Unit test for the new sort case in the sort-builder (asserts the SQL fragment,
  direction, and `NULLS LAST` handling).
- Migration/backfill assertion: after migration, `original_date = mtime` for
  existing rows.

## Out of scope

- Reading embedded (`calibre:timestamp`/`dc:date`) dates.
- `birthtime` capture.
- Preserving `mtime` across in-app write-back (`utimes` restore).
- Jump-bucket year grouping for the new sort.

## Risks / notes

- If some files' `mtime` was reset during transfer (a clump near install date),
  those rows snapshot the wrong date. Mitigation: audit the `mtime` distribution
  on the VPS (`GROUP BY date_trunc('year', mtime)`) before deploying; handle any
  reset clump separately. This audit is a pre-deploy check, not part of the code.
- Correlated-subquery sort has no dedicated index on `original_date` (matches the
  accepted `fileSize`/`format` precedent). An index can be added later if needed.
