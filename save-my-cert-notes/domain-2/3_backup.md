# Backup, Restore, and Disaster Recovery Strategies on AWS


A backup and restore strategy defines how you capture recoverable copies of your data and how quickly you can bring a workload back after deletion, corruption, or a Regional outage. On the SOA-C03 exam, Task 2.3 tests whether you can automate backups with AWS Backup and native tooling, restore databases to a point in time, protect storage with versioning, and match one of four named disaster recovery patterns to a stated RTO, RPO, and cost requirement. The questions are rarely about whether to back up — they ask which mechanism meets a quoted recovery objective at the lowest cost, and what actually happens when you run the restore. Expect scenario stems that quote recovery targets in minutes or hours, and distractors that confuse high availability with backup. This lesson covers the services, the restore mechanics, and the trade-offs the exam expects you to weigh.

What you’ll learn
Define RTO and RPO and use them to drive every backup and disaster recovery decision
Automate backups centrally with AWS Backup plans, vaults, and tag-based resource assignment
Compare native automation: Data Lifecycle Manager, RDS automated backups, and DynamoDB PITR
Predict what a restore produces — new instances, new tables, new volumes — and plan for endpoint changes
Protect S3 and FSx data with versioning, replication, and automatic backups
Rank the four DR patterns by RTO, RPO, and cost, and pick the right one from a scenario
RTO and RPO: the numbers behind every backup decision
RTO and RPO are the two numbers that decide every question in this task. Recovery time objective (RTO) is the maximum acceptable time between failure and full restoration of service — how long you can afford to be down. Recovery point objective (RPO) is the maximum acceptable data loss, measured as time — how far back your most recent recoverable copy is allowed to be. A nightly backup gives you an RPO of up to 24 hours: fail just before tonight's backup and you lose almost a full day of writes. Continuous mechanisms such as point-in-time recovery shrink RPO to minutes or seconds.

Every strategy in this lesson is a three-way trade between RTO, RPO, and cost. Tighter objectives always cost more, because meeting them means copying data more often, keeping more infrastructure warm, or both. When an exam stem quotes "no more than 15 minutes of data loss" it is handing you an RPO; "back online within an hour" is an RTO. Translate the sentence into those two numbers first, then eliminate any option that cannot meet them, and pick the cheapest survivor.

One boundary to fix in your head now: Multi-AZ is availability, not backup. A Multi-AZ RDS deployment synchronously replicates to a standby, so an accidental DELETE or a corrupted table replicates to the standby instantly and faithfully. Multi-AZ protects you from infrastructure failure (Task 2.2 territory); only backups, snapshots, and versioning protect you from bad data.

AWS Backup: centralize and automate backups at scale
AWS Backup is the exam's default answer whenever a scenario asks you to automate and centrally manage backups across multiple services or accounts. Instead of configuring each service's native mechanism separately, you define a backup plan: one or more rules, each with a schedule (for example, daily at 03:00 UTC), a retention period, and an optional lifecycle that transitions recovery points to lower-cost cold storage after a number of days before eventual expiry.

Backups land in a backup vault, an encrypted container for recovery points with its own access policy. Vault Lock — know it at concept level — makes a vault write-once-read-many, so recovery points cannot be deleted before their retention ends, even by administrators. That is the answer to "protect backups from accidental or malicious deletion" and to ransomware-flavored stems.

Resources join a plan through resource assignments, and the operationally scalable method is assignment by tag: assign every resource tagged backup=daily and any new instance, volume, or table created with that tag is backed up automatically, with no per-resource configuration. When a question mentions hundreds of resources or new resources appearing constantly, tag-based assignment is the intended answer.

Backup plans can also copy recovery points to another Region or another account. Cross-Region copies are the foundation of Regional disaster recovery; cross-account copies isolate backups from a compromised source account. Supported resources you should recognize include EC2 instances, EBS volumes, RDS and Aurora databases, DynamoDB tables, EFS file systems, FSx file systems, and S3 buckets — one service, one policy, many resource types.

Native backup automation: DLM, RDS, and DynamoDB
Native, per-service mechanisms still appear on the exam, usually contrasted with AWS Backup or with each other. Amazon Data Lifecycle Manager (DLM) automates EBS snapshots specifically: you define schedules that target volumes or instances by tag, and DLM creates, retains, and deletes snapshots on that cadence. If the scenario is only about automating EBS snapshots, DLM is a valid answer; if it spans multiple services, AWS Backup wins.

RDS automated backups run a daily snapshot during the backup window and continuously capture transaction logs, which together enable point-in-time restore (PITR) to any moment inside the retention window. Retention is configurable up to 35 days; setting it to 0 disables automated backups entirely. Automated backups are deleted when their retention expires or (by default) when the instance is deleted. Manual snapshots are the contrast case: taken on demand, they are kept until you delete them, survive instance deletion, and are the right answer for "retain a backup for years" or pre-change safety copies.

DynamoDB offers two mechanisms with the same split. Point-in-time recovery (PITR), once enabled, continuously backs up the table and lets you restore to any second in the last 35 days — the answer to "restore the table to just before the bad write." On-demand backups are full snapshots kept until deleted, suited to long-term retention and compliance. Neither impacts table performance. Map the pattern: continuous mechanism → small RPO within a rolling window; manual/on-demand snapshot → indefinite retention, RPO equal to snapshot age.

Restore mechanics: every restore creates a new resource
The single most-tested restore fact is this: restores create new resources — they do not repair the original in place. An RDS point-in-time restore creates a new DB instance with a new endpoint; restoring from an RDS snapshot likewise creates a new instance. DynamoDB PITR and on-demand backups both restore to a new table with a new name — you cannot restore over an existing table. An EBS snapshot restore creates a new volume that you then attach to an instance (detaching or replacing the old one yourself).

The classic ops gotcha follows directly: after an RDS restore, the application is still pointing at the old endpoint. Your runbook must include repointing the application — update the connection string, or better, front the database with a Route 53 CNAME or an application-level configuration so a restore only changes one record. If an exam stem says "the database was restored but the application still shows old data" or "cannot connect after restore," the answer is almost always the unchanged endpoint. Also remember that restored resources come up with settings you may need to re-apply: for DynamoDB restores, things like auto scaling settings, tags, and alarms are not carried to the new table.

Finally, an untested restore is an assumption, not a strategy. Restore testing is the only proof that your RTO is achievable (how long does the restore actually take on a production-sized dataset?) and that your RPO holds. Schedule periodic restore tests and game days; the exam rewards answers that validate recovery, not just configure backups.

S3 versioning: undelete and overwrite protection
S3 versioning protects objects from overwrite and deletion by keeping every version instead of replacing it. With versioning enabled on a bucket, an overwrite writes a new current version and the previous one becomes a retrievable noncurrent version. A simple delete does not remove data at all — it places a delete marker as the current version, which makes the object appear gone while every prior version remains underneath.

Walk the recovery: a user reports that reports/q3.pdf was accidentally deleted this morning. You list the object's versions, see a delete marker on top of the intact versions, and delete the delete marker. The previous version immediately becomes current again and the object "reappears" — no restore job, no data copy. To roll back an unwanted overwrite instead, copy the desired noncurrent version back over the current one (or delete the newer versions). Any stem containing "accidentally deleted object" plus S3 is pointing at versioning and delete markers.

Two companions to know. MFA Delete, at concept level, requires multi-factor authentication to permanently delete a version or change the bucket's versioning state — an extra guard on destructive actions. And because versioning keeps everything, pair it with a lifecycle rule for noncurrent versions: for example, expire noncurrent versions after 90 days, or transition them to a cheaper storage class first. That is the standard answer to "versioned bucket costs keep growing." Note that once enabled, versioning cannot be disabled — only suspended, which stops creating new versions but keeps existing ones.

S3 replication and FSx backups
S3 replication continuously copies objects from a source bucket to a destination bucket: Cross-Region Replication (CRR) for geographic separation and DR, Same-Region Replication (SRR) for log aggregation or keeping a copy in another account. Two rules the exam leans on: replication requires versioning enabled on both buckets, and a new replication rule copies only new objects by default — objects that existed before the rule was created are not replicated. The concept-level fix for pre-existing objects is S3 Batch Replication, which replicates existing objects retroactively.

Treat replication as an availability and locality tool, not a backup by itself: it faithfully propagates changes, so an overwritten object is overwritten in the destination too. It is versioning (on both sides) and backups that give you history; replication gives you a second location. Combining CRR with versioning and lifecycle rules is a common correct answer for "durable, geographically separated copies of critical data."

Amazon FSx file systems carry their own protection. FSx takes automatic daily backups during a backup window with a configurable retention period, and supports user-initiated backups that are kept until you delete them — the same automatic-versus-manual retention split you saw with RDS. FSx backups are incremental and can also be managed centrally through AWS Backup. For versioning-like protection of individual files, FSx for Windows File Server supports shadow copies, letting users restore previous versions of files themselves — know that at concept level only.

The four DR patterns compared
Exam guide v1.1 names four disaster recovery patterns, and the exam expects you to rank them by RTO, RPO, and cost and to recognize each from a one-line description. They form a spectrum: each step up keeps more infrastructure running in the recovery Region, which costs more and recovers faster.

Pattern	What runs before disaster	RTO	RPO	Cost	Trigger phrase
Backup and restore	Nothing — only backups stored (ideally cross-Region)	Hours	Hours (backup age)	Lowest	"Lowest cost", "can tolerate hours of downtime"
Pilot light	Data replicated live; core services (database) on, everything else off or minimal	Tens of minutes	Seconds–minutes	Low	"Data must be current, servers can be started on failover"
Warm standby	A scaled-down but fully functional copy of the whole stack	Minutes	Seconds–minutes	High	"Minutes of downtime, scaled-down copy already running"
Multi-site active/active	Full-capacity stack serving traffic in multiple Regions	Near zero	Near zero	Highest	"No downtime acceptable", "serve from both Regions"
The distinction that trips candidates: pilot light versus warm standby. In pilot light the data layer is alive and replicating but the application tier is not serving — you must start and scale it at failover. In warm standby everything is already running end to end, just undersized; failover is scale-up plus traffic switch, hence the faster RTO. Given equal ability to meet the stated objectives, always choose the cheaper pattern — that is the compensatory logic the exam applies.

Failover operations and a worked scenario
Failover is an operational sequence, not a checkbox. The standard mechanics: Route 53 failover routing shifts DNS from the primary to the recovery endpoint (the health checks that automate this are Task 2.2 material — here you need the routing role). For databases, you either promote a cross-Region read replica to a standalone, writable instance, or restore from a cross-Region backup copy — promotion takes minutes with near-current data; restore takes longer and loses data back to the copy's age. Then repoint applications (that endpoint change again), scale the recovery stack to production size, and verify.

Work the numbers on a typical stem: an RDS-backed application must survive a Regional outage with RPO of 15 minutes and RTO of 1 hour, at minimum cost. Test each pattern. Backup and restore fails on both counts: cross-Region copies are hours old (blowing the 15-minute RPO) and restoring plus rebuilding the stack in a fresh Region will strain an hour. Warm standby and active/active both meet the objectives but keep expensive infrastructure running. Pilot light is the cheapest fit: a cross-Region read replica replicates continuously, so RPO is seconds to minutes — well inside 15 minutes; on failover you promote the replica, launch the pre-staged application tier from AMIs or templates, and switch Route 53 — achievable inside an hour if rehearsed. "If rehearsed" is the operational point: the promotion, the endpoint change, and the scale-up must live in a tested runbook, because the first time you run a failover should never be during the disaster.

💡
Tip. Expect scenario stems that quote recovery objectives and ask for the cheapest compliant design: "restore to any point in the last N days" points at PITR and automated backups, "cheapest DR" at backup and restore, "minutes of downtime with a scaled-down copy running" at warm standby, and "accidentally deleted object" at S3 versioning and delete markers. Restore-mechanics questions probe the endpoint gotcha — an application broken after an RDS restore means nobody repointed it at the new instance. Watch for Multi-AZ offered as a distractor on data-corruption questions, and for AWS Backup with tag-based assignment as the answer whenever backups must be automated across many resources or services.

Key takeaways
RPO = maximum acceptable data loss, RTO = maximum acceptable downtime — translate every scenario into these two numbers first.
AWS Backup centralizes schedules, retention, cold-storage lifecycle, and cross-Region/cross-account copies; tag-based resource assignment is the at-scale answer.
RDS automated backups (retention up to 35 days) enable point-in-time restore; manual snapshots are kept until you delete them.
DynamoDB PITR restores to any second in the last 35 days; on-demand backups persist until deleted.
Every restore creates a NEW resource with a new endpoint or name — repointing the application belongs in the runbook.
S3 versioning turns a delete into a delete marker — remove the marker to restore; replication needs versioning and copies only new objects by default.
DR patterns by rising cost and falling RTO/RPO: backup and restore → pilot light → warm standby → multi-site active/active; pick the cheapest that meets the objectives.
Multi-AZ is availability, not backup — corruption replicates to the standby; and an untested restore proves nothing about RTO or RPO.