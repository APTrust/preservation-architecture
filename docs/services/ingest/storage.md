# Copy to Preservation

The `apt_preservation_uploader` worker copies files from the staging bucket to preservation buckets. For new bags, it copies all files. For reingests, it copies only those files that are new or have been changed since the last ingest.

This worker uses server-side copying (direct S3-to-S3 copies that use none of our EC2 instance's bandwidth) when copying from staging to any other AWS bucket in the same region (US East 1). For all other uploads, it streams each file down from the staging bucket and then back out to the preservation bucket. The bits never touch local disk.

Although we could use server-side copying for all AWS-hosted preservation buckets, we've chosen not to because AWS throttles cross-region S3-to-S3 copies at 50 MiB/second, with performance often below that. Streaming the data through the EC2 instance is typically six times faster (or more). This makes a big difference on large files, which take hours instead of days to copy.

In addition to copying file contents into preservation storage, the uploader also copies the following metadata:

* __Content-Type__ - The file's mime type, as identified by Siegfried or extension.
* __x-amz-meta-institution__ - The identifier of the institution that owns the bag/intellectual object. E.g. virginia.edu, emory.edu, etc.
* __x-amz-meta-bag__ - The name of the bag to which the file belongs. E.g. virginia.edu/bag-of-photos.
* __x-amz-meta-bagpath__ - The path this file occupied within the bag.
* __x-amz-meta-md5__ - The file's md5 digest, as caculated at ingest.
* __x-amz-meta-sha256__ - The file's sha256 digest, as calculated at ingest.

## Resource Usage

The preservation uploader:

* uses a lot of bandwidth for both downloading from staging and uploading to preservation.
* uses a noticeable amount of CPU (for on-the-fly etag calculation) and memory (for read/write buffers).
* makes one read and one write call to Redis for each file
* makes one read one update call to Pharos for the WorkItem
