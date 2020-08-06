# Metadata Recording

After storage validation is complete, the record worker records all information about the ingest in Pharos. It copies the interim object record and all of the interim file records from Redis to Pharos. It also creates all required PREMIS events for the ingest.

For bags containing large numbers of files, this step can be very taxing on Pharos. Each ingest produces the following number of records:

* 1 intellectual object creation or update
* 3 object-related PREMIS events
* 1 generic file creation or update _per file_
* 6 PREMIS events _per file_
* 3 checksum creations _per file_

A new bag containing 10,000 files would create 100,004 new records through Pharos (4 for the object and 10,000 * 10 for the files). While the file records are recorded in batches, the operation is still taxing.

Like other workers, the recorder marks each object and file as saved once Pharos returns a successful response, and then saves each object/file back to Redis. If recording fails before completion, the next worker to pick up the task will know what's been saved and what hasn't.

## Resource Usage

* One read and one update for every interim file record in Redis.
* One read and one update of the Pharos WorkItem.
* __Heavy__ POST and/or PUT operations to Pharos to record file, checksum, and event data.

In general, we should not run more than one record worker to avoid overwhelming Pharos during heavy ingest. The single record worker can be tuned to use a set number of workers to utilize Pharos to its limits without overwhelming it.
