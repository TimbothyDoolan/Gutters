# Jobs and storage

<!-- Split out of the root CLAUDE.md, which points here. Read before working on this area. -->

**Jobs** (JOBS banner; top bar's Jobs button → `openJobs()`). The open job still autosaves to
`localStorage["primeGutterEstimate"]`; every job with something in it is also kept in IndexedDB (same
`primeGutterPhotos` DB, v2): its JSON in `IDB_JOBS`, a list record in `IDB_META` (`jobMeta()`: name, address,
total, updated, `blobs`), both keyed by `curJobId` and written in one transaction by `storeJob()` from every
`saveLocal()`. `curJobId` is serialized as top-level `jobId`, deliberately not in `P.meta` (undo swaps meta).
Only the open job's images are in memory (`idbLoadKeys(jobBlobIds())`); `idbGC()` keeps a blob while any listed
job's record names it and deletes nothing if the list can't be read. New job / opening another job only drop
blobs from memory, never from IndexedDB. A backup file always opens as a new job (`showLoadedJob(d,null)`), so
an old backup can't overwrite a newer listed job. "Save" is now Jobs → Export this job (`exportBackup()`).
