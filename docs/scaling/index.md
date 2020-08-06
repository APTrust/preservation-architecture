# Overview

This page describes considerations for scaling each of our services. All scaling is horizontal, meaning we simply run services on more servers to handle increasing loads.

Most of the time, a single server running all services (ingest, fixity, deletion and restoration) is fine. During ingest floods and periods of heavy restoration, we need to scale only specific micro services, as described below.

When adding new instances, we must ensure that workers on the instance have network access to the following services:

* Pharos, for getting and recording WorkItems.
* NSQ, for knowing which items to work on.
* Redis, for accessing and preserving interim processing data.

Keep in mind that Pharos, Redis, and NSQ may be running on a private subnet or VPN. The new servers must be able to reach that subnet or connect to that VPN.

In addition, additional servers to handle temporary load spikes should be in the US East region for fast access to primary preservation storage, which is in S3 in US East.

## Ingest

Extensive profiling has shown us that bandwidth is the bottleneck for ingest operations. The [pre-fetch worker](../services/ingest/pre_fetch.md), [staging uploader](../services/ingest/staging.md) and [preservation uploader](../services/ingest/storage.md) all use substantial bandwidth, and all should be scaled horizontally during periods of heavy ingest.

[Format identification](../services/ingest/format_identification) uses at least a noticeable amount of bandwidth and CPU. Additional servers may be relieve pressure from the primary server during heavy ingest.

The [validation](../services/ingest/validation.md), [storage validation](../services/ingest/storage_validation.md), and [cleanup](../services/ingest/cleanup.md) workers use few resources and would generally not benefit from additional instances.

The [reingest worker](../services/ingest/reingest_check.md) may flood Pharos with requests when reingesting bags with many files, and the [metadata recording worker](../services/ingest/metadata_recording.md) taxes Pharos on every ingest. For these reasons, __we should not run more than one instance of either of these workers.__ Instead, we can tune the number of Go routines within a single instance of each worker to ensure they utilize Pharos to its maximum capacity without overloading it.

## Fixity Checking

Because fixity checking is handled by a single worker, we can scale up by starting a new EC2 instance and running an additional fixity checker there. Because fixity checking is bandwidth-intensive, consider choosing an instance type with medium or better network IO. Micro and small instances will be of little help.

## Deletion

Deletion scaling may never be necessary. The only time we've ever had deletion backlogs have been after depositors request bulk deletion of tens of thousands of files. Even then, the depositor doesn't care if it takes ten minutes or 24 hours for the deletion to complete.

## Restoration

TBD. The third-generation restoration code isn't written yet.
