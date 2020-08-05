# Fixity Checking

APTrust checks fixity on all files in S3 preservation storage approximately every 90 days. We do not check fixity on files stored in Glacier because the process of restoring Glacier files is cumbersome and expensive.

While our service contract with depositors says we will check fixity on all S3 files every 90 days, this may occasionally slip to 91 or 92 days in practice because during ingest floods, we prioritize ingest over fixity checking. In the second generation of APTrust (Exchange), we were not able to scale services horizontally and sometimes turned off fixity checking for 1-2 days so ingest services could use all of the server's CPU and bandwidth. This should not be an issue with the third-generation code.

Because ingest floods tend to occur in regular cycles (before the Spring and Fall meetings, and at the end of the calendar year), fixity check floods tend to coincide with ingest floods. That is, while ingesting a million new files, we may also have to run fixity checks on the million files we ingested 360 days ago. Again, this should be less of a problem in the third-generation system.

## Fixity Check Process

1. A cron job called `apt_queue_fixity` runs every 30 minutes on one of our EC2 instances querying Pharos for files that have not had a fixity check in the past 90 days. It sends the identifiers of up to 2500 files on each run into an NSQ topic called `apt_fixity_topic`.
1. A worker called `apt_fixity_check` picks up the file identifier from NSQ and retrieves the GenericFile record from Pharos. This record contains, among other things, the URL of the file in preservation storage, and the most current md5 and sha256 digests for the file.
1. The worker retrieves the file from S3 preservation storage, streaming it through Golang's md5 and sha256 hash functions. The reads are fast because the data never touches the disk. It goes from S3 through the hash functions and into /dev/null.
1. The worker compares each fixity value (md5 and sha256) to the known fixity values in the GenericFile record it got from Pharos.
1. The worker records one fixity check PREMIS event in Pharos for each digest. In each case, if the digest matches, it records a successful event. If the digest does not match, it records a failed event, along with the expected and actual checksums.

The fixity check process is fast and efficient, often processing over 100,000 files per day with little noticeable effect on CPU or memory.

Fixity checking is the only operation on preservation files that does not produce a WorkItem record in Pharos. (Ingests, deletions, and restorations do produce WorkItems.) Fixity checks produce only PREMIS events.
