# Storage Validation

After the preservation uploader copies files to preservation storage, the verifier queries the S3 bucket to ensure that each file is present and that the size of each matches the size recorded in the Redis interim data. We don't currently check for etag matches, because etags can change, depending on how files are chunked in multipart uploads.

We include this step because of a peristent bug in our second-generation services in which AWS S3 servers reported that files were copied even when zero bytes were written.

The verified marks each file record as verified and saves the information back to Redis. If any file fails verification, the verifier will mark the WorkItem as failed, and it will be up to the APTrust administrator to manually requeue the item (through the Pharos UI) in the `ingest06_storage` topic.

The requeue process here is not automated, because we want the APTrust admin to look into the issue. (As of August, 2020, storage validation has not failed on the 100,000+ files we've pushed through our staging system.)

## Resource Usage

This worker issues a potentially large number of stat/head requests to S3, sending and receiving only a kilobyte or so of information in each request. It uses very little CPU or memory.
