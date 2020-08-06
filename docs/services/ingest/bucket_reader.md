# Bucket Reader

The `ingest_bucket_reader` is a cron job that scans receiving buckets for newly uploaded items. For each item, it checks Pharos to see if an ingest WorkItem with matching bucket, key, and etag exists. If not, the bucket reader creates it and adds the new WorkItem ID to NSQ's ingest pre-fetch topic.

While we could have set up an S3 trigger on the receiving buckets to do this work, we chose not to for a few reasons. First, during our scheduled maintenance windows, Pharos can be offline for two hours or more. In addition, during routine deployments, Pharos may be unavailable for a few seconds. In both cases, data from the S3 triggers would be lost, as there would be no endpoint to receive them.

Second, to ensure our system is portable and not dependent on vendor-specific features, we wanted to avoid AWS-specific lambda triggers that may not port to open-source S3-compliant systems like Minio or even to other cloud providers like Wasabi.

Third, the cron job provides a simple mechanism for throttling ingest during periods of heavy load. It's easier to change the timing on a single cron job than to suspend and then re-enable triggers in twenty or so S3 buckets.
