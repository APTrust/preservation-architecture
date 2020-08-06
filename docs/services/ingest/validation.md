# Validation

The validation worker validates the metadata that the pre-fetch worker stored in Redis. It checks that:

* The bag has all tag files required by the BagIt profile.
* The bag has all required tags with valid values, as defined by the BagIt profile..
* All files listed in the manifests and tag manifests are present.
* All file checksums match checksums in the manifests.
* There are no extraneous files in payload directory.

If the bag passes validation, the worker updates the WorkItem in Pharos and pushes the WorkItem ID into the next NSQ topic, which is the reingest check topic.

If there are transient errors, which are almost always network errors, the worker requeues the item and notes the transient errors in the Pharos WorkItem.

If the bag is invalid, that's a fatal error. The worker marks the WorkItem as failed, including a description of why it failed validation, then it pushes the item to the Cleanup queue.

## Resource Usage

The validation worker normally completes quickly without taxing any systems or resources too heavily. It:

* Makes many read calls to Redis, including one for the object and one for each of its files.
* Reads and updates the WorkItem in Pharos.
