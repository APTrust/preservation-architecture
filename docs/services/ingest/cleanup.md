# Cleanup

After a bag has been ingested, the cleanup worker does the following:

* deletes all of the interim data associated with the WorkItem from Redis
* deletes all of the bag's interim files from the staging bucket
* deletes the original tarred bag from the depositor's receiving bucket
* marks the WorkItem complete

## Resource Usage

The cleanup worker uses few resources of any kind.
