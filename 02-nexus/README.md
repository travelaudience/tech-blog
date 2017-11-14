# Artifact repository management at TA

This article describes how TA does artifact repository management on top of Google Cloud Platform technologies.

## The Problem

At [travel audience](https://travelaudience.com) (TA), we design, implement, run and maintain a bunch of software pieces. These software pieces appear in different forms, and adopt different technologies and deployment strategies, depending on their target needs. For instance, we have apps that consume, process and generate millions of messages per second, where minimalism is key to achieve low-latency, high-throughput and high-availability. These apps are written in compiled languages such as Go and are usually packed as containers so to make it easier to scale (horizontally) the number of instances needed at a certain time, e.g. based on user demand. On the other end of the spectrum, we also have apps that process terabytes of data per day, sometimes in near-realtime, sometimes within a few hours or days span. These apps are usually packed as Python, Java or Scala archives that are to be scheduled as jobs into a batch and/or stream processing engine.

And - shedding a little more light about our platform - most of our workloads run atop Google Cloud Platform (GCP):
- Containers are orchestrated with Kubernetes Engine (GKE), the fully-managed Kubernetes service.
- Batch and streaming pipelines are provided by Dataproc and Dataflow.
- Data is stored in Cloud Storage, Cloud SQL, BigTable and BigQuery.
- etc.

Given how heterogeneous our apps and deployment environments look like, we quickly realized we would need to be able to store, catalog and distribute our artifacts, that when stitched together form the fabric of TA.

## The Search and Decision

The hunt for a repository manager solution began and we started looking for solutions that would satisfy, at least, the following criteria:
1. Securely store and distribute private:
  - JAR (Java ARchive) packages, containing Java and Scala code;
  - Python packages (PyPI);
  - Docker containers;
2. Securely distribute public:
  - JAR (Java ARchive) packages, containing Java and Scala code;
  - Python packages (PyPI);
  - Docker containers;
3. Integrate with external authentication systems, and maybe authorization.
4. Easy to run and maintain in a highly-available fashion.
5. Able to recover in a disaster scenario (Disaster Recovery).

Time was of essence so we narrowed down our trials to a couple contenders, [Sonatype Nexus](https://www.sonatype.com/nexus-repository-sonatype) and [JFrog Artifactory](https://www.jfrog.com/artifactory/). And long story made short, we decided to go with the Nexus, mostly because the OSS version seemed to deliver most of what we were looking for.

After the decision was made, our Ops team devised a plan and set sail to adapt Nexus to our needs.

## The Execution

At this point, our Ops team had already identified certain limitations we would have to overcome:
- Caching Docker containers in Nexus is complex, therefore we decided to drop it for the time being.
- Integration with Google Cloud Identity Access Management (Cloud IAM) is nonexistent, so we would have to design and implement it.
- Integration with Google Cloud Storage for backup storage and retrieval is nonexistent, so we would have to design and implement it.
- There's no official support for Kubernetes, the container orchestration solution in place at TA, so we would have to work on it.

After a thoughtful design process, this is what we came up with:

![alt text](big-picture.png "TA Nexus big picture")

1. Developer tools, such as Maven and Docker and GKE reach Nexus through a Google Cloud Load-Balancer (GCLB) instance.
2. The GCLB instance directs incoming traffic to an instance of `nexus-proxy`.
3. `nexus-proxy` checks the request headers for user-identity and then for GCP Organization membership using the Cloud IAM APIs.
4. If the user is authorized, `nexus-proxy` redirects incoming traffic to Nexus.
5. `nexus-backup` periodically uploads backups to Cloud Storage.
6. Nexus caches packages from Maven Central and PyPI.

### Nexus authentication with Cloud IAM

We knew beforehand that we would need to authenticate Nexus against Cloud IAM but we wanted to make this authentication optional so that the tool we would design and implement could be used by others, in simpler scenarios, e.g. no Cloud IAM.

In order to deliver, we have [designed](https://github.com/travelaudience/nexus-proxy/blob/master/docs/design.md) and implemented [`nexus-proxy`](https://github.com/travelaudience/nexus-proxy).

Interestingly enough, later we would found out this option would make it much easier to fix other issues, unrelated to authentication, e.g. Nexus can't expose Docker private registry with the same set-up used to expose the other artifact repositories which would defeat the decision of using HTTPS for everything. It also replaced Nginx as the reverse-proxy behind GCLB.

### Nexus backup

Nexus can be configured to back up its internal database on a regular basis. However, this process does not take blob stores into account. Furthermore, the backups are persisted in the local disk alone, meaning if the disk is lost, the backups are lost too.

So, we came up with a tool, [`nexus-backup`](https://github.com/travelaudience/docker-nexus-backup) (a container made up of a bunch of scripts), to execute the backup procedure and then upload the result to Cloud Storage.

![alt text](nexus-backup.png "Nexus backup design")

1. Nexus' blobstores are stored in a Google Persistent Disk (PD) accessible by `nexus-backup`.
2. A file used to trigger backups is also stored in the same PD.
3. A task configured in Nexus periodically dumps a backup of the Nexus database to the same PD.
4. Another task, configured in Nexus, signals that a backup is occurring by touching the trigger file.
5. `nexus-backup` watches the trigger file and, whenever the trigger file is touched, starts the blobstore backup procedure.
6. `nexus-backup` fetches the Nexus database dump and blobstore backup files from the PD.
7. `nexus-backup` uploads the backup to a pre-configured Cloud Storage bucket.
8. A recovery procedure may be conducted by copying a backup to a PD attached to Nexus. Nexus will pick the backup and restore it automatically upon restart.

Worthy of note, whenever a lock file has been present for more than 12 hours (probably meaning a failed backup), the lock file is removed so that further backups can happen.

### Nexus lifecycle management & usage

We rely on Kubernetes (GKE) to deploy and manage the lifecycle of our Nexus set-up.
We have open-sourced [detailed instructions](https://github.com/travelaudience/kubernetes-nexus) on [how we do it](https://github.com/travelaudience/kubernetes-nexus#deployment), including [disaster-recovery](https://github.com/travelaudience/kubernetes-nexus#backup-and-restore), and [how our developers use it](https://github.com/travelaudience/kubernetes-nexus#usage).

Also, we are currently working on an Helm chart for it, which should be made available real soon.
