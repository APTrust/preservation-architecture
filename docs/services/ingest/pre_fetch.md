# Pre-Fetch

The pre-fetch worker, `ingest_pre_fetch`, streams a tarred bag from a receiving bucket through a tar reader to extract metadata. All of the extracted metadata is stored in Redis, where subsequent workers can access it.

The pre-fetch worker collects the following data as it reads the bag:

* the URL of the bag's BagIt profile (if not found, defaults to APTrust)
* file names
* file sizes
* a first guess at the file's format, based on extension
* checksums read from manifests (and or all of md5, sha1, sha256 and sha512)
* checksums calculated from files (md5, sha1, sha256, sha512)
* tag names and values parsed from bagit.txt, bag-info.txt, aptrust-info.txt, and any other text-based non-payload files that use BagIt tag file format

The pre-fetch worker requeues items with transient errors. When it encounters fatal errors, it marks the WorkItem as Failed.

Transient errors are almost always network errors, such as "connection reset by peer." Fatal errors include invalid file format (i.e. the bag is not really a tar file) and "key not found," which means the bag was deleted from the recieving bucket before the worker could get to it.

The pre-fetch will mark the WorkItem as cancelled and not process it if the etag of the bag in the receiving bucket does not match the etag in the WorkItem. This happens when the depositor uploads a new version of the bag, overwriting the version to which the original WorkItem referred.

For information about how the pre-fetch worker stores information, see [Redis](../../components/redis.md).

## Resource Usage

The pre-fetch worker:

* Uses a lot of bandwidth downloading large bags or large quantities of small bags from S3 receiving buckets. It does not write the data to disk, but it does pull it across the network.
* Uses a noticeable amount of CPU while calculating checksums.
* Makes many calls to Redis, mostly writes, to save JSON data about the object and each file in the bag.
* Makes a few calls to Pharos, mainly to read and update a WorkItem, but generally does not tax Pharos.
