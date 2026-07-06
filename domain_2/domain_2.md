## RDS High Availability and Resilience

Multi-AZ is a synchronous replication mechanism. When you write to the primary instance, the write is not acknowledged to the application until it has been committed on both the primary and the standby. This means zero data loss on failover — RPO is effectively zero. The tradeoff is a small write latency increase because every write must round-trip to the standby before completing.

The standby instance is in a different Availability Zone within the same region. It runs on separate physical infrastructure, separate power, and separate network. The purpose is to survive an AZ-level failure. The standby is completely passive — it is not readable, it does not serve any traffic, and it does not appear as a separate endpoint. There is one endpoint for the DB instance, and it transparently points to whichever instance is currently primary.

Failover is automatic. RDS detects a failure (instance hardware failure, AZ failure, OS crash, or you trigger it manually) and updates the DNS record for the DB endpoint to point to the standby. This takes 60 to 120 seconds. During that window, connections are dropped and the application must reconnect. The application needs to handle reconnection — either built-in retry logic or a connection pool that handles dropped connections. Aurora Multi-AZ failover is faster, typically under 30 seconds, because Aurora uses a shared distributed storage layer and the new primary doesn't need to replay a transaction log.

The exam distinguishes Multi-AZ from Read Replicas on almost every RDS question. Multi-AZ is purely for high availability and durability. It does not help with read scaling at all. Read Replicas are for read scaling — they use asynchronous replication, they are readable endpoints, and they can be in the same region, a different AZ, or a different region entirely. Asynchronous means there is replication lag. Replication lag is usually a few seconds but can grow under heavy write load. This means a Read Replica is not suitable for reads that require seeing the most recent write — it's suitable for analytics, reporting, or read-heavy workloads that can tolerate slightly stale data.

You can have up to five Read Replicas per RDS instance (up to fifteen for Aurora). Read Replicas can be promoted to standalone instances — this is the DR pattern for cross-region failures. If the primary region fails, you promote the cross-region Read Replica and update your application's connection string. RTO is the time to promote plus the time to update DNS, which is minutes. RPO is whatever the replication lag was at the time of failure.

Aurora differs from standard RDS in its storage architecture. Aurora separates compute from storage. The storage layer is a distributed, self-healing cluster that spans three AZs with six copies of the data — two per AZ. The storage layer handles replication internally. Aurora replicas (up to 15) read from the same shared storage, so there is essentially no replication lag for reads. If the primary fails, Aurora promotes one of the existing replicas in under 30 seconds without needing to replay transactions.

Aurora Global Database extends this to multiple regions. One primary region handles writes; up to five secondary regions handle reads. The replication lag between regions is typically under one second (physical replication at the storage layer). If the primary region fails, you can promote a secondary region to primary in under one minute. This is effectively a Pilot Light DR pattern for Aurora — you always have a running, nearly up-to-date copy in the secondary region.

Point-in-Time Recovery allows you to restore a DB instance to any second within your backup retention window, which you set between 1 and 35 days. The way it works: RDS takes daily automated backups (snapshots) and continuously archives transaction logs to S3. To restore to a specific moment, RDS restores the nearest snapshot and then replays transaction logs up to the target time. Critically, PITR always creates a new DB instance with a new endpoint. It does not modify the existing instance. If your application needs to switch to the restored instance, it must update its connection string.

Manual snapshots are retained indefinitely — they persist even after you delete the DB instance, unlike automated backups which are deleted when the instance is deleted. This is a common exam trap: if you delete an RDS instance and want to preserve the data, either take a manual snapshot first or enable the "retain automated backups" option.

RDS Proxy is worth knowing for the exam. It sits between your application and the RDS instance, maintaining a pool of database connections. If the application has many short-lived connections (common with Lambda functions), RDS Proxy prevents connection storms from overwhelming the database. During a Multi-AZ failover, RDS Proxy holds application connections and transparently re-routes them to the new primary, reducing the time your application sees errors from the full 60–120 seconds to just a few seconds.

## Disaster Recovery Strategies

DR strategy selection is always a cost versus RTO/RPO tradeoff. The exam gives you a scenario with specific RTO and RPO requirements and asks which strategy meets them at the lowest cost, or gives you a budget and asks what RTO/RPO you can achieve.

Backup and Restore is the cheapest strategy. In the DR region, you have nothing running. Your data is backed up to S3 (database snapshots, AMIs, CloudFormation templates). When disaster strikes, you restore from backups — launch EC2 instances from AMIs, restore RDS from snapshots, apply CloudFormation stacks. RTO is measured in hours because you're starting from scratch. RPO is also measured in hours, bounded by how frequently you back up. The cost in the DR region is essentially just S3 storage for the backups.

Pilot Light keeps a minimal set of critical resources always running in the DR region. The "pilot light" metaphor: the gas is always flowing to the pilot light (your database), so ignition (full failover) is faster than starting from cold. Typically this means the database is always running and replicating from the primary region. The application tier is not running. On failover, you launch the application servers, point them at the DR database, and scale up. RTO is minutes to low tens of minutes — you're launching app servers, not restoring from snapshots. RPO is low because the database is continuously replicating. Cost is low but not zero — you're always paying for the DB instance in the DR region.

Warm Standby runs a fully functional but scaled-down version of your production environment in the DR region. Where production runs on r5.4xlarge instances with 10 nodes in an Auto Scaling group, your warm standby might run on r5.large instances with 2 nodes. On failover, you scale up to production capacity and shift traffic. RTO is minutes — you just need to scale and re-route DNS. RPO is very low because everything is running and synchronized. Cost is moderate — you're always paying for real infrastructure, just smaller than production.

Multi-Site Active/Active runs identical production environments in two or more regions simultaneously, serving real user traffic from both. Route 53 or Global Accelerator distributes traffic between them. RTO is near zero — if one region fails, the other is already serving traffic. RPO is near zero — both regions are processing writes. This requires your application to handle multi-region writes, which means your data layer must support multi-region active writes (DynamoDB Global Tables, Aurora Global Database write forwarding, or custom conflict resolution). This is the most expensive strategy and the most architecturally complex.

The exam often presents a scenario with a stated RTO and RPO and asks you to choose the most cost-effective strategy that meets those requirements. The decision logic:

- If RTO is hours and RPO is hours and cost is the primary constraint → Backup and Restore.

- If RTO is tens of minutes and RPO is minutes and the data tier must survive → Pilot Light.

- If RTO is minutes and RPO is seconds and the app must be ready to take full traffic immediately → Warm Standby.

- If RTO and RPO are both near zero and budget is not a constraint → Multi-Site Active/Active.

Aurora Global Database sits between Pilot Light and Warm Standby. You get sub-second RPO and sub-minute RTO for the data layer. The application layer in the secondary region is the variable — if you run a scaled-down application tier alongside Aurora in the secondary region, that's effectively Warm Standby. If you have no application tier running, it's closer to Pilot Light.

## Auto Scaling

An Auto Scaling Group manages a fleet of EC2 instances. It has a minimum capacity, a desired capacity, and a maximum capacity. The minimum is a floor — ASG will never scale below it. The maximum is a ceiling. The desired is the target the ASG tries to maintain at any given time. When a scaling policy fires, it adjusts the desired capacity, and the ASG launches or terminates instances to match.

Target Tracking scaling is the simplest and the exam's default answer for "least operational overhead." You specify a metric and a target value. The ASG uses a PID-like algorithm to continuously adjust capacity to keep the metric at the target. Common metrics: ASGAverageCPUUtilization, ALBRequestCountPerTarget, ASGAverageNetworkIn/Out. You can also use a custom metric published to CloudWatch. The ASG scales out aggressively (undershoots are expensive — users are affected) and scales in conservatively (waits until the metric has been below target for a sustained period).

Step Scaling lets you define multiple scale adjustments based on how far the metric is from the threshold. For example: if CPU is between 70% and 85%, add 2 instances. If CPU is above 85%, add 5 instances. This gives you proportional response — a small breach gets a small response; a large breach gets an aggressive response. You define the steps manually, which adds operational overhead but gives more control than Target Tracking.

Scheduled Scaling fires at a specific time regardless of metrics. You define a start time and an end time (or just a start time for a one-time action), and the desired capacity during that window. Used for predictable traffic patterns — scale up at 8 AM on weekdays, scale down at 8 PM.

Predictive Scaling uses ML to analyze historical metric patterns and forecast future load. It scales out proactively before the load arrives, rather than reactively after the metric breaches a threshold. This eliminates the lag inherent in reactive scaling — if your traffic spikes every Monday morning, reactive scaling fires after the spike has already started; predictive scaling adds capacity before Monday morning arrives. It can be used in forecast-only mode (observes and predicts, you act manually) or forecast-and-scale mode (acts automatically).

Cooldown periods prevent thrashing. After a scale-out event, the default cooldown (300 seconds) prevents another scale-out during that window, giving newly launched instances time to start handling traffic and allowing metrics to stabilize. You can set per-policy cooldowns that override the default. Target Tracking manages cooldown automatically — you don't need to tune it manually.

Lifecycle hooks are the most operationally important ASG feature for complex deployments. When the ASG decides to launch an instance, the instance enters Pending:Wait state. It pauses there — it's not added to the load balancer, not serving traffic — until you call CompleteLifecycleAction with CONTINUE (proceed) or ABANDON (terminate the instance). If you don't call it within the heartbeat timeout (default 1 hour), the ASG uses the DefaultResult (CONTINUE or ABANDON, configurable).

The pattern for launch hooks: instance comes up → lifecycle hook fires → EventBridge rule catches the event → Lambda runs your bootstrap logic (register with config management, run tests, notify a deployment system) → Lambda calls CompleteLifecycleAction. Only then does the instance join the load balancer.

Terminate hooks work similarly. When the ASG selects an instance for termination, the instance enters Terminating:Wait. You have time to drain connections, publish final logs, deregister from service registries, or capture a memory dump before the instance is terminated.

Warm pools are a feature for reducing scale-out latency. Instances in the warm pool are already launched and initialized — they've run userdata and are in a stopped or running state, waiting. When a scale-out event occurs, the ASG pulls instances from the warm pool instead of launching new ones from scratch. This reduces scale-out latency from minutes (full launch + userdata) to seconds (start a stopped instance). You pay for the stopped instances (storage only) or the running instances (full compute cost) depending on configuration.

Instance refresh is how you update instances in an ASG when you've published a new launch template or AMI. Instead of terminating all instances and relaunching (which causes downtime), instance refresh performs a rolling replacement. You specify a minimum healthy percentage (e.g., 90%) and a warm-up time. The ASG terminates a batch of instances, replaces them with new ones running the updated configuration, waits for them to pass health checks and warm up, then moves to the next batch.

Health checks in an ASG can come from two sources: EC2 status checks (default) or ELB health checks. EC2 status checks only detect whether the instance itself is reachable and the OS is running. ELB health checks detect whether the application on the instance is responding correctly. If you have an ASG behind a load balancer, you should enable ELB health checks — otherwise, an instance with a crashed application but a running OS will appear healthy to the ASG and won't be replaced.

## Route 53 for Resilience

Route 53 health checks are the mechanism that transforms routing policies from static configurations into dynamic failover systems. Without health checks, a routing policy just distributes or directs traffic — it doesn't respond to failures. With health checks, Route 53 continuously monitors your endpoints and stops routing traffic to unhealthy ones.

Health check types: endpoint checks (HTTP, HTTPS, TCP — Route 53 sends requests from 15 global health checker locations every 30 seconds by default, or every 10 seconds for fast health checks), calculated health checks (combine multiple health checks with AND/OR/NOT logic — similar to composite alarms), and CloudWatch alarm health checks (monitors a CloudWatch alarm state — this is the mechanism for checking private resources).

A health check marks an endpoint as unhealthy when it fails a threshold number of health checker locations (default: 18% of checkers must agree the endpoint is down, which prevents false positives from individual checker network issues). It marks it healthy again when it passes the threshold. Route 53 propagates this state change to DNS within 60 seconds.

Failover routing: you have a primary record and a secondary record. The primary record has a health check associated. When Route 53 evaluates a DNS query, if the primary is healthy it returns the primary record. If the primary is unhealthy, it returns the secondary. The secondary can be any endpoint — another region's load balancer, a static S3 website, a maintenance page. It can also have its own health check, so if both primary and secondary are unhealthy, Route 53 returns the secondary anyway (it always returns something rather than returning nothing).

Latency routing: Route 53 measures latency from each AWS region to various geographic locations and maintains a latency table. When a query comes in, Route 53 looks up which region has the lowest latency to the user's resolver location and returns that region's record. Paired with health checks, if the lowest-latency region is unhealthy, Route 53 automatically returns the next-best region. This gives you both performance optimization and automatic failover in a single routing policy.

Weighted routing: you associate a weight with each record. Traffic is distributed proportionally. Weight 0 stops traffic to that record without removing it. This is used for canary deployments (send 5% of traffic to the new version), gradual migration (increase weight of new infrastructure over time), and blue/green DNS switching (shift weights from 100/0 to 0/100 over time). Weighted routing with health checks: if a weighted record's health check fails, Route 53 removes it from the calculation and redistributes its traffic proportion to the remaining healthy records.

Geolocation routing: maps user location (determined by the IP address of the user's DNS resolver) to records. You define records for specific countries, continents, or a default. This is used for data sovereignty (users in the EU must be served by EU infrastructure), localization (serve French content to French users), or restricting access by geography. If no matching geolocation record exists and there's no default record, Route 53 returns NODATA — the query returns no result. Always create a default record as a fallback.

Geoproximity routing: routes based on the geographic distance between the user and the resource. Unlike geolocation, which is country/continent based, geoproximity measures actual distance. You can apply a bias — a positive bias expands the region that a resource covers (attracting more traffic), a negative bias shrinks it. Requires Route 53 Traffic Flow and is configured in the Traffic Flow visual editor.

Multivalue Answer routing: returns up to eight healthy records in response to DNS queries, shuffled randomly. Clients choose one. Route 53 performs health checks and only includes healthy records in the response. This is a simple load distribution mechanism at the DNS layer. It is not a substitute for a load balancer — DNS responses are cached by resolvers and clients, so it doesn't handle instance-level failures as quickly as an ELB health check would.

Private hosted zones: associated with VPCs. DNS resolution for those domains only works from within the associated VPCs. Health checks on private resources require the CloudWatch alarm mechanism because Route 53 health checkers are outside the VPC and cannot reach private endpoints. You create a CloudWatch alarm on a metric from the private resource (e.g., an RDS metric, an ALB health check metric), then create a Route 53 health check that monitors the state of that CloudWatch alarm.

## S3 Durability and Data Protection

S3 durability is 99.999999999% — eleven nines. This means if you store 10 million objects, you can expect to lose one object every 10,000 years on average. Durability is achieved by storing multiple redundant copies across at least three AZs (for Standard, Standard-IA, and Intelligent-Tiering storage classes). One Zone-IA stores data in a single AZ — durability is still 99.999999999% within that AZ, but if the AZ is destroyed, the data is gone.

Availability is different from durability. Standard class is 99.99% available, meaning it's designed to be accessible 99.99% of the time. Standard-IA is 99.9% available. One Zone-IA is 99.5%. Durability is about data persistence; availability is about whether you can read it right now.

Versioning keeps all versions of every object. When you delete an object in a versioned bucket, S3 places a delete marker — the object isn't actually gone, it's just hidden. You can restore it by deleting the delete marker. To permanently delete a versioned object, you must delete the specific version ID. Versioning cannot be fully disabled once enabled — you can only suspend it, which stops creating new versions but preserves existing ones.

MFA Delete requires multi-factor authentication to permanently delete object versions or to change the versioning state of the bucket (suspend or re-enable). It can only be enabled by the root account using the CLI — you cannot enable it in the console, and you cannot enable it as an IAM user. This protects against both accidental deletion and compromised IAM credentials.

Object Lock implements WORM — Write Once Read Many. Two modes: Governance mode allows users with the s3:BypassGovernanceRetention permission to delete objects. Compliance mode allows no one — not even the root account — to delete an object before the retention period expires. Compliance mode is used for regulatory requirements where data must be preserved immutably. Legal Hold is a separate flag that prevents deletion regardless of retention period — it must be explicitly removed before an object can be deleted.

Replication: Cross-Region Replication copies objects to a bucket in a different region. Same-Region Replication copies to a bucket in the same region. Both require versioning on source and destination buckets. Replication is asynchronous and best-effort — it is not instantaneous. Only new objects uploaded after the replication rule is created are replicated by default. To replicate existing objects, use S3 Batch Replication. Replication preserves storage class, object tags, and ACLs by default, but you can configure it to change the storage class in the destination.

Replication Time Control is an optional add-on to CRR/SRR that provides a 15-minute SLA — 99.99% of objects will be replicated within 15 minutes. It also enables replication metrics so you can monitor replication lag in CloudWatch.

Lifecycle policies transition objects between storage classes based on age or other conditions. Common pattern: objects start in Standard, transition to Standard-IA after 30 days, transition to Glacier Flexible Retrieval after 90 days, expire (delete) after 365 days. Lifecycle policies also manage incomplete multipart uploads — a common cost leak where uploads are abandoned mid-way and the partial data accumulates.

S3 Event Notifications fire when objects are created, deleted, or restored. Destinations: SNS, SQS, Lambda, and EventBridge. EventBridge is the most flexible destination because it supports routing to many more targets and content-based filtering. For near-real-time processing of S3 events, S3 → EventBridge → Lambda is the current recommended pattern.

## AWS Backup

AWS Backup is a centralized backup management service that covers RDS, Aurora, DynamoDB, EBS, EFS, FSx, S3, and Storage Gateway. The exam pattern is: any question asking for centralized backup management with the least operational overhead across multiple services or accounts points to AWS Backup, not service-specific snapshot features.

A backup plan defines the schedule (daily, weekly, etc.), the retention period, and the lifecycle (when to transition to cold storage). You assign resources to a backup plan by tag or by specific ARN. Tag-based assignment is the operational pattern — tag all production RDS instances with Environment=Production and create a backup plan that targets that tag. New instances automatically get backed up when they receive the tag.

Cross-account backup copies recovery points to a vault in another account. This protects against account-level compromise — even if your production account is ransomwared or credentials are stolen, the backups in the separate backup account are safe. The backup account's vault should have strict IAM policies limiting who can delete recovery points.

Vault Lock implements WORM for backup vaults. Once set, no one — including the root account and AWS — can delete recovery points before the retention period expires. This satisfies regulatory requirements for immutable backups. Vault Lock must be tested in a 72-hour grace period after configuration before it becomes permanent (cannot be undone after grace period).

AWS Backup integrates with AWS Organizations. You can create backup policies in the management account that apply to all member accounts. Member accounts can be prevented from modifying or deleting backup plans. This is the centralized compliance pattern.