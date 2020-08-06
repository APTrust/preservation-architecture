# Format Identification

The format identification worker streams a chunk of each file in the staging bucket through the [Siegfried](https://www.itforarchivists.com/siegfried/) format identifier, which compares file signatures to a PRONOM database. Because Siegfried is written in Go, it's embedded in the worker.

If Siegfried can identify the file by signature, the worker stores the format information in the file's interim record in Redis. If Siegfried cannot identify the file by signature, the worker leaves the file's extension-based format identification (noted by the pre-fetch worker) unchanged.

## Resource Usage

The format identification worker:

* can use substantial amounts of nework IO, primarily in reads from the S3 staging bucket.
* can use substantial CPU when identifying a large number of files (e.g. if a bag contains 10,000 files)
* makes one read and one update call to Pharos' WorkItems endpoint
* makes one read and one update call _per file_ to Redis
