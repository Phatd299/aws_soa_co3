I'll pull together a comprehensive Domain 2 deep-dive. Let me verify a handful of exam-critical numbers first (auto scaling cooldowns, Multi-AZ failover times, health check intervals) before compiling the full reference.# Domain 2: Reliability & Business Continuity — Full Deep-Dive Reference

Covers Task 2.1 (Scalability & Elasticity), Task 2.2 (HA & Resilience), Task 2.3 (Backup & Restore) at exam depth.

---

## Task 2.1: Scalability and Elasticity

### Skill 2.1.1 — Scaling mechanisms in compute environments

**EC2 Auto Scaling policy types**

| Policy | Behavior | When exam favors it |
|---|---|---|
| Target tracking | Maintains a metric at a target value (e.g., CPU 50%); auto-creates CloudWatch alarms | Default recommendation — "least operational overhead" |
| Step scaling | Different step adjustments based on alarm breach magnitude | Need granular response to how far a threshold is breached |
| Simple scaling | One adjustment per alarm, then waits full cooldown before evaluating again | Legacy; almost never the "best" answer |
| Scheduled scaling | Set capacity at specific date/time | Predictable, known traffic patterns (e.g., batch job at 2 AM) |
| Predictive scaling | ML-based, analyzes daily/weekly patterns, forecasts and pre-scales | Recurring, cyclical traffic (e.g., business-hours spikes) — proactively launches ahead of demand instead of reacting |

**Key ASG mechanics**

| Setting | Default | Notes |
|---|---|---|
| Default cooldown | 300 seconds | Applies to simple scaling and ASG-level default; blocks further scaling activity until it expires |
| Instance warmup (target tracking/step) | Set via `--default-instance-warmup` | Excludes a newly launched instance's metrics from aggregate calcs until warmup completes — prevents premature scale-in from artificially low CPU on cold instances |
| Health check grace period | 0 via API/CLI (console UI defaults to 300s) | Must cover full app bootstrap time or ASG cycles instances forever, marking them unhealthy before they're ready |
| Termination policies | `Default` (balances AZs, then oldest launch config/template, then closest to next billing hour) | Can override with `OldestInstance`, `NewestInstance`, `OldestLaunchTemplate`, `AllocationStrategy` (Spot) |

**Lifecycle hooks vs Warm pools vs Instance refresh — a common distractor cluster**
- **Lifecycle hooks**: pause an instance in `Pending:Wait` or `Terminating:Wait` to run custom actions (e.g., install software, drain connections) before it enters service or is terminated. Default timeout 3600s (1 hr), extendable via heartbeat, max 48 hrs (172,800s) with heartbeats.
- **Warm pools**: pre-initialized instances kept in a stopped/running "pool" outside the ASG's active fleet, so scale-out is near-instant without paying full compute for idle capacity — solves slow-boot-time scaling.
- **Instance refresh**: replaces ASG instances with a new launch template version in a controlled, rolling fashion, respecting `MinHealthyPercentage` and an optional warmup — used to roll out AMI/config changes without downtime. **Distractor**: this is different from a Blue/Green CodeDeploy deployment — instance refresh only manages the ASG's own instance replacement, not app-level traffic shifting.

**Application Auto Scaling — the unified control plane**

Application Auto Scaling (distinct product from EC2 Auto Scaling) is the scaling engine behind: ECS services, DynamoDB tables/GSIs, Aurora Replicas, Spot Fleet, EMR clusters, AppStream 2.0 fleets, SageMaker endpoint variants, Lambda provisioned concurrency, Comprehend, Keyspaces, Neptune, custom resources.

| Cooldown default | Applies to |
|---|---|
| **300 seconds** (both scale-in and scale-out) | ECS services, Spot Fleet, EMR clusters, AppStream 2.0 fleets, Aurora DB clusters, SageMaker endpoint variants, custom resources |
| **0 seconds** | DynamoDB tables/GSIs, Lambda provisioned concurrency, Comprehend endpoints |

> **Exam trap**: assuming every Application Auto Scaling target uses the 300s default. DynamoDB and Lambda provisioned concurrency default to **0s** — this is a frequently-tested distinction.

**ECS-specific scaling — two independent layers**
- **Service Auto Scaling** (Application Auto Scaling): adjusts the *desired count* of tasks based on CPU/memory/ALB request count.
- **Cluster Auto Scaling** (Capacity Providers): adjusts the *underlying EC2 capacity* the tasks run on. Managed scaling uses a target capacity % and a Capacity Provider Reservation metric. Managed termination protection prevents scale-in from terminating instances that still have running tasks. (Fargate uses Fargate/Fargate Spot capacity providers — no cluster scaling needed.)

**Lambda concurrency scaling**
- **Burst concurrency**: initial burst allowance before Lambda linearly increases at 500 instances/min (varies by region — some regions have 500-3000 initial burst).
- **Reserved concurrency**: caps *and* guarantees concurrency for a function — carved out of the account-level pool (default 1,000/region).
- **Provisioned concurrency**: pre-initializes execution environments to eliminate cold starts — can itself be scaled via Application Auto Scaling target tracking.

---

### Skill 2.1.2 — Caching to enhance scalability

**ElastiCache: Redis (OSS/Valkey) vs Memcached**

| Feature | Redis (or Valkey) | Memcached |
|---|---|---|
| Data structures | Strings, lists, sets, sorted sets, hashes, streams, geospatial | Simple key-value strings only |
| Multi-AZ / replication | Yes — replication groups, automatic failover | No native replication |
| Persistence (snapshots/AOF) | Yes | No — pure in-memory, data lost on node failure/restart |
| Transactions / Pub-Sub | Yes | No |
| Scaling | Cluster mode: shard (scale out) + replicas (scale reads) | Horizontal scale-out via multiple nodes (client-side sharding) |
| Multi-threaded | No (single-threaded per shard, though I/O can be multi-threaded in newer versions) | Yes, natively multi-threaded — can use multiple cores per node |
| Use case signal | Need durability, complex data types, leaderboards, session store with failover | Need pure simplicity, multi-threading, no persistence required |

**DAX vs ElastiCache — high-value distractor pair**

| | DAX | ElastiCache |
|---|---|---|
| Purpose | Purpose-built, DynamoDB-native cache | General-purpose in-memory cache for any data source |
| Integration | Drop-in with DynamoDB SDK — near-zero app code changes | Requires app-level cache-aside logic |
| Latency | Microsecond | Sub-millisecond, but requires manual integration |
| Write-through | Yes, automatic | Manual (app writes to cache and DB) |
| Exam signal | "Cache DynamoDB reads with minimal code change" → **DAX** | "Cache computed/aggregated results, session data, or non-DynamoDB queries" → **ElastiCache** |

**CloudFront caching mechanics**
- TTL controlled via `Cache-Control`/`Expires` headers or CloudFront cache policy min/max/default TTL.
- **Invalidation** (`aws cloudfront create-invalidation`) forces re-fetch from origin — costs money past the free tier (1,000 paths/month free) and isn't instant across all edge locations. **Distractor**: for frequently-changing content, versioned object keys (cache-busting via filename) are the "least cost/most reliable" answer over repeated invalidations.
- **Origin Shield**: an additional caching layer in front of your origin to reduce origin load and improve cache hit ratio, especially for multi-region distributions.

---

### Skill 2.1.3 — Scaling AWS managed databases

**RDS read replicas**

| Engine | Max read replicas |
|---|---|
| Aurora | 15 (Aurora Replicas, low replica lag via shared storage) |
| MySQL, MariaDB, PostgreSQL, Oracle | 5 |
| SQL Server | 5 (Enterprise Edition, since RDS support added) |

- Replication is **asynchronous** — replica lag possible; monitor `ReplicaLag` CloudWatch metric.
- Cross-Region read replicas supported for all engines above — used for DR/read-locality, **not** for synchronous HA.
- A read replica can be **promoted** to a standalone writable instance (breaks replication permanently) — common DR/migration answer.
- **Distractor**: read replicas scale *read* throughput; they are not a Multi-AZ HA mechanism by themselves (though a replica can be Multi-AZ itself, and can be promoted during a DR event).

**Aurora-specific scaling**
- **Aurora Auto Scaling**: Application Auto Scaling adds/removes Aurora Replicas based on CPU or connections.
- **Aurora Serverless v2**: scales compute in fine-grained **ACUs** (Aurora Capacity Units), min as low as 0.5 ACU, scales in seconds without connection drops — unlike Serverless v1, it has **no pause/cold-start** capability and can be used in a cluster alongside provisioned instances.

**DynamoDB scaling**

| Mode | Behavior |
|---|---|
| On-Demand | Pay-per-request, scales instantly to traffic, no capacity planning — "least operational overhead" answer for unpredictable/spiky traffic |
| Provisioned + Auto Scaling | Application Auto Scaling adjusts RCU/WCU via target tracking; **default target utilization is 70%** |
| Adaptive capacity | Automatically boosts throughput to hot partitions without manual intervention (built-in, always on) |
| Global Tables | Multi-Region, multi-active replication — not a "scaling" feature per se but tested alongside for global read/write scaling and DR |

**RDS Proxy**
- Sits between application and DB, pools/multiplexes connections — solves connection exhaustion from Lambda or many short-lived clients.
- Reduces failover time for RDS/Aurora because the proxy maintains connections and re-routes internally instead of waiting on DNS TTL propagation.
- Requires Secrets Manager for credential management (or IAM auth) — **cannot** use plaintext DB credentials in the RDS Proxy target config.

---

## Task 2.2: Highly Available and Resilient Environments

### Skill 2.2.1 — ELB and Route 53 health checks

**Load balancer types — health check behavior**

| | ALB | NLB | GWLB |
|---|---|---|---|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP/TLS) | 3 (GENEVE) |
| Health check protocol | HTTP/HTTPS (status code match) | TCP/HTTP/HTTPS | TCP/HTTP/HTTPS |
| Cross-zone load balancing | **On by default**, no extra charge | **Off by default** — enabling can incur inter-AZ data transfer charges | Off by default |
| Idle timeout | 60s default (1–4000s configurable) | TCP: 350s fixed; not configurable | N/A |
| Deregistration delay (connection draining) | **300s default** | 300s default | 300s default |

**ELB health checks vs Route 53 health checks — core distractor**
- **ELB target group health checks** control which registered targets receive traffic *within* a load balancer — local, layer 4/7 decision.
- **Route 53 health checks** control DNS-level failover *across* endpoints/regions (e.g., failover routing policy, or removing an unhealthy resource from weighted/latency/multivalue responses) — global, DNS decision. Route 53 can also monitor a **CloudWatch alarm** as a health check (useful for monitoring resources with no public endpoint, like RDS).

**Route 53 health check mechanics (verified current numbers)**

| Setting | Values |
|---|---|
| Request interval | Standard: 30s (default) · Fast: 10s |
| Failure threshold | 1–10 consecutive checks; **default is 3** |
| Minimum detection time (default config) | 30s × 3 = ~90s |
| Minimum detection time (fastest config) | 10s × 1 = ~10s |
| HTTP/HTTPS response time requirement | 4 seconds |
| TCP connection requirement | Within 10 seconds |
| Healthy checker consensus | >18% of global health checkers must agree endpoint is healthy |
| Health check types | Endpoint, Calculated (health check of health checks, with a threshold of how many children must be healthy), CloudWatch alarm-based |

**Route 53 routing policies**

| Policy | Use case |
|---|---|
| Simple | Single resource, no health check logic |
| Weighted | % traffic split — canary releases, A/B |
| Latency-based | Route to Region with lowest latency |
| Failover | Active-passive DR — requires health checks |
| Geolocation | Route by user's geographic location (compliance, content localization) |
| Geoproximity (Traffic Flow only) | Route by geographic distance, with a "bias" to shift more/less traffic to a region |
| Multivalue answer | Return up to 8 healthy records — simple DNS-level load distribution + health check, not a substitute for a real load balancer |

---

### Skill 2.2.2 — Fault-tolerant systems (Multi-AZ)

**RDS Multi-AZ DB Instance vs Multi-AZ DB Cluster — the single highest-value Domain 2 distractor pair**

| | Multi-AZ DB Instance | Multi-AZ DB Cluster |
|---|---|---|
| Standby readability | **Not readable** — pure standby | **2 readable** standby instances (readers) |
| Replication | Storage-level, synchronous | Engine-native (logical) replication to both readers, **quorum-based**: write requires ack from ≥1 reader |
| Failover time | **60–120 seconds** typical | **Under 35 seconds** typical (faster because standbys already have a warmed, near-current state via active replication) |
| Supported engines | MySQL, MariaDB, PostgreSQL, Oracle, SQL Server | **MySQL and PostgreSQL only** |
| With RDS Proxy | Reduces failover impact | Can reduce failover downtime to **~1 second or less** for minor version upgrades |
| Read scaling | None from standby (need separate read replicas) | Yes — reader instances can serve read traffic directly |

> **Exam signal phrase**: "fastest failover," "minimal write downtime," "read scaling + HA in one deployment" → Multi-AZ DB **Cluster**. "Any RDS engine," "simplest HA, no read traffic needed" → Multi-AZ DB **Instance**.

**EC2 placement groups**

| Type | Behavior | AZ constraint | Use case |
|---|---|---|---|
| Cluster | Instances packed close together, low-latency/high-throughput networking (up to 10 Gbps) | Single AZ | HPC, tightly-coupled workloads |
| Spread | Instances on distinct underlying hardware, max 7 per AZ per group | Can span multiple AZs | Small number of critical instances that must not share a failure domain |
| Partition | Instances grouped into partitions (up to 7 per AZ) on separate racks, no shared power/network within a partition | Can span multiple AZs | Distributed workloads (Hadoop, Cassandra, Kafka) needing partition-aware placement |

**Aurora storage architecture (why Aurora is inherently fault-tolerant)**
- Data replicated **6 ways across 3 AZs** automatically.
- **Quorum**: writes require 4/6 copies to ack; reads require 3/6 copies — tolerates loss of an entire AZ (2 copies) without write impact, and loss of one additional copy without read impact.
- Storage self-heals — no snapshot restore needed for storage-layer failures.

---

## Task 2.3: Backup and Restore Strategies

### Skill 2.3.1 — Automate snapshots and backups

**AWS Backup — centralized backup orchestration**

| Concept | Detail |
|---|---|
| Backup plan | Defines schedule, lifecycle (transition to cold storage, expiration), and backup window |
| Backup vault | Logical container for recovery points; supports encryption at rest via KMS |
| **Vault Lock** | WORM (write-once-read-many) enforcement — two modes: **Compliance mode** (immutable, cannot be altered/deleted even by root, cannot shorten retention) vs **Governance mode** (can be altered by users with special IAM permission — reversible) |
| Cross-account/cross-Region copy | Native — copy recovery points to another account/Region as part of the backup plan for DR |
| Supported resources | EC2, EBS, RDS/Aurora, DynamoDB, EFS, FSx, Storage Gateway, DocumentDB, Neptune, S3 (backup) |
| Org-wide governance | AWS Backup **policies via AWS Organizations** enforce backup plans across all member accounts — key answer for "central compliance across many accounts" |

**EBS snapshots — DLM vs AWS Backup**
- **Data Lifecycle Manager (DLM)**: EBS-and-AMI-specific, free, simpler tag-based scheduling — good when you only need EBS/AMI lifecycle policies and nothing else.
- **AWS Backup**: multi-service, centralized, supports Vault Lock/cross-account/cross-Region, more auditable — the "least operational overhead" answer when backing up more than just EBS.
- EBS snapshots are **incremental** (only changed blocks after the first full snapshot) but restore as a full new volume immediately usable — **Fast Snapshot Restore (FSR)** pre-warms a snapshot so restored volumes have full performance immediately (avoids the normal lazy-load first-access latency penalty).

**RDS automated backups**

| Setting | Value |
|---|---|
| Retention period | 0–35 days (0 **disables** automated backups) |
| Backup window | User-configurable or system-assigned |
| Automated backup vs manual snapshot | Automated backups are **deleted** when the instance is deleted (unless you check "retain automated backups"/final snapshot); manual snapshots persist indefinitely until explicitly deleted |
| Transaction logs | Uploaded to S3 roughly every **5 minutes**, enabling PITR |

**DynamoDB backup types**
- **On-demand backups**: full point-in-time snapshot, no performance impact, retained until explicitly deleted, good for long-term/compliance archival.
- **PITR**: continuous, no manual trigger needed (see restore section below).

---

### Skill 2.3.2 — Restore methods to meet RTO/RPO

| Service | Restore mechanism | Granularity | Result of restore |
|---|---|---|---|
| RDS/Aurora | PITR | ~5 minutes (transaction log upload interval) | **New** DB instance — original is untouched |
| Aurora (MySQL-compatible only) | **Backtrack** | Configurable window up to **72 hours** | Rewinds the *existing* cluster in place — much faster than restore-from-snapshot, no new instance |
| DynamoDB | PITR | Per-second, within configured recovery period | **New** table |
| DynamoDB | On-demand backup restore | Point of backup | New table |
| EBS | Snapshot restore | Point of snapshot | New volume |
| S3 | Versioning rollback | Per-object-version | In place (previous version becomes current, or is copied over) |

**DynamoDB PITR — recently changed, exam-relevant**
- Default recovery period: **35 days**.
- As of a 2025 launch, the recovery period is now **configurable per table from 1–35 days** (previously fixed at exactly 35 days) — useful for compliance requirements demanding a *shorter* guaranteed window, or cost/simplicity reasons. Pricing is based on table size, not the chosen retention, so shortening it doesn't reduce PITR cost.
- `LatestRestorableDateTime` is typically **5 minutes** before "now" — you cannot restore to the most recent few minutes of writes.

**RTO/RPO cost-speed spectrum** (ties directly into Skill 2.3.4 DR strategies below — memorize this ordering, it's tested from both the backup-mechanism angle and the DR-strategy angle):

Lower RTO/RPO (closer to zero data loss, near-instant recovery) = higher cost/complexity. Higher RTO/RPO (tolerate hours of downtime/data loss) = lower cost.

---

### Skill 2.3.3 — Versioning for storage services

**S3 Versioning**

| Behavior | Detail |
|---|---|
| States | Unversioned → **Enabled** → **Suspended** (you can never fully "disable" once enabled — only suspend, which stops creating new versions but keeps existing ones) |
| MFA Delete | Requires MFA to permanently delete a version or change versioning state — extra layer against accidental/malicious deletion; can only be enabled/disabled by the **bucket owner root account** via CLI (not console) |
| Lifecycle + versioning | Add **noncurrent version transition/expiration** rules — separate from current-version lifecycle rules; critical for controlling storage cost growth from version accumulation |
| Delete marker | Deleting a versioned object doesn't erase data — it inserts a delete marker; removing the delete marker "restores" the object |

**S3 Object Lock (distinct from versioning, but requires versioning enabled)**

| Mode | Behavior |
|---|---|
| Governance mode | Users with `s3:BypassGovernanceRetention` permission **can** override/delete | 
| Compliance mode | **No one**, including root, can override or delete until retention expires — true WORM |
| Retention period | Fixed date-based lock |
| Legal hold | Independent of retention period — on/off switch, no expiration, requires explicit removal |

**FSx versioning/point-in-time equivalents**
- **FSx for Windows File Server**: uses **Shadow Copies (VSS)** — Windows-native previous-version file recovery, scheduled via Task Scheduler on the file system.
- **FSx for NetApp ONTAP**: native **Snapshots** — instant, low-cost, can be automated via ONTAP snapshot policies; SnapMirror for cross-Region replication (DR).
- **FSx for Lustre**: no native versioning — designed for transient/scratch data; data repository tasks sync back to S3 as the durability layer.

---

### Skill 2.3.4 — Disaster recovery procedures

**The four canonical DR strategies (RTO/RPO/cost spectrum — cheapest/slowest to fastest/most expensive)**

| Strategy | RTO | RPO | Description | Typical AWS implementation |
|---|---|---|---|---|
| **Backup & Restore** | Hours | Hours (since last backup) | Data backed up, infrastructure provisioned only after disaster declared | AWS Backup / S3 cross-Region replication + CloudFormation to stand up infra on demand |
| **Pilot Light** | 10s of minutes | Minutes | Core data services (e.g., DB replica) always running in DR region; compute is stopped/minimal, scaled up on failover | RDS cross-Region read replica kept warm + AMIs ready; ASG scaled to 0 until needed |
| **Warm Standby** | Minutes | Seconds–minutes | Scaled-down but **fully functional** copy of production always running in DR region; scale up on failover | Reduced-capacity ASG + RDS replica already running, Route 53 failover routing |
| **Multi-Site Active/Active** | Near-zero | Near-zero | Full production workload running in ≥2 regions simultaneously, serving live traffic | DynamoDB Global Tables / Aurora Global Database + Route 53 latency or weighted routing across regions |

**AWS Elastic Disaster Recovery (DRS)**
- Continuous, **block-level** replication from source servers (on-prem or other cloud) to a low-cost staging area in AWS.
- On failover, launches full-spec EC2 instances from replicated data — RPO of seconds, RTO of minutes. This is the direct successor to CloudEndure and is the primary exam answer for "lift-and-shift DR of on-premises/other-cloud servers into AWS."

**Route 53 Application Recovery Controller (ARC)**
- **Readiness checks**: continuously audits whether DR resources (capacity, configuration) are actually ready to take over — catches "your standby would fail if you failed over right now" before it's an emergency.
- **Routing controls**: simple on/off switches wired into Route 53 health checks, letting you manually/programmatically force traffic shift during an incident, independent of automated health check evaluation.
- **Zonal shift / Zonal autoshift**: shifts traffic away from an impaired **Availability Zone** within a Region quickly (minutes) without a full regional failover — useful for AZ-level (not Region-level) events, and autoshift can trigger automatically based on AWS's own internal health signals.

---

## Consolidated Key Numbers — Domain 2

| Value | Number |
|---|---|
| EC2 ASG default cooldown | 300s |
| Application Auto Scaling default cooldown (ECS, Spot Fleet, EMR, AppStream, Aurora clusters, SageMaker) | 300s |
| Application Auto Scaling default cooldown (DynamoDB, Lambda provisioned concurrency, Comprehend) | 0s |
| DynamoDB auto scaling default target utilization | 70% |
| Lifecycle hook default timeout / max with heartbeat | 3600s (1 hr) / 172,800s (48 hrs) |
| ALB idle timeout default | 60s |
| NLB TCP idle timeout | 350s (fixed) |
| ELB deregistration delay default | 300s |
| Route 53 health check interval | 30s standard / 10s fast |
| Route 53 failure threshold default | 3 |
| Route 53 HTTP(S) response requirement | 4s |
| RDS Multi-AZ **instance** failover | 60–120s |
| RDS Multi-AZ **cluster** failover | <35s |
| RDS Multi-AZ cluster + RDS Proxy minor version upgrade downtime | ~1s or less |
| Max RDS read replicas (Aurora) | 15 |
| Max RDS read replicas (MySQL/MariaDB/PostgreSQL/Oracle/SQL Server) | 5 |
| Aurora storage replication | 6 copies / 3 AZs (4/6 write quorum, 3/6 read quorum) |
| RDS automated backup retention range | 0–35 days |
| RDS transaction log upload interval (PITR granularity) | ~5 minutes |
| Aurora Backtrack max window | 72 hours |
| DynamoDB PITR retention | 1–35 days, configurable (default 35) |
| DynamoDB PITR restore point lag | ~5 minutes behind "now" |
| EFS lifecycle IA transition options | 1, 7, 14, 30, 60, 90, 180, 270, 365 days |
| EFS default lifecycle (recommended settings) | IA after 30 days, Archive after 90 days |
| Placement group spread — max instances/AZ | 7 |
| Placement group partition — max partitions/AZ | 7 |

---

Want to move straight into scenario-based practice questions for Domain 2, or head into Domain 4's remaining skills next?

