# Restoration

Depositors can request restoration of files or objects through the Pharos Web UI. Clicking the __Restore__ button for a file restores a single file; for an object, it restores all files belonging to that object in BagIt format.

When the user clicks __Restore__:

1. Pharos creates a WorkItem saying that user X has requested restoration of file or object Y.
2. The `apt_queue` cron job, which runs every 10 minutes or so, copies the WorkItem ID of the restoration request into the appropriate topic in NSQ.
3. NSQ passes the WorkItem ID to a worker to fulfill the request.

## Restoring Files

The file restoration worker performs the following tasks:

1. After receiving the WorkItem ID from NSQ, it retrieves the WorkItem from Pharos.
2. The worker retrieves the GenericFile record from Pharos of the file to be restored.
3. The worker copies the file directly from preservation storage to the depositor's receiving bucket.
4. The worker marks the WorkItem complete, and includes in the WorkItem note the URL from which the depositor may retrieve the file.
5. The worker marks the NSQ message as finished.

## Restoring Objects

Object restoration is more complex, since restoring an object requires retrieving and packaging multiple files before sending the entire restored package to the depositor's restoration bucket.

The object restoration worker performs the following tasks:

1. After receiving the WorkItem ID from NSQ, it retrieves the WorkItem from Pharos.
2. The worker retrieves the IntellectualObject record from Pharos of the object to be restored.
3. The worker retrieves each GenericFile record belonging to that object from Pharos.
4. For each active GenericFile (where state = "A" and not "D"), the worker copies the file from preservation storage to a staging area for bagging. Note that, other than bagit.txt, APTrust preserves tag files in addition to payload files.
4. The worker validates the actual checksums of each file during the download to staging. If any file cannot be found, or if any file's checksum does not match the most recent checksum in Pharos, the restoration fails.
5. The worker bags all of the files (including perserved tag files) according to the profile used when the bag was initially ingested. The profile information is part of the IntellectualObject record, and may be APTrust or Beyond the Repository (BTR). As part of the bagging process, the restoration worker creates manifests and tag manifests.
6. The worker requests a list from Pharos of all PREMIS events pertaining to this object. It includes this list in the bag as a JSON file. The primary reason for this is so depositors can see that some items in the bag may have been deleted or re-ingested while APTrust had custody of the object.
7. The worker validates the bag.
8. The worker copies the bag to the depositor's receiving bucket.
9. The worker marks the WorkItem complete, and includes in the WorkItem note the URL from which the depositor may retrieve the file.
10. The worker marks the NSQ message as finished.

Note that because APTrust rebags objects when it restores them, the restored bag will not be byte-for-byte identical to the initially ingested bag. We do guarantee that the payload will be complete, and that the new bag will include the tag files from the original (except for bagit.txt, which we reconstitute), the restored bag can have the following differences:

1. Items in the manifest may be listed in a different order than the original.
2. The restored bag may include additional manifests or tag manifests not present in the original bag.
3. The restored bag will not include files that were deleted during the object's time in APTrust's custody.

## Restoring from Glacier

Glacier restorations follow the steps outlined above for files and objects, with these additional steps at the beginning:

1. The WorkItem IDs of Glacier items first go into an NSQ topic called `apt_glacier_restore_init`.
2. The `apt_glacier_restore_init` worker issues one Glacier restore request per file to the vault holding the files.
3. The worker checks the status of the restore requests every few hours.
4. When all Glacier files have been copied to S3, the Glacier restore worker pushes the WorkItem ID into the normal NSQ restoration topic. From there, the restoration proceeds as described above.
