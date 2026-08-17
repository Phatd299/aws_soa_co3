I'll pull together a comprehensive Domain 1 reference at the depth you've been using — dense tables, exact numbers, and distractor pairs. Let me verify a handful of numbers that AWS tends to update (EBS gp3/io2 limits, Performance Insights retention, EventBridge quotas) before I lock them in.# Domain 1: Monitoring, Logging, Analysis, Remediation & Performance Optimization (22%)

One flag before we start: **RDS Performance Insights is being folded into CloudWatch Database Insights.** As of June 30, 2026, the classic Performance Insights console, its flexible retention tiers, and separate PI pricing are being retired in favor of CloudWatch Database Insights (Standard/Advanced modes). Since the exam guide still lists "Performance Insights" by name, expect both terms to appear — I've noted this under 1.3.5.

---

## Task 1.1 — Metrics, Alarms, and Filters

### Skill 1.1.1 — CloudWatch, CloudTrail, AMP

| Service | Core mechanic | Exam-critical detail |
|---|---|---|
| **CloudWatch Metrics** | Namespace + dimensions + timestamp + value | EC2 basic monitoring = 5-min datapoints; **detailed monitoring** = 1-min (paid, enabled per-instance or via launch template). High-resolution custom metrics can go to 1-second. |
| **Metric retention** | Automatic downsampling | <60s data kept 3 hrs; 60s (1-min) data kept **15 days**; 5-min data kept **63 days**; 1-hour data kept **15 months**. This is why a 13-month-old dashboard suddenly looks "chunkier." |
| **CloudWatch Logs retention** | Per log group setting | Options: 1,3,5,7,14,30,60,90,120,150,180,365,400,545,731,1096,1827,2192,2557,2922,3288,3653 days, or **Never Expire** (default — logs kept forever and billed forever until you set a policy). |
| **CloudTrail Event history** | Default, always-on | Last **90 days** of **management events only** — no S3 bucket needed, searchable in console, no data/insight events. |
| **CloudTrail Trail** | Must be explicitly created | Needed for >90 days retention, **data events** (S3 object-level, Lambda invoke), and **Insight events** (anomaly detection on write API call volume). Delivers to S3, optionally CloudWatch Logs. |
| **Trail types** | Single-Region vs. all-Region vs. **Organization trail** | Org trail (created in mgmt account) applies to all member accounts; member accounts can't disable/delete it. |
| **Log file integrity validation** | SHA-256 + RSA digest files | Detects tampering/deletion of delivered log files; digests delivered to a separate S3 prefix each hour. |
| **AMP (Amazon Managed Prometheus)** | Workspace-based, PromQL | Ingests via **remote_write**; pairs with Amazon Managed Grafana for visualization; used for container/EKS metrics beyond what Container Insights natively exposes. |

**Distractor pair:** *"Need to see who called TerminateInstances 6 months ago"* → CloudTrail **Event history alone won't work** (90-day cap) — you need a **Trail** delivering to S3 (or Athena query against it).

---

### Skill 1.1.2 — CloudWatch Agent (EC2 / ECS / EKS)

| Concept | Detail |
|---|---|
| **What EC2 gives you for free (no agent)** | CPUUtilization, NetworkIn/Out, DiskReadOps/WriteOps (device-level, not filesystem %), StatusCheckFailed. **Memory and disk-space-used % are NOT available without the agent** — classic exam trap. |
| **Unified CloudWatch Agent** | Single agent replaces the legacy "CloudWatch Logs agent" and old "SSM Agent metrics" script; collects both logs and detailed OS metrics (mem, disk, swap, processes). |
| **Install/config path** | Installed via SSM **Run Command** or bundled in AMI → config generated via `amazon-cloudwatch-agent-config-wizard` → JSON config stored locally or centrally in **SSM Parameter Store** → agent started/controlled via `amazon-cloudwatch-agent-ctl`. |
| **IAM requirement** | Instance profile needs `CloudWatchAgentServerPolicy` (to push metrics/logs) — agent will silently fail to publish without it, not throw a blocking error. |
| **procstat plugin** | Per-process CPU/memory metrics — used when you need to watch a *specific application process*, not just host-level stats. |
| **ECS** | Two paths: (1) run CloudWatch agent as a sidecar container reading task metadata endpoint, or (2) enable **Container Insights** (now "Container Insights with enhanced observability") for automatic task/service/cluster-level metrics without manual agent config. |
| **EKS** | Container Insights via CloudWatch agent + Fluent Bit deployed as a **DaemonSet**; Fluent Bit ships container logs, CloudWatch agent ships metrics (CPU/mem/disk/network at pod, node, cluster level). |

**Distractor pair:** Exam gives "instance CPU looks fine in CloudWatch but the customer says the app OOM-killed" → answer is almost always **"the CloudWatch agent isn't installed/configured to report memory metrics"** — not an alarm misconfiguration.

---

### Skill 1.1.3 — Alarms, Composite Alarms, EventBridge Actions

| Concept | Detail |
|---|---|
| **Alarm states** | `OK`, `ALARM`, `INSUFFICIENT_DATA` (not enough data points yet, or metric stopped reporting). |
| **Evaluation** | Defined by **Period** × **Evaluation Periods** × **Datapoints to Alarm** (the "M out of N" model) — e.g., 3 out of 5 periods breaching triggers ALARM, reduces false positives from single spikes. |
| **Treat Missing Data** | 4 options: `missing` (default, doesn't affect state), `notBreaching` (treated as OK), `breaching` (treated as ALARM), `ignore` (retains last state). Getting this wrong is a common scenario question. |
| **Composite alarms** | Combine multiple alarms with `AND`/`OR`/`NOT` in an **AlarmRule expression**, e.g. `ALARM(cpu-high) AND ALARM(latency-high)`. Used to **suppress noisy child-alarm notifications** and only alert on the meaningful combined condition. |
| **Alarm actions (direct)** | EC2 actions: **Stop, Terminate, Reboot, Recover**. **Recover only works on instances with these characteristics: EBS-backed, and only on supported instance types** — it moves the instance to new hardware, keeping the same instance ID/IP, used for underlying hardware failure (StatusCheckFailed_System). |
| **Alarm actions (via EventBridge)** | Anything not natively supported by direct alarm actions (e.g., invoking Lambda, Step Functions, SSM Automation) goes through **EventBridge**, since CloudWatch alarms only have a small native action list (EC2 actions, Auto Scaling, SNS, and now SSM OpsItem/Incident creation). |
| **High-resolution alarms** | Period as low as **10 or 30 seconds** — only usable against high-resolution custom metrics, incurs added cost. |

**Distractor pair:** *GuardDuty/Inspector/Config are DETECT-only by themselves* — pairing them with **EventBridge → Lambda/SSM Automation** is what gets you auto-remediation. A question asking "how do we automatically remediate a Config non-compliant finding" wants **Config remediation configuration invoking an SSM Automation document**, not just "enable Config."

---

### Skill 1.1.4 — Dashboards (cross-account, cross-Region)

| Concept | Detail |
|---|---|
| **Cross-account/cross-Region dashboards** | Requires **CloudWatch cross-account observability** — set up via **Amazon CloudWatch Observability Access Manager (OAM)**: a **monitoring account** creates a **sink**, source accounts create a **link** to that sink, specifying which telemetry types (metrics, logs, traces, X-Ray) to share. |
| **Legacy method** | Older approach used an IAM role named exactly `CloudWatch-CrossAccountSharing-*` — largely superseded by OAM but can still appear as a distractor "older" answer. |
| **Dashboard body** | Defined as JSON (widgets array) — can be created/updated via API/CLI (`put-dashboard`) for IaC-managed dashboards. |
| **Widget types** | Metric graphs, **alarm status widgets**, text (markdown), and **log table/log query widgets** (embed a Logs Insights query directly in a dashboard). |
| **Sharing** | Dashboards can be **shared publicly (unauthenticated read-only link)** or shared within account via IAM permissions — public sharing is a distinct, explicit opt-in feature (`PutDashboard` + sharing settings), not default. |

---

### Skill 1.1.5 — SNS Notification Integration

| Concept | Detail |
|---|---|
| **Subscription protocols** | Email, Email-JSON, SMS, Lambda, SQS, HTTP/S, mobile push (APNs/FCM), and **another SNS topic via subscription (fan-out)**. |
| **Alarm → SNS flow** | Alarm action on state transition (e.g., OK→ALARM) publishes to an SNS topic; requires the topic to have a **resource policy allowing `cloudwatch.amazonaws.com`** to publish (auto-added by console, must be manual via CLI/CFN). |
| **Filter policies** | Subscription-level **message filtering** so a single topic can fan-out selectively based on message attributes — avoids building N separate topics. |
| **Delivery retries** | SNS has built-in retry policies per protocol (e.g., HTTP/S has configurable backoff); failed deliveries after exhausting retries can go to a **dead-letter queue (DLQ)** if configured on the subscription. |
| **Encryption** | SNS supports **SSE with KMS** — but note Lambda/SQS subscribers to an encrypted topic need `kms:Decrypt` permission, a common "why isn't my subscriber receiving messages" trap. |

---

## Task 1.2 — Identify and Remediate Issues

### Skill 1.2.1 — Analyze & Automate Remediation

| Pattern | Mechanics |
|---|---|
| **Alarm → remediation chain** | CloudWatch Alarm (metric breach) → EventBridge rule (or direct alarm action) → target (Lambda function, SSM Automation, Auto Scaling policy). |
| **AWS User Notifications** | Newer service that **aggregates and centralizes notifications** across many AWS services (CloudWatch, Health, Budgets, etc.) into a single hub with configurable delivery channels (email, chat apps, SNS) — distinct from raw SNS: it groups/de-dupes and gives a unified console. |
| **Auto Scaling remediation** | **Target tracking** (maintain a metric at a set value, e.g., 50% CPU), **step scaling** (different scaling amounts based on alarm breach magnitude), **simple scaling** (one adjustment, then cooldown), **predictive scaling** (ML forecast + scheduled scaling ahead of predicted load). |
| **Systems Manager role** | Central automation engine — Automation documents/runbooks triggered by EventBridge, Config rules, or manually for remediation actions (restart service, patch, quarantine instance, etc.) |

**Distractor pair:** Target tracking vs step scaling — target tracking = "keep this metric near X" (simplest, least operational overhead — strong exam signal); step scaling = need **granular control over how much to scale per alarm breach tier**.

### Skill 1.2.2 — EventBridge Routing, Enrichment, Troubleshooting

| Concept | Detail |
|---|---|
| **Bus types** | Default bus (AWS service events + custom `PutEvents`), custom bus (your own event bus), partner bus (SaaS integrations like Zendesk/Datadog). |
| **Rule matching** | **Event pattern** (JSON content-based filtering on event fields) OR **schedule expression** (`rate()` / `cron()`) — scheduled rules **only work on the default bus**. |
| **Targets & quotas** | **Max 5 targets per rule** (hard limit, not raise-able) — need more fan-out? Point the rule at **SNS** and let SNS fan out further, or use **Step Functions**. **300 rules per bus by default (raisable up to 2,000)**. |
| **Retry/DLQ** | Configurable retry policy (max age of event, max retry attempts) per target; failed-after-retries events can route to a target-level **DLQ (SQS)**. |
| **Enrichment** | **Input Transformer** reshapes the event JSON before sending to target; **Input Path** extracts a sub-portion of the event (e.g., just `$.detail`). |
| **Archive & Replay** | Archive events matching a pattern for later **replay** into the bus — useful for reprocessing after a downstream outage. |
| **Cross-account** | Requires a **resource-based policy on the event bus** granting the sending account `events:PutEvents`; target account's rule then routes it — a frequent "events aren't arriving" root cause. |
| **Troubleshooting checklist** | (1) Event pattern doesn't actually match (test with `TestEventPattern` API) → (2) target IAM role/resource policy missing permission → (3) target quota/throttling → (4) cross-account bus policy missing. |

### Skill 1.2.3 — SSM Automation Runbooks

| Concept | Detail |
|---|---|
| **Document types** | AWS-owned (`AWS-*` prefix, e.g., `AWS-RestartEC2Instance`) vs. custom-authored (YAML or JSON), stored as SSM Documents. |
| **Common step actions** | `aws:runCommand`, `aws:invokeLambdaFunction`, `aws:executeAwsApi`, `aws:branch` (conditional logic), `aws:approve` (manual approval gate — pauses automation until an approver acts, useful for change-controlled remediation), `aws:executeScript`. |
| **Invocation paths** | Manually, via **Maintenance Windows** (scheduled), via **EventBridge rule**, or as the **remediation action attached to an AWS Config rule** (auto-remediate non-compliant resources). |
| **Rate control** | `MaxConcurrency` and `MaxErrors` parameters control how many targets run in parallel and how many failures abort the rest of the run — important for blast-radius control across a fleet. |
| **Outputs** | Each step can produce outputs referenced by later steps (`{{ StepName.OutputName }}`). |

---

## Task 1.3 — Performance Optimization

### Skill 1.3.1 — Compute Optimization

| Tool | What it does |
|---|---|
| **Compute Optimizer** | ML-based rightsizing recommendations for **EC2 instances, Auto Scaling groups, EBS volumes, Lambda functions, and ECS on Fargate** — needs 14+ days (up to 3 months for higher confidence) of CloudWatch metric history; requires opt-in per account/org. |
| **Resource tags** | Cost/usage attribution and automation targeting (e.g., SSM Automation targeting resources by tag) — **tag policies** in AWS Organizations enforce naming/casing consistency. |
| **Trusted Advisor** | Performance category checks (e.g., high-utilization EC2, service limits nearing quota, EBS/RDS underutilization) — depth of checks depends on **Support plan tier** (full check set needs Business/Enterprise Support). |

### Skill 1.3.2 — EBS Performance

| Volume type | Baseline / max | Metric to watch |
|---|---|---|
| **gp2** | 3 IOPS/GB baseline, burst to 3,000 IOPS via burst-bucket credits, max 16,000 IOPS at 5,334 GB+ | **BurstBalance** (running low = imminent performance cliff) |
| **gp3** | **3,000 IOPS / 125 MB/s baseline included free**, independently provisionable up to **80,000 IOPS and 2,000 MB/s** per volume (2025 limit increase) — IOPS/throughput decoupled from size, unlike gp2 | **VolumeQueueLength**, no BurstBalance (not credit-based) |
| **io1** | Provisioned IOPS, up to 64,000 IOPS/1,000 MB/s, up to 50 IOPS/GB ratio | Sub-ms consistency for latency-sensitive DBs |
| **io2 / io2 Block Express** | Up to **256,000 IOPS / 4,000 MB/s**, 99.999% durability (100x gp3's), up to 1,000 IOPS/GB, Multi-Attach up to 16 Nitro instances in the same AZ | Highest tier — SAP HANA, Oracle, large SQL Server OLTP |
| **st1** (throughput HDD) | Up to ~500 MB/s per volume, throughput-credit bucket model | Big data, log processing — **cannot be a boot volume** |
| **sc1** (cold HDD) | Lowest cost, infrequent access, throughput-credit model | Archival-adjacent, infrequently accessed data |

**Distractor pair:** BurstBalance exists on **gp2, st1, sc1** (credit-bucket types) — **not on gp3 or io1/io2** (provisioned, not credit-based). If a scenario says "gp3 volume randomly throttling," burst credits are the wrong diagnosis — look at **provisioned IOPS/throughput ceiling** or **instance-level EBS bandwidth** (EBS-optimized limit) instead.

### Skill 1.3.3 — S3 Performance Strategies

| Tool | Use case | Key detail |
|---|---|---|
| **DataSync** | Large-scale online transfer, on-prem ↔ S3/EFS/FSx, or S3-to-S3 cross-Region | Agent-based (on-prem) or serverless (AWS-to-AWS); handles incremental/scheduled syncs with checksums. |
| **S3 Transfer Acceleration** | Speeds up long-distance uploads over the public internet | Routes through **CloudFront edge locations** to the nearest edge, then over the AWS backbone — bucket name **cannot contain periods**, and there's a per-request "speed comparison tool" to validate benefit before enabling. |
| **Multipart upload** | Parallel, resumable uploads | **AWS recommends for objects >100 MB**, **required for objects >5 GB** (single PUT max object size is 5 GB). |
| **S3 Lifecycle policies** | Automate tiering/expiration | Transition: Standard → **Standard-IA (min 30 days in Standard first)** → Intelligent-Tiering → Glacier Instant Retrieval → Glacier Flexible Retrieval → Glacier Deep Archive. Standard-IA/One Zone-IA also carry a **minimum 30-day storage charge** and per-GB retrieval fee — early deletion before 30/90/180 days (IA/Glacier/Deep Archive respectively) incurs an early-deletion charge. |
| **Request-rate scaling** | S3 auto-partitions by key prefix | Modern S3 **auto-scales request rate**; the old "randomize key prefixes" advice is largely obsolete for request-rate purposes (still relevant only for extreme, sudden-spike edge cases) — a frequent outdated-answer distractor. |

### Skill 1.3.4 — EFS & FSx

| Service | Key config choices |
|---|---|
| **EFS Performance mode** | **General Purpose** (default, lowest latency, most workloads) vs. **Max I/O** (higher aggregate throughput/IOPS at the cost of higher per-operation latency — for highly parallelized workloads, e.g., big data across thousands of instances). **Set at creation, cannot be changed later.** |
| **EFS Throughput mode** | **Bursting** (throughput scales with size, burst credits) vs. **Provisioned** (fixed throughput independent of size, pay for what you provision) vs. **Elastic** (auto-scales up/down instantly, good for spiky/unpredictable workloads, most "least operational overhead"). |
| **EFS Storage class** | Standard (multi-AZ) vs. **One Zone** (single AZ, ~47% cheaper, no AZ redundancy) — combine with **Lifecycle Management** to auto-transition files to IA (or One Zone-IA) after N days of no access (14/30/60/90 day options), and **now with Archive class** for coldest tier. |
| **FSx for Windows File Server** | SMB protocol, Microsoft AD integration, native Windows ACLs — for lift-and-shift Windows workloads. |
| **FSx for Lustre** | HPC/ML workloads, **native S3 integration** (lazy-load objects as files, write back to S3) — sub-millisecond latencies, hundreds of GB/s throughput. |
| **FSx for NetApp ONTAP** | Multi-protocol (NFS, SMB, iSCSI), snapshot/cloning features, good migration target for on-prem NetApp. |
| **FSx for OpenZFS** | High IOPS, low-latency, snapshot/clone features, Linux-native workloads. |

### Skill 1.3.5 — RDS Monitoring & Performance

| Concept | Detail |
|---|---|
| **Performance Insights** | Samples DB load every second, shows **Average Active Sessions (AAS)** and top SQL/wait-event breakdowns. Free tier = 7 days retention; paid tiers historically allowed 1–24 months (up to 731 days). **⚠️ AWS is retiring the standalone PI console/pricing after June 30, 2026, migrating this capability into CloudWatch Database Insights (Standard/Advanced modes)** — expect the exam to still reference "Performance Insights" as the concept even as the underlying delivery mechanism shifts. |
| **Proactive Insights** | Performance Insights can surface **proactive recommendations** flagging emerging problems (e.g., a query trending toward becoming the top load contributor) before they become critical. |
| **RDS Proxy** | Connection pooling layer sitting in front of RDS/Aurora — reduces failover time (Proxy re-routes to new primary faster than a client re-resolving DNS), handles connection storms from Lambda, supports **IAM authentication** pass-through. Requires a **minimum Aurora Capacity Unit or instance class** floor to attach. |
| **Key CloudWatch RDS metrics** | `FreeableMemory` (watch for swap risk), `CPUUtilization`, `ReadIOPS`/`WriteIOPS`, `DatabaseConnections` (compare against max_connections parameter), `ReplicaLag` (read replica staleness), `FreeStorageSpace`, `DiskQueueDepth`. |
| **Enhanced Monitoring** | OS-level metrics (not DB-engine-level) at granularity down to **1 second** — distinct from Performance Insights (DB-engine/query-level) and basic CloudWatch (host-level, 60s). |

**Distractor pair:** Enhanced Monitoring (OS metrics, agent-based, 1–60s granularity) vs. Performance Insights (DB engine query/wait-event metrics) vs. standard CloudWatch RDS metrics (host-level, 60s, free) — a scenario asking "why is my query slow" wants **Performance Insights**; "why is CPU steal happening at the hypervisor level" wants **Enhanced Monitoring**.

### Skill 1.3.6 — EC2 Compute, Storage & Networking Optimization

| Concept | Detail |
|---|---|
| **Cluster placement group** | Single AZ, instances packed close together for **low-latency, high-throughput, 10 Gbps+ single-flow networking** — HPC, tightly-coupled node-to-node workloads. Higher risk of correlated hardware failure. |
| **Spread placement group** | Each instance on **distinct underlying hardware**, max **7 instances per AZ per group** — for small numbers of critical instances that must not share a failure domain. |
| **Partition placement group** | Instances divided into **up to 7 partitions per AZ**, each partition on separate hardware racks — for large distributed/replicated workloads (Hadoop, Cassandra, Kafka) that need rack-awareness without single-instance isolation. |
| **Enhanced Networking (ENA)** | Higher PPS, lower jitter/latency, lower CPU utilization for networking — required for placement group max throughput. |
| **EBS-optimized instances** | Dedicated bandwidth for EBS traffic, separate from general network throughput — most current-gen instances are EBS-optimized by default; older/smaller types may need explicit enabling and have a **baseline + burst bandwidth model** just like storage. |
| **Instance store vs. EBS** | Instance store = ephemeral, physically attached NVMe SSD, **data lost on stop/terminate/most underlying hardware failures** — highest raw IOPS, used for cache/scratch/temp data, not durable storage. |
| **EBS Multi-Attach** | io1/io2 only, same AZ, up to 16 Nitro instances — requires a **cluster-aware filesystem** (e.g., not ext4/XFS as-is) to avoid corruption from concurrent writers. |

---

## Domain 1 — Consolidated Key Numbers

| Item | Value |
|---|---|
| EC2 basic vs. detailed monitoring | 5-min (free) vs. 1-min (paid) |
| CloudWatch metric retention | 60s data → 15 days; 5-min → 63 days; 1-hr → 15 months |
| CloudTrail Event history | 90 days, management events only |
| EventBridge targets per rule | **5 max** (not raisable) |
| EventBridge rules per bus | 300 default, up to 2,000 |
| S3 multipart upload | Recommended >100 MB, **required >5 GB** |
| S3 single PUT max object size | 5 GB |
| S3 Standard-IA minimum storage duration | 30 days |
| EBS gp2 burst | 3 IOPS/GB baseline, burst to 3,000 IOPS |
| EBS gp3 baseline (free) | 3,000 IOPS / 125 MB/s |
| EBS gp3 max (provisionable) | 80,000 IOPS / 2,000 MB/s |
| EBS io2 Block Express max | 256,000 IOPS / 4,000 MB/s, 99.999% durability |
| EBS Multi-Attach | io1/io2 only, same AZ, up to 16 instances |
| Spread placement group | Max 7 instances per AZ |
| Partition placement group | Up to 7 partitions per AZ |
| RDS Performance Insights free tier | 7 days |
| RDS Performance Insights paid retention | Up to 731 days (being phased into CloudWatch Database Insights by June 30, 2026) |

---

Since Domain 1 was already marked complete in earlier sessions, want me to run a few **scenario-based practice questions** across these skills to pressure-test retention before we move back to closing out Domain 4?