# Reingest Check

The reingest worker checks Pharos to see if the bag being ingested has ever been ingested before. If Pharos has a record for a bag with the same name, belonging to the same institution, and the bag record is not marked deleted (state == 'D'), then we're reingesting an existing bag.

In this case, the reingest worker checks to see if Pharos has a non-deleted record for each file in the bag being reingested. If so, the reingest worker:

* Assigns the UUID of the existing Pharos file to the interim IngestFile record in Redis. We do this because, if we're going to move the new file into preservation storage, we want it to overwrite the old one. To do that, it must have the same UUID (S3 key) as the existing file. We also do not want to store multiple copies of any file under multiple UUIDs in preservation storage, because we have no way of tracking more than one perservation UUID per file.
* Checks whether the checksum of the new file matches the checksum of the existing file as reported by Pharos. If the checksums do not match, the new file is marked as needing to be copied to preservation. If the checksum has not changed, the file is marked as not needing to be copied.
* Forces the Storage Option of all files to match the Storage-Option of existing files in Pharos so that we do not wind up with mismatched versions of files in different S3 buckets, Glacier vaults and Wasabi buckets. The rationale for this is [documented in our user guide](https://aptrust.github.io/userguide/bagging/#allowed-storage-option-values) and has been explained verbally to depositors at our tech and in-person meetings.

The reingest worker updates object and file records in Redis to record the results of its work.

## Resource Usage

For new bags, the reingest worker uses few resources. It reads and updates a Pharos WorkItem, makes one query to the Pharos object endpoint, and reads and updates one object record in Redis.

For reingests, it does all of the above plus the following:

* makes two queries to Pharos for each file in the bag: one for the GenericFile record, and one for the file's checksums
* makes one read request and one write request to Redis for each file

Since most bags contain between 5 and 20 files, this typically isn't too bad. Some bags, however, contain hundreds of thousands of files. In those more extreme cases, this worker may put a heavy load on Pharos and Redis.
