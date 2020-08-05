# APTrust Preservation Services Architecture

This site describes the architectural components and general workflow of APTrust's third-generation preservation services. As of August, 2020, APTrust's production and demo repositories are running on our second-generation services suite, code-named Exchange.

The services described in these documents are currently being tested in our staging environment. In replacing our existing, stable, and heavily patched second-generation services, the goals of the new service suite include:

* Modularize code to separate concerns
* Improve code clarity and maintainability
* Improve test coverage
* Improve horizontal scalability by storing interim processing information in network accessible services instead of on local disk
* Support BagIt profiles other than APTrust
* Support S3-compliant storage providers other than AWS
* Eliminate reliance on expensive and problematic EBS volumes

The new preservation architecture also supports the primary goals of the prior architecture, which include:

* __Resilience__ - The system must be able to recover from failures and resume whatever processing was occuring at the time of failure without any loss of information.
* __Reliability__ - The system must process all valid requests for ingest, deletion, fixity checking, restoration to completion.
* __Accountability__ - The system must maintain a record of all actions performed on all objects and files. It must also record the existence and outcome of all tasks that were requested but could not be completed.
* __Security__ - All files in preservation storage must be unreachable except through the administrative interface of Pharos. Destructive actions, such as deletions, require multiple approvals from depositors.
* __Transparency__ - Depositors must be able to see the present state of any item, ingested or not, that is within the normal flow of processing. Administrators must be able to see all that and information about abnormal or failed states.
* __Service Provider Agnosticism__ - APTrust services must, whenever possible, avoid relying on vendor-specific technologies. The entire system should be portable to any cloud provider or to any data center that runs Linux servers. Preservation services adhere to this principle by being open source, written in a cross-platform language (Go), and relying only on open source services such as Redis and NSQ, and standardized or de facto protocols and APIs such as HTTPS and S3.

Performance takes last place in our priority list. In general, it doesn't matter to a depositor whether it takes ten minutes or ten hours for us to ingest materials. However, APTrust receives periodic ingest floods which our second-generation architecture could take weeks to clear and which required regular human intervention.

The horizontal scaling capabilities of the new architecture should allow us to process weeks-long backlogs in a matter of days, with little or no human intervention.

## Services Provided

APTrust provides the following preservation services:

* Ingest
* Ongoing Fixity Checks
* Deletion
* Restoration

Each service is composed of the following:

* __Temporary Storage__ includes S3 buckets in Amazon Web Services (AWS) to which depositors can upload items for ingest (receiving buckets), from which depositors can download restored objects and files (restoration buckets), and in which APTrust ingest processes store temporary data (staging buckets).
* __Preservation Storage__ consists of Glacier and S3 buckets provided by AWS and other commercial cloud providers. Depositors have no access to these buckets, and even APTrust staff have limited access.
* __Pharos__, a Rails application backed by a Postgres database, which tracks information about all objects and files preserved in (and deleted from) preservation storage. Pharos also allows authorized users to requestion deletion and restoration of objects and files.
* __Microservices__ running on Amazon EC2 instances perform the work required for ingest, fixity checking, deletion and restoration.
* __Supporting Services__ help microservices orchestrate multistep processes. These include NSQ, for managing work queues, and Redis for tracking internal state and metadata across processes in which multiple microservices must coordinate work.
