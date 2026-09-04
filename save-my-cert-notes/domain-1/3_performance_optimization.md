# EC2, EBS, S3 and RDS Performance Optimization for SOA-C03


Performance optimization on the SOA-C03 exam means reading utilization metrics, recognizing which resource they say is the bottleneck, and choosing the AWS-native fix — right-sizing an instance from Compute Optimizer recommendations, migrating a burst-exhausted gp2 volume to gp3, or putting RDS Proxy in front of a connection-storming database. Task 1.3 absorbed what was an entire domain on the previous exam, so expect several scenario questions from it. They follow one pattern: you are given symptoms — a nightly batch job that slows after twenty minutes, an API whose database latency spikes at peak — and asked to name the metric that proves the diagnosis and the cheapest change that fixes it. This lesson works through compute right-sizing and burstable-instance credits, EBS volume types and their ceilings, S3 transfer strategies, EFS versus FSx selection, RDS Performance Insights and RDS Proxy, and EC2 placement groups — each from the operator's seat.

What you’ll learn
Right-size EC2 instances using CloudWatch utilization data and AWS Compute Optimizer recommendations
Diagnose T-family CPU credit exhaustion and decide between unlimited mode and a fixed-performance family
Troubleshoot EBS bottlenecks with BurstBalance and VolumeQueueLength, and migrate gp2 volumes to gp3 live
Select the right S3 transfer strategy: multipart upload, Transfer Acceleration, or AWS DataSync
Choose between EFS and the FSx family for shared storage by protocol and workload
Resolve RDS performance problems with Performance Insights, key CloudWatch metrics, RDS Proxy, and read replicas
Right-size compute from utilization evidence
Right-sizing starts with evidence, not instinct. CloudWatch already records CPUUtilization, network throughput, and per-volume disk I/O for every instance; memory and disk-space usage appear only after the CloudWatch agent publishes them (configuring that agent belongs to task 1.1 — here you consume its output). An instance averaging 4% CPU for two weeks is over-provisioned and burning money. One pinned above 80% with climbing queue depths is under-provisioned and burning users.

AWS Compute Optimizer automates that judgment. It applies machine learning to your CloudWatch utilization history and classifies each EC2 instance as over-provisioned, under-provisioned, or optimized, then recommends specific alternative instance types alongside the projected utilization on each. It produces the same style of recommendation for EBS volumes (better volume type or IOPS/throughput settings), Lambda functions (memory size), and Auto Scaling group configurations. When an exam scenario asks which tool recommends a different instance type based on historical utilization, the answer is Compute Optimizer — Trusted Advisor flags idle and low-utilization instances as a cost check, but Compute Optimizer is the service built to recommend replacements.

Resource tags make optimization operational at scale. Tag instances with team, application, and environment keys, activate them as cost allocation tags, and you can slice CloudWatch dashboards and Cost Explorer by workload, attribute an oversized fleet to its owner, and target exactly the right instances when you act on a recommendation. Untagged fleets turn every optimization exercise into archaeology, which is why the exam names tags as a performance-optimization tool, not just a billing nicety.

Instance families and the T-family credit trap
Instance families encode a resource ratio, and picking one is really deciding which resource your workload exhausts first. General purpose (M, and burstable T) balances CPU and memory and fits most app servers. Compute optimized (C) gives a higher CPU share for encoding, batch processing, and CPU-bound APIs. Memory optimized (R and X) suits caches, in-memory analytics, and databases that show memory pressure. Diagnose from metrics: an instance out of memory needs R, not a bigger C.

Burstable T-family instances are the exam's favorite trap. A T instance earns CPU credits at a fixed rate while running below its baseline CPU share and spends them to burst above it. Sustained load drains the balance, and in standard mode a T instance with zero credits is throttled to its baseline — the workload suddenly crawls even though nothing in the application changed. That signature — performance degrades after a period of sustained load, or a service slows at the same busy hour every day, on a t3 or t4g instance — should send you straight to CPUCreditBalance in CloudWatch. A balance sliding to zero right before the slowdown is the diagnosis.

Remediation is a cost decision. Unlimited mode lets the instance keep bursting past an empty balance, billing you for the surplus credits — the right call for occasional overruns. But if the instance is busy most of the day, surplus charges accumulate and a fixed-performance family (M or C) is cheaper and steadier. Burstable pricing assumes a mostly-idle, spiky workload; constant load on a T instance is a sizing mistake, not a tuning opportunity.

EBS volume types and their performance ceilings
Every EBS volume type has a performance ceiling, and troubleshooting starts with knowing which ceiling you bought. The exam expects you to choose among these:

Type	Performance model	Best for	Operational watch-out
gp3	3,000 IOPS and 125 MiB/s included at any size; provision up to 16,000 IOPS and 1,000 MiB/s independently of capacity	Default choice for boot and general workloads	Provisioned IOPS/throughput are settings you must raise deliberately
gp2	Baseline 3 IOPS per GiB (minimum 100); smaller volumes burst to 3,000 IOPS on credits	Legacy general purpose	BurstBalance exhaustion throttles you to baseline
io1 / io2	Provisioned IOPS SSD; consistent sub-millisecond-class latency, io2 adds higher durability	I/O-intensive databases needing sustained high IOPS	Most expensive; size the IOPS to measured need
st1	Throughput-optimized HDD with burst throughput model	Large sequential I/O: log processing, ETL, data warehouses	Poor for small random I/O; cannot be a boot volume
sc1	Cold HDD, lowest cost per GiB	Infrequently accessed sequential data	Lowest performance; cannot be a boot volume
The decisive difference: gp2 couples performance to size — the only way to get more baseline IOPS is a bigger volume — while gp3 decouples them, so you dial IOPS and throughput up without adding capacity, usually at lower cost. That is why the standing recommendation is to migrate gp2 volumes to gp3. Reach for io1/io2 only when a database needs sustained IOPS beyond gp3's ceiling, and for st1/sc1 when the access pattern is large and sequential, where throughput matters and IOPS do not.

Walkthrough: diagnosing gp2 burst exhaustion and fixing it live
Here is the incident the exam loves, end to end. A nightly reporting job reads heavily from a 500 GiB gp2 data volume. The job starts fast, then slows to a crawl about twenty-five minutes in, every night. The application team swears nothing changed.

Diagnose from the volume's CloudWatch metrics. BurstBalance starts the night at 100% and slides to 0% right when the slowdown begins — the volume was bursting to 3,000 IOPS on credits, ran dry, and got throttled to its baseline of 1,500 IOPS (500 GiB × 3 IOPS/GiB). VolumeQueueLength climbs at the same moment: I/O requests are arriving faster than the throttled volume can service them, so they queue. That pair — BurstBalance at zero, queue length rising — is the textbook burst-exhaustion signature.

Fix it without downtime. Elastic Volumes lets you modify a volume's type, size, and provisioned performance while it stays attached and in use. Change the volume from gp2 to gp3 — it now has 3,000 IOPS with no credit mechanism at all — and if the job needs more, provision extra IOPS or throughput directly. No detach, no restart, no maintenance window.

Then check the other ceiling. Volumes are not the only limit: every EBS-optimized instance type has its own maximum aggregate EBS throughput and IOPS. A small instance can saturate its instance-level EBS limit while every attached volume still looks healthy. If volume metrics are clean but storage is still slow, compare the instance's aggregate EBS traffic against its documented instance-type maximum — the fix there is a larger instance (or spreading volumes across instances), not a faster volume.

S3 performance: moving data in faster and storing it smarter
S3 optimization questions are about matching a transfer tool to a transfer problem.

Multipart upload splits a large object into parts uploaded in parallel and reassembled by S3. Parallelism raises throughput, a failed part is retried alone instead of restarting the whole object, and an interrupted upload can resume. AWS recommends it for objects over 100 MB, and since a single PUT is capped at 5 GB, very large objects require it. Pair it with a lifecycle rule that aborts incomplete multipart uploads, or abandoned parts quietly accrue storage charges.

S3 Transfer Acceleration targets distance: clients upload to the nearest CloudFront edge location and the data rides the AWS backbone to your bucket. It helps when users are far from the bucket's Region — field teams on other continents uploading to a central bucket — and does nothing useful when uploads originate near the Region.

AWS DataSync is the managed bulk-transfer service: an agent reads from on-premises NFS or SMB storage (or another AWS storage service) and runs scheduled, incremental, verified transfers into S3, EFS, or FSx with built-in bandwidth throttling. Choose it for migrations and recurring data-center-to-AWS syncs; choose Transfer Acceleration for ad-hoc uploads from distant clients; choose multipart for any single large object.

Two storage-side levers finish the picture. S3 scales request throughput per prefix — a documented 3,500 writes and 5,500 reads per second per prefix — so spreading hot objects across multiple prefixes multiplies aggregate throughput for parallel workloads. And S3 Lifecycle policies keep storage efficient by transitioning aging objects to infrequent-access and archive tiers and expiring what nobody reads.

Shared storage selection: EFS versus the FSx family
Shared-storage questions are selection questions: match the protocol and workload to the service. The four options you must tell apart:

Service	Protocol / clients	Sweet spot	Key traits
Amazon EFS	NFS — Linux instances and containers	General shared file storage across many Linux clients	Fully elastic capacity, Regional storage spans multiple AZs; lifecycle policies move cold files to Infrequent Access automatically
FSx for Windows File Server	SMB — Windows clients	Windows file shares	Integrates with Active Directory, supports Windows ACLs and DFS namespaces
FSx for Lustre	Lustre — Linux HPC clients	High-performance computing and ML training scratch space	Very high throughput parallel file system; can link to an S3 bucket, presenting objects as files
FSx for NetApp ONTAP	NFS, SMB, and iSCSI together	Mixed Linux and Windows environments; NetApp shops migrating	Multiprotocol access to the same data, ONTAP features like snapshots, cloning, and tiering
The trigger words do most of the work. Linux servers need a shared file system is EFS. Windows file share with Active Directory permissions is FSx for Windows File Server. Shortest training time for an ML job or HPC cluster scratch storage is FSx for Lustre. Both Linux and Windows clients must access the same data is FSx for NetApp ONTAP.

Optimization within a choice matters too: the skill statement calls out EFS lifecycle policies, which transition files unread for a configured period to the Infrequent Access class and can move them back on access — the standard answer for an EFS file system that has grown expensive with cold data.

RDS performance: Performance Insights, RDS Proxy, and read offload
RDS troubleshooting starts with Performance Insights, which turns raw database load into a picture: average active sessions over time against the instance's vCPU line, sliced by wait event, top SQL statement, host, and user. When the load line crosses the vCPU line, sessions are waiting; the top-SQL breakdown names the query, and the wait events say whether it is CPU, I/O, or lock contention. Performance Insights also surfaces proactive recommendations for detected problems, so the exam treats it as the first stop for why is the database slow — before anyone reaches for a bigger instance.

The CloudWatch side gives four metrics to read fluently: CPUUtilization (compute-bound queries or an undersized class), DatabaseConnections (an unusual spike often precedes exhaustion), FreeableMemory (a steady decline signals memory pressure and cache eviction, pushing reads to disk), and ReadLatency/WriteLatency (storage-level slowness pointing at IOPS limits or volume type).

RDS Proxy answers the connection-storm scenario. It sits between application and database, pooling and sharing connections, so thousands of short-lived clients — classically Lambda functions scaling out — reuse a small warm pool instead of overwhelming the database with connection churn. It also shortens failover impact by holding client connections while the database recovers.

Read replicas offload read-heavy traffic: point reporting and read-only API paths at replicas and reserve the primary for writes, accepting replication lag. And when the resource itself is undersized, scale the instance class or the storage — moving RDS storage to gp3 or provisioned IOPS raises the I/O ceiling. Automated scaling policies belong to Domain 2; here you make the sizing call.

EC2 placement groups and enhanced networking
Placement groups control where EC2 puts instances relative to each other, and each of the three strategies answers a different exam prompt.

A cluster placement group packs instances close together inside a single Availability Zone for the lowest network latency and highest per-flow throughput between them. It is the answer to lowest latency between nodes — tightly coupled HPC and any chatty node-to-node workload. The trade-off is blast radius: everything shares one AZ, so a zone event takes the whole group.

A spread placement group does the opposite: each instance lands on distinct underlying hardware, with a limit of seven running instances per AZ per group. It exists to keep a small set of critical instances from failing together — think a handful of servers where simultaneous hardware failure would be unacceptable.

A partition placement group divides instances into partitions (up to seven per AZ) where each partition occupies racks that share no hardware with other partitions. Large distributed systems that manage their own replication — Hadoop, Kafka, Cassandra — map replicas to partitions so one rack failure cannot touch multiple replicas, and the group scales far beyond spread's seven-instance cap.

Enhanced networking earns a mention-level answer: the Elastic Network Adapter (ENA) gives instances higher packet-per-second performance, lower latency, and lower jitter, and it is enabled by default on current-generation instance types with current AMIs. If a scenario pairs a placement group with a network-performance requirement, enhanced networking is the companion feature, not a competing choice.

💡
Tip. This task is tested almost entirely as symptom-to-fix scenarios: you map a described behavior to the metric that proves it, then pick the cheapest effective remediation. Watch the trigger phrases — a workload that slows after sustained load or degrades at the same time daily points at credit exhaustion (CPUCreditBalance on T instances, BurstBalance on gp2 volumes); thousands of short-lived connections from Lambda means RDS Proxy; lowest latency between nodes means a cluster placement group; a Windows file share with Active Directory means FSx for Windows File Server; Linux servers sharing a file system means EFS. When Compute Optimizer and Trusted Advisor both appear as options for recommending instance types from utilization history, choose Compute Optimizer.

Key takeaways
Compute Optimizer turns CloudWatch history into over/under-provisioned classifications with specific EC2, EBS, and Lambda recommendations — Trusted Advisor only flags, it doesn't size.
A T-family instance that slows under sustained load has drained its CPU credits — check CPUCreditBalance; fix with unlimited mode or a fixed-performance family.
gp3 includes 3,000 IOPS and 125 MiB/s at any size and decouples performance from capacity; gp2 ties IOPS to size and throttles when BurstBalance hits zero.
Elastic Volumes modifies volume type, size, and IOPS on a live attached volume — gp2 to gp3 migration needs no downtime.
The instance's own EBS-optimized throughput cap can bottleneck before any volume limit — if volumes look healthy but storage is slow, check the instance ceiling.
Multipart upload for large objects, Transfer Acceleration for distant clients, DataSync for bulk or recurring transfers, prefixes for request-rate parallelism.
EFS = Linux NFS, FSx for Windows = SMB with Active Directory, FSx for Lustre = HPC/ML scratch, FSx for NetApp ONTAP = multiprotocol.
RDS Proxy pools connections to absorb connection storms; read replicas offload reads; Performance Insights names the top SQL and wait events behind DB load.