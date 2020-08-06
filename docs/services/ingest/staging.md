# Copy to Staging

After the reingest check, the `ingest_staging_uploader` does the following:

* Streams the tarred bag from the depositor's receiving bucket through a tar reader.
* Copies files from the tar reader, one by one, into the staging bucket.

Because tar is a sequential format, the tar reader must read and upload files one at a time. No data ever touches the local disk in this operation. The staging uploader streams tarred data down from one bucket and sends untarred data back to another.

The efficiency and cost effectiveness of this process depends on the receiving buckets, the staging bucket, and the staging uploader all running within the same AWS region.

We copy files to a staging bucket instead of to a local disk for two reasons:

1. We never know how much disk space we will need at any given time. In practice, we know it varies from less than 1 GB to over 20 TB. In practice, scaling EBS volumes was problematic and expensive and EFS just didn't work. You'll find more on those issues in our [white paper on why we chose S3 for staging](https://docs.google.com/document/d/1DXgOEv_OHKBR9lc3aOk12R7z16CoPN94sh_g-VNeLt4/edit?usp=sharing)
2. We want the unpacked files to be available to all workers on any EC2 instance for further processing. Storing them on a local EBS volume makes them accessible only to the server to which the volume is attached.

## Resource Usage

The staging uploader uses large amounts of bandwidth for simultaneous sending and receiving. It also uses a noticeable amount of CPU, because S3 copies calculate checksums/etags as they go, and memory, because both the uploader and downloader buffer contents as they read/write.
