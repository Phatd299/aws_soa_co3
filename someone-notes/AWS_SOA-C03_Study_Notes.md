# AWS CloudOps Engineer — SOA-C03 Study Notes

*Complete reference · All 5 exam domains*

> **Note on this conversion:** In the source PDF, the colored section-banner headers were offset by one section relative to their content (a page-layout artifact). This markdown version re-aligns each banner with its actual content so the notes match the Table of Contents below. No content was removed — only the headings were corrected and the text was cleaned up for readability.

## Exam facts

| | |
|---|---|
| Questions | 65 |
| Time | 130 minutes |
| Passing score | 720 / 1000 |
| Scored questions | 50 (15 are unscored) |

## Domain weightings

| # | Domain | Weight |
|---|--------|--------|
| 1 | Monitoring, Logging, Analysis, Remediation & Performance | 22% |
| 2 | Reliability & Business Continuity | 22% |
| 3 | Deployment, Provisioning & Automation | 22% |
| 4 | Security & Compliance | 16% |
| 5 | Networking & Content Delivery | 18% |

## Table of Contents

1. [Amazon Machine Image](#01-amazon-machine-image)
2. [AWS Systems Manager](#02-aws-systems-manager)
3. [EC2 High Availability (Multi-AZ) and Scalability](#03-ec2-high-availability-multi-az-and-scalability)
4. [CloudFormation](#04-cloudformation)
5. [Lambda](#05-lambda)
6. [Storage — EBS, EFS, S3](#06-storage--ebs-efs-s3)
7. [CloudFront](#07-cloudfront)
8. [Databases](#08-databases)
9. [Monitoring, Auditing and Performance](#09-monitoring-auditing-and-performance)
10. [AWS Account Management](#10-aws-account-management)
11. [Disaster Recovery](#11-disaster-recovery)
12. [Security and Compliance](#12-security-and-compliance)
13. [Networking — Route 53](#13-networking--route-53)
14. [VPC](#14-vpc)
15. [Other Services](#15-other-services)

---

## 01. Amazon Machine Image

### EC2 fundamentals

- Changing instance type is allowed only for EBS-backed instances. EBS optimization is enabled by default for most types (not for `t2.nano`, for example).
- **Enhanced networking (drivers):**
  - *Elastic Network Adapter (ENA):* up to 100 Gbps, for newer-generation instances (e.g. `t3.micro`).
  - *Elastic Fabric Adapter (EFA):* Linux only, built for HPC — great for instances in the same cluster.
- **Placement groups:**
  - *Cluster:* low latency, single AZ, riskier (all eggs in one basket), up to 10 Gbps if ENA is enabled — good for big-data jobs.
  - *Spread:* for critical applications, max 7 instances per AZ.
  - *Partition:* like spread but not physically isolated — each partition is a rack, up to 7 partitions per AZ.
- **Shutdown behavior (from the OS):**
  - `stop` (default)
  - `terminate` — even with termination protection enabled, the instance **will** terminate if the terminate signal comes from inside the OS (not from the CLI/API). Termination protection can never be enabled for Spot Instances.
- **Resizing an instance:**
  - Only possible if the root device is an EBS volume. If the root device is an instance store volume, you must migrate the application to a new instance of the desired type.
  - You must **stop** an EBS-backed instance before changing its instance type. AWS moves the instance to new hardware; the instance ID does not change.
  - If the instance is in an Auto Scaling group, ASG marks a stopped instance as unhealthy and may terminate + replace it.
- **Launch troubleshooting:**
  - `InstanceLimitExceeded` (max vCPU per Region) — request a quota increase or launch in a different Region. Applies only to On-Demand and Spot instances (check Service Quotas for increase limits).
  - `InsufficientInstanceCapacity` — wait, request fewer instances, or request a different instance type (capacity issue is specific to that AZ).
  - `InstanceTerminateImmediately` — corrupted EBS snapshot, EBS volume limit reached, or an encrypted EBS volume with no KMS key permissions.
- **SSH troubleshooting:**
  - `400` permissions required on the `.pem` key, otherwise "unprotected private key file".
  - "Host key not found" / "permission denied" / "connection closed" → wrong username.
  - "Connection timed out" → security group misconfigured, VPC route table issue, high CPU load, or missing IPv4 address.
  - Remember to allow an inbound rule on port 22 for the authorized IP.
  - EC2 Instance Connect (CLI) pushes a one-time key valid for 60 seconds to a published IP range — you must allow that published range in your security group.
- **Purchasing options:**
  - *On-Demand:* short, predictable workloads, billed per second.
  - *Reserved:* 1 or 3 years, long-running workloads (scope: regional or zonal).
  - *Savings Plans:* commitment to an hourly spend (e.g. $10/h), flexible across instance sizes/families.
  - *Spot:* short, cheap — workload must be resilient to interruption (data analysis, batch jobs, image processing — **never a database**).
    - You can cap a max price; if exceeded, instances are stopped/terminated after a 2-minute warning.
    - To terminate cleanly, delete the Spot request first, then terminate the instance.
    - Use **Spot Fleets** to get the best price for a target capacity; you can also define an On-Demand base capacity guaranteeing a minimum instance count for up to 6 hours.
    - **Spot Instance Advisor** is informational only — it shows historical interruption rates but does not automate pool selection or guarantee your ASG uses the most available pools.
    - **Capacity-optimized allocation strategy** is the one that actually makes Auto Scaling select the Spot pools with the most available capacity to minimize interruption risk.
  - *Dedicated Hosts:* physical server billed to you — bring your own licenses, supports compliance needs, full visibility at the hardware level.
  - *Dedicated Instances:* dedicated hardware that can still be shared with other instances in the same account; can move location after stop/start.
  - *Dedicated (bare-metal) server:* no other customer ever shares your hardware.
  - *Capacity Reservations:* reserve capacity in a specific AZ for any duration (you pay even if unused) — combine with Regional Reserved Instances/Savings Plans for a discount; good for short-term workloads tied to one AZ.
- **Burstable instances (T2/T3, e.g. `t2.micro`):** baseline CPU performance with burst credits accumulated over time; if credits run out, CPU performance drops. "Unlimited" burst mode avoids throttling but costs extra.
- **Elastic IP:** gives a fixed IP that can be remapped across instances (good for masking a failure). Max 5 per account by default (can request an increase).
- **CloudWatch metrics for EC2 (no memory or disk by default):**
  - Default: 5-minute interval. Detailed monitoring: 1-minute interval, paid.
  - Includes CPU (with burst credits), network, disk operations — **not** memory or application-level metrics.
  - Custom metrics (RAM, application-level, etc.) require the CloudWatch agent.
- **Unified CloudWatch agent:** the only way to get memory and disk (used space) metrics.
  - Collects logs and pushes them to CloudWatch Logs.
  - Configuration can be stored in SSM Parameter Store.
  - Requires an IAM role that allows pushing logs/metrics to CloudWatch.
  - Includes the **procstat** plugin — monitors individual processes by name or PID.
  - Turning on *detailed monitoring* only increases metric **frequency** (1-minute); it never adds memory metrics.
  - For an Auto Scaling group: bake the CloudWatch agent into the AMI and configure it to publish memory metrics, so every new instance reports memory automatically — this is the scalable pattern (rather than configuring each instance individually).
- **EC2 status checks** (automated hardware/software checks):
  - **System status check** — problem with the underlying host (loss of network/power, host software/hardware issues). Either wait for AWS to fix the host, or stop/start the instance (if EBS-backed) to move it to new hardware.
  - **Instance status check** — software/network configuration issue on your instance (bad startup config, exhausted memory, corrupted file system, incompatible kernel). Requires your action: reboot or reconfigure.
  - **Attached EBS status check** — hardware/software/connectivity issues on the attached EBS volume(s). Requires your action: restart the instance or replace the volume.
  - Metrics: `StatusCheckFailed` (combined), `StatusCheckFailed_Instance`, `StatusCheckFailed_System` (0 = OK, 1 = failed).
  - To automate recovery: a CloudWatch alarm on these metrics can trigger **EC2 auto-recovery**, which restores the instance with the same private IP, metadata, and placement group (preferred) — or use an Auto Scaling group, which instead launches a replacement with a different IP.
- **Warm pools:** reduce latency for applications with very long boot times (e.g. instances that must write large amounts of data to disk) without over-provisioning the Auto Scaling group.
- **Instance store vs. EBS-backed:**
  - *Instance store-backed:* local, ephemeral storage as the root device — high performance, fast boot, but data is lost on stop/terminate.
  - *EBS-backed:* persistent, network-attached root volume — must be stopped before changing instance type; AWS moves it to new hardware but keeps the instance ID.
  - To resize an instance store-backed instance: create an AMI, launch a new instance of the desired type, and attach an Elastic IP.

> **EXAM TIP:** EC2 Instance Status Checks are a must-know — expect 2–3 questions on the differences between system, instance, and EBS status checks.

- **EC2 Hibernate:** preserves RAM to a file on the EBS volume for a faster boot; useful for long-running processes that take time to initialize. Requires an encrypted root volume. Available only for On-Demand, Spot, and Reserved Instances.

### Amazon Machine Images (AMI)

- AMIs are built for a specific Region — **AMIs are Region-locked**; the same AMI ID cannot be used across Regions.
- Sources: a public AMI, your own AMI, or the AWS Marketplace (third parties sell AMIs there).
- To create an AMI: right-click the instance → *Create image*.
  - Choosing "no reboot" skips rebooting the instance but risks filesystem inconsistency (AWS Backup also doesn't guarantee integrity without a reboot).
- AMIs are used to migrate an instance between AZs (still within the same Region, since AMIs are Region-locked).
- **Cross-account AMI sharing:**
  - Works directly if the underlying EBS volume is unencrypted.
  - If encrypted, you must use a customer-managed KMS key and share that key too (grant IAM permission to use the key).
- **Cross-account AMI copy:** requires IAM permission to read the underlying EBS snapshot.
- **EC2 Image Builder:**
  - Automates the creation, validation, maintenance, and testing of EC2 AMIs (and container images).
  - Can be scheduled; you pay only for the underlying resources, not for the service itself.
  - Tag approved AMIs and attach IAM policies that force users to launch instances only from pre-approved AMIs (e.g. `Environment: prod`) — enforce with **AWS Config**.

---

## 02. AWS Systems Manager

- Manages EC2 and on-premises systems at scale.
- Integrated with AWS Config and CloudWatch.
- Used for automated patching — **does not replace AMIs**.
- Free service; requires the SSM Agent to be installed (pre-installed on Amazon Linux 2 AMIs).

> **EXAM TIP:** If a question states that Amazon Linux 2 AMI is used, favor Systems Manager–based solutions.

- Instances need the correct IAM role: attach an instance profile containing the `AmazonSSMManagedInstanceCore` policy.

**Key features:**

- Tags can be used to build resource groups. SSM is a **Regional** service.
- **SSM Documents** (YAML or JSON) define parameters and actions.
  - **Run Command** executes a single command or a document across multiple instances (e.g. a resource group) — no SSH required, the agent executes the commands.
  - Can be invoked through EventBridge.
- **Automation:**
  - Automation runbooks are documents of type "Automation"; common uses: restart instances, create AMIs/snapshots, etc. They can be triggered on a schedule or by an event.
- **Parameter Store:**
  - Secure storage for configuration values and secrets, access controlled through IAM.
  - The **Advanced** tier (paid) supports TTL on parameters and multiple policies.
- **Inventory & State Manager:**
  - *Inventory* collects metadata from managed instances.
  - *State Manager* defines and enforces a desired state (e.g. port 22 closed, antivirus installed) via a State Manager **association**, built from SSM documents.
- **Patch Manager:**
  - Manages OS, security, and application updates.
  - *Patch baselines* are applied per **patch group** (a tag, e.g. `patchGroup: dev`), so different groups of instances can use different baselines.
- **Session Manager:**
  - Gives a shell on an EC2 instance with no SSH, no keys, and no bastion host.
  - Access is controlled by IAM, and can be scoped by resource tag, e.g.:
    ```json
    "Condition": {
      "StringEquals": {
        "ec2:ResourceTag/Environment": "Production"
      }
    }
    ```
  - The instance's security group can have **no inbound rules at all**.
  - Example pattern: require EC2 access only via Session Manager → send session logs to CloudWatch Logs → create a CloudWatch metric filter + alarm on relevant security events.

---

## 03. EC2 High Availability (Multi-AZ) and Scalability

**Golden rule for the exam:** on AWS, "high availability" = spread across multiple **Availability Zones** (not multiple Regions — that's disaster recovery / multi-Region).

When you see "highly available" in a question, think:

- ≥ 2 Availability Zones
- ≥ 2 instances
- A load balancer in front
- An Auto Scaling group (to replace failed instances automatically)

You don't need to memorize exact numbers — "1 instance" is never HA; "multi-AZ + multiple instances" is.

| Concept | Key phrase |
|---|---|
| Fault tolerance / High availability | Multi-AZ |
| Disaster recovery | Multi-Region |
| Automatic scalability | Auto Scaling Group |
| Load balancing | ELB / ALB |
| Cost efficiency | Spot / On-Demand mix |
| Performance | Scaling policies, instance types |

**Cross-Region HA scenario:** to achieve HA across Regions, deploy a full copy of the stack in a second Region (e.g. `us-west-2`), add a Route 53 A record aliasing the second Region's ELB, and configure both A records with a **failover routing policy** and health checks (primary = `us-east-1` ELB, secondary = `us-west-2` ELB). Route 53 automatically stops sending traffic to an unhealthy primary. A single SOA record, or attaching EC2 instances from another Region to an existing (regional) ELB, does not work — ELBs are Regional and cannot span Regions. VPC peering alone provides no DNS-level failover.

**ASG vs. ALB health checks:** the Auto Scaling group only checks EC2 instance health; the ALB performs its own HTTP/HTTPS health check on a path/port. If the ALB's target-group health check is pointed at the wrong port or path, instances can appear "healthy" to the ASG but "unhealthy" to the ALB — this is a very common scenario question (e.g. a misconfigured target-group health check causing traffic loss despite instances looking fine).

### Load balancing

Purpose: separate public from private traffic, expose a single DNS entry point, spread load across instances, provide HA across zones.

**ELB (general):**

- Fully managed by AWS (HA, patching, maintenance handled for you), integrates with many AWS services.
- Health checks (e.g. HTTP on a given port/path) — unhealthy instances stop receiving traffic.
- The ALB does **not** send SNS notifications directly — to notify on unhealthy targets, use a CloudWatch Alarm on the ALB's published metrics, which then notifies SNS.
- Load balancer security group should allow HTTP/HTTPS from anywhere; the application's security group should only allow traffic from the load balancer.
- ASG activity notifications cover launch/termination and EC2-level unhealthy events — but a "healthy" EC2 instance can still fail the *ELB-side* health check.

**ALB (Application Load Balancer):**

- Layer 7 (HTTP/HTTPS) — great for microservices, load balancing multiple containers on the same machine, and supports WebSocket.
- **You can route traffic differently based on the path** (via listener rules): e.g. `/users` → target group 1, `/posts` → target group 2. Also supports routing by hostname, query string, or headers.
- Target groups can be EC2 instances (via ASG), Lambda functions, ECS tasks, or private IPs.
- Has a fixed hostname: `xxx.region.elb.amazonaws.com`.
- The client's IP is preserved in an HTTP header (not visible directly to the backend as the source IP).

**NLB (Network Load Balancer):**

- Layer 4 — forwards TCP/UDP traffic; very high performance (millions of requests/second).
- Provides **one static IP per AZ** (supports Elastic IP) — use it when clients need to allowlist a small, fixed set of IPs.
- Target groups can be instances or private IPs (including on-prem).
- Can sit in front of an ALB to expose a single fixed IP.
- Health checks support TCP, HTTP, and HTTPS.

**Gateway Load Balancer:**

- Used to inspect traffic with third-party virtual appliances (e.g. firewalls) before it reaches the NLB — a single entry/exit point for all traffic.
- Uses the **GENEVE** protocol on port 6081.

**Sticky sessions** (ALB and NLB): redirect the same client to the same instance.

- Cookies have an expiration date.
- Types: application cookie and duration-based cookie (both generated by the LB), and a custom cookie (generated by the target).

**CloudFront in front of an ALB:** `User → CloudFront → ALB → Target Group → EC2`

- CloudFront acts as a reverse proxy/cache layer in front of the ALB. It caches content for downloads, not uploads.
- To cache different object versions per device type, forward these headers to the origin: `CloudFront-Is-Desktop-Viewer`, `CloudFront-Is-Mobile-Viewer`, `CloudFront-Is-SmartTV-Viewer`, `CloudFront-Is-Tablet-Viewer`.
- Only forward/cache-key query strings, cookies, or headers that actually change the origin's response — forwarding ones that don't (e.g. a tracking parameter) just multiplies cache variants, increasing cache misses and cost.
- **Sticky sessions through CloudFront:** if CloudFront isn't configured to forward cookies, the ALB never receives the stickiness cookie on later requests, so users can be routed to a different instance and get logged out early. Fix: configure the CloudFront behavior to forward cookies, and align the ALB's stickiness duration with the application session (use the app cookie, or a longer duration) so the ALB doesn't expire the session before the app does.
- **Certificate expiry:** an expired certificate on the origin causes TLS errors between CloudFront and the origin, surfacing as HTTP 502s to the client. Fix: check and renew the origin certificate.

**Cross-zone load balancing:**

- Enabled by default (free) for ALB — can be disabled at the target-group level.
- Disabled by default (and charged when enabled) for NLB and Gateway Load Balancer.

SSL/TLS certificates (managed via ACM) encrypt traffic between the client and the load balancer.

**Connection draining / deregistration delay:** time given to complete in-flight requests while an instance is deregistering or unhealthy.

- Default health check port: 80.
- Errors: 4xx = client, 5xx = server.
- LB metrics are pushed to CloudWatch; access logs are stored in S3.
- **Request routing algorithms:** least outstanding requests, round robin (equal distribution), flow hash.

### Auto Scaling Group (ASG)

- Follows the load of traffic; the ASG service itself is free (you pay for the underlying resources).
- Set minimum and maximum capacity.
- Can scale in response to CloudWatch alarms.
- **Scaling policies:**
  - *Simple scaling* — triggers a scaling action on a threshold breach, always adjusting capacity by a fixed amount.
  - *Step scaling* — variable adjustment by threshold band, e.g. CPU 50–60% → +1 instance; 60–70% → +2; 70–80% → +4.
  - *Target tracking* — automatically maintains a target metric value (e.g. average CPU ≈ 40%).
  - *Predictive scaling* is also available.
- Launch templates define the instance configuration for the ASG.
- **SQS-driven scaling:** an SQS queue can feed alarms into CloudWatch to trigger scaling. `ApproximateNumberOfMessagesVisible / InService instances` reflects the backlog per active instance — this is the standard pattern for scaling SQS consumers.
- ASGs can span multiple AZs.

---

## 04. CloudFormation

- **Termination Protection** prevents accidental stack deletion.
- **StackSets** let you deploy stacks across multiple AWS accounts and Regions.
- **Custom resources** automate non-native operations, like emptying and deleting a non-empty S3 bucket.
- When a stack creation fails and reaches `ROLLBACK_COMPLETE`, that state is **terminal** — the stack cannot be updated or retried directly; it must be deleted and recreated.

**Nested stacks:**

- Break common components into reusable, dedicated templates.
- Pass parameters and define conditions per nested stack.
- Reduce duplication by referencing shared components.
- Different environments are handled via parameters.

**Compare the CloudFormation modularity/tooling features (a common exam distractor set):**

- **Change sets** — preview changes before updating a stack; do **not** modularize templates.
- **Macros** — allow custom processing/transformation of template code; not specifically for reusable components.
- **StackSets** — deploy stacks across multiple accounts/Regions; not for template reuse within a single account.

**Wait conditions & signaling:**

- CloudFormation **wait conditions** pause stack creation until resources signal successful configuration.
- If an ASG's instances aren't signaling CloudFormation once their setup finishes, the wait condition never receives the required signal count, causing the stack (and the ASG) to fail and roll back.
- `cfn-signal` is the CloudFormation helper script that sends that signal (e.g. after software install completes).
- A **CreationPolicy** on an EC2 resource makes CloudFormation wait for an explicit success signal (from `cfn-signal`) within a specified timeout (e.g. 4 hours to cover a 2–3 hour install plus buffer). If the signal isn't received in time, CloudFormation marks the resource as failed and rolls back the whole stack.
- `Conditions` do **not** provide timeouts or wait-for-signal behavior.
- `DependsOn` only enforces ordering — no timeout semantics.
- `Metadata` is descriptive only — no effect on orchestration or failure handling.

**OnFailure:**

- The `OnFailure` property of `CreateStack` controls what happens if stack creation fails: `DO_NOTHING`, `ROLLBACK`, or `DELETE`.
- You can specify either `OnFailure` or `DisableRollback`, but not both.
- Setting `OnFailure=DO_NOTHING` prevents CloudFormation from terminating the EC2 instances that were already created by the stack — useful for post-mortem debugging.

---

## 05. Lambda

- By default, a Lambda function runs **outside your VPC** and cannot reach VPC resources (RDS, Elasticsearch, etc.) unless you make that resource public with a public IP — which is not secure.
- If you configure a VPC ID, subnets, and security groups for the function, Lambda creates an ENI inside your VPC so it can reach private resources.
- Because Lambda is serverless, every invocation can open a new DB connection, which can lead to `TooManyConnectionsException` against a database.
- **RDS Proxy** solves this: place RDS Proxy in a (public or private) subnet inside your VPC; it pools and manages the actual DB connections. The Lambda function connects to the proxy instead of the database directly, and — critically — **the Lambda function itself can stay outside the VPC** while still reaching the proxy, avoiding the cold-start/ENI overhead of putting Lambda in a VPC just for DB access.

---

## 06. Storage — EBS, EFS, S3

### Amazon S3

- S3 supports cross-region replication; replication does **not** support chaining (to replicate A → B → C you must configure A → B and A → C separately).
- Enabling versioning does not retroactively version existing objects — their existing versions show as `null`.
- **Object Lock (WORM):**
  - Two mechanisms: **retention period** (fixed lock duration, can be set per-object or as a bucket default; bound with the `s3:object-lock-remaining-retention-days` condition key in a bucket policy) and **legal hold** (no expiration — stays until explicitly removed, independent of retention periods).
  - Requires S3 Versioning to be enabled; lock info is stored in the object version's metadata.
  - To delete a locked object you need both IAM permission and to pass `x-amz-bypass-governance-retention: true` in the request header.
  - **MFA Delete** can only be enabled by the **root user**, and only via the CLI.
- **Presigned URLs:**
  - Do not grant access via the end user's own credentials — the URL is signed with the **permissions of whoever generated it**. If the generator lacks `s3:PutObject`, the URL cannot be used to upload, even though it looks valid.
  - The end user needs no AWS credentials at all — `curl`, a browser, or Postman all work; all authentication is embedded in the signed link.
- **In-transit encryption:** enforce via a bucket policy that denies requests unless `aws:SecureTransport` is true (HTTPS only).
- **S3 Transfer Acceleration:** uses CloudFront's global edge locations to speed up uploads — the upload goes to the nearest edge location, then travels over AWS's optimized backbone to the bucket. Especially effective for large objects from geographically distant clients.
- **Server access logging:** detailed records of requests made to a bucket — useful for security and access audits.
- **Content-MD5 header:** lets the client supply an MD5 checksum with an upload; S3 validates it and rejects the upload if it doesn't match, catching corrupted uploads immediately. (Other headers like `Content-Disposition`, `x-amz-object-lock-mode`, or `x-amz-server-side-encryption-customer-algorithm` handle metadata/locking/encryption — not integrity verification.)
- **Enforcing default encryption:** turning on *default encryption* alone is **not sufficient** — it only encrypts objects when the client doesn't specify encryption; it doesn't stop a client from explicitly uploading unencrypted. To truly enforce it, use a **bucket policy** that denies any `PutObject` lacking server-side encryption.

### Amazon EBS

- A network drive attached to a running instance; bound to a specific AZ.
  - To move it, you need a snapshot (snapshots are stored in S3) and a new volume created from that snapshot in the target AZ.
- **Fast Snapshot Restore (FSR)** restores a volume fully initialized at maximum performance immediately — but it's expensive.
- You can set a retention period after deleting a snapshot (e.g. keep it in a "bin" for 1 month), or move it to an archive tier with Amazon Data Lifecycle Manager (cheaper, slower retrieval).
- EBS attaches to one instance at a time (except Multi-Attach volumes) and communicates over the network.
- **Delete on Termination:** disable it if you want to preserve the root volume after the instance is terminated.
- Enabling encryption also encrypts the volume's snapshots and any volume created from those snapshots.
- **Volume types:**
  - `gp2` / `gp3` — general-purpose SSD, balance of price and performance.
  - `io1` / `io2` — SSD, highest performance for critical low-latency workloads (e.g. **databases**); both support **Multi-Attach** (more than one instance, same AZ); both can be used as boot volumes.
  - `st1` — low-cost HDD for frequently accessed, throughput-heavy workloads (big data, data warehousing, log processing).
  - `sc1` (cold HDD) — lowest cost, for rarely accessed data.

> **IMPORTANT:** On `gp2`, performance (IOPS) scales with volume size — if you're bursting frequently, resize the volume to get more baseline performance.

- **EC2 Instance Store:** very high I/O performance (good for buffering/caching, or high-IOPS batch/data processing), but data is lost when the instance is stopped or terminated.

### Amazon EFS

- NFS file system, Multi-AZ, can be mounted by many instances concurrently ($$).
- **Mount targets are AZ-specific** — you must create one mount target in each AZ where your EC2 instances reside.
- "Elastic" because throughput automatically scales with your workload.
- **Lifecycle management** moves files between storage tiers: Standard, Infrequent Access, Archive.
- Backup/restore is at **file-level** granularity — unlike EBS snapshots, which are volume-level (you'd have to restore the whole volume).
- If the `PercentIOLimit` metric is consistently high (throughput/IOPS bottleneck), the fix is to switch the file system to **Provisioned Throughput** mode.

### Windows & hybrid storage

- EBS and EFS are **Linux-only**. For Windows, use **Amazon FSx** — still HA, provides fully managed Windows file shares (SMB) with simultaneous access from multiple application servers.
- **File Gateway** — SMB/NFS-based access to data stored in S3, with local caching.
- **Tape Gateway** — lets you replace physical on-prem tapes with virtual tapes in AWS, without changing existing backup workflows.
- Maintenance/reboot of a Storage Gateway VM:
  - *File Gateway* — just shut down the VM.
  - *Volume Gateway* / *Tape Gateway* — stop the gateway, reboot the VM, then start the gateway.

---

## 07. CloudFront

- A Content Delivery Network (CDN) that improves read performance by caching at edge locations.
- **Origin Shield** = an additional layer of caching in front of your origin.
- Provides DDoS protection, integrates with AWS Shield and AWS WAF.

**Origins CloudFront can use:**

- **S3 buckets** — distributes and caches files at edge locations globally; secure access via **Origin Access Control (OAC)**.
  - When you enable CloudFront on S3 as the origin, you can create a bucket policy that restricts bucket access to only that CloudFront distribution.
- **VPC origins (ALB, NLB, EC2)** — content stays private in your VPC subnets and becomes reachable only through CloudFront.
- **Custom origins** — any public HTTP backend.

**Request flow:** `Request → CloudFront → check cache → forward to origin if miss → cache the response`.

**CloudFront vs. S3 Cross-Region Replication (as a distribution mechanism):**

| | CloudFront | S3 replicas |
|---|---|---|
| Network | Global edge network | One setup per Region |
| Freshness | Cached with a TTL | Updated in near real time |
| Best for | Static content needed everywhere | Read-only, changing/dynamic content |

- You can restrict access to a list of countries (geo-restriction, backed by a third-party GeoIP database).
- Access logs go to an S3 bucket — but a **separate log bucket**, not the origin bucket.

**Caching is based on:**

1. **Headers** — forward all headers to the origin, allow a whitelist, or don't cache based on headers at all (best cache performance). You can also define custom origin headers for every request.
2. **Session cookies** (specific request headers) — default: don't process cookies; or whitelist specific cookies; or process all (worst performance, but lets the app use them freely).
3. **Query string parameters** (in the URL) — default: don't process; or whitelist; or process all (many variants to handle).

To maximize cache hits, split static and dynamic requests: serve dynamic (HTTP/REST, cookie-dependent) content with tighter caching rules, and serve static content (S3) with no special cache rules. Monitor the **Cache Hit Rate** metric.

- Only forward/whitelist the specific headers, cookies, and query strings that actually change the origin's response — otherwise you create unnecessary cache variants (lower hit rate, higher cost). See the ALB/sticky-session example in [Section 03](#03-ec2-high-availability-multi-az-and-scalability) — the `AWSALB` sticky-session cookie is a canonical case of a cookie you must whitelist and forward.

---

## 08. Databases

**Why RDS over self-managed DB on EC2:**

- Automated OS patching.
- Automated backups.
- Read replicas to improve read performance.
- Multi-AZ setup for disaster recovery.
- Storage auto scaling (vertical and horizontal) — great for unpredictable application workloads.
- It's a managed service, but you **cannot SSH** into the underlying instance.

**Encryption in transit:** e.g. for PostgreSQL on RDS, force SSL-only connections by enabling the `rds.force_ssl` parameter (default `"0"`) via a DB parameter group.

**Read replicas (Multi-AZ use cases):**

- Up to **15** read replicas, within an AZ, across AZs, or cross-Region.
- Replication is **asynchronous**.
- Use case 1 — offload a reporting/analytics workload from the primary onto a read replica. You only pay the inter-AZ/Region network cost if the replica lives outside the primary's AZ/Region.
- Use case 2 — **Multi-AZ for DR**: synchronous replication, a single DNS name that fails over automatically, and it is **not** used for read scaling. Read replicas can themselves be configured as Multi-AZ for DR purposes.

**Comparing HA strategies (a classic exam contrast):**

- *RDS Multi-AZ (MySQL) + S3 backup* — automatic synchronous replication to a standby, automatic and fast failover (usually < 1 minute), minimal data loss, no manual intervention needed. ✅ Best for HA.
- *RDS Single-AZ + read replica + S3 backup* — a Single-AZ DB has **no automatic failover** if the primary fails; a read replica is for read scaling, not HA. ❌ Not sufficient for HA.

**Going from Single-AZ to Multi-AZ:** no need to stop the DB — just click *Modify*. Internally, AWS takes a snapshot and restores a new DB from it into a new AZ, then starts synchronous replication.

**Backups vs. snapshots:**

- *Automated backups* are continuous and allow **point-in-time recovery (PITR)**; if you delete the instance, you can choose to retain them. No scheduling needed — second-level granularity, retained up to 35 days.
- *Snapshots* briefly pause I/O, are incremental, and **manual** snapshots never expire and can be shared if unencrypted (automated snapshots cannot be shared).
- **In any case, restoring from a backup or snapshot always creates a brand-new DB instance.**
- **PITR only restores within the same Region** — it does not replicate data cross-Region or provide a read/write replica. For cross-Region DR on DynamoDB specifically, use **DynamoDB Global Tables**, which use DynamoDB Streams under the hood to give multi-Region, active-active replication.

**RDS monitoring:**

- CloudWatch metrics: `DBConnections`, `SwapUsage`, read/write IOPS, latency, throughput, etc.
- **RDS Events** cover changes in DB instance *state* (separate from CloudWatch metrics).

### Amazon Aurora

- MySQL- and PostgreSQL-compatible drivers work unchanged.
- Very cloud-optimized (and pricier); storage grows automatically.
- Highly available cluster: **6 copies of data across 3 AZs**.
- Supports cross-Region replication.
- A **Reader Endpoint** load-balances connections across the read replicas.
- Automated backups included.
- Scale for read spikes by adding Aurora Replicas; you can assign each replica a failover priority.
- You can migrate a MySQL snapshot into an Aurora cluster.
- `AuroraReplicaLagMaximum` — max replication lag (seconds) between the writer and any Aurora replica; high lag can, e.g., cause a shopping cart to appear not to update correctly.
- **Aurora Serverless** auto-scales based on actual database load, including connection count and query volume.

### In-memory caches

- **ElastiCache** — **Redis** (high availability, persistent) or **Memcached** (no HA, multi-threaded); ideal for read-intensive workloads without heavy app code changes.
  1. Classic cache pattern: RDS behind ElastiCache, fall back to RDS on a cache miss.
  2. Session store pattern: keep the app stateless by storing session data in ElastiCache.
- **Redis replication:**
  - *Cluster mode disabled* — asynchronous, 1 primary + up to 5 replicas.
  - *Cluster mode enabled* — data is sharded (up to 5 replicas per shard), best for scaling writes, Multi-AZ, scales automatically; can be reconfigured online or offline (more configuration options offline).
- **Memcached** has no API to resize an existing cluster's node type in place — the actual approach is to call `CreateCacheCluster` to spin up a **new** cluster with the desired node type, since Memcached cannot be modified in place (there is no `ModifyCacheCluster` option for changing node types).

---

## 09. Monitoring, Auditing and Performance

- CloudWatch collects metrics for virtually every AWS service.
- EC2 default metric interval is 5 minutes (1 minute with paid detailed monitoring).
- **Custom metrics** (e.g. RAM usage) use the `PutMetricData` API and a custom namespace.
- **Dashboards** are global — they can include graphs pulled from different accounts and Regions.

**CloudWatch Logs:**

- A *log group* represents an application; a *log stream* represents an instance/source within that application.
- Logs can be exported to S3, but **not in real time**.
- **Subscriptions** deliver logs in real time to destinations like Kinesis Data Streams or Lambda.
- **CloudWatch Logs Insights** lets you query logs with a purpose-built query language and export the results.

**Alarms:**

- States: `OK`, `INSUFFICIENT_DATA`, `ALARM`.
- Can target EC2 actions, EC2 Auto Scaling, or SNS (native integration).
- **CloudWatch Anomaly Detection** uses ML to model an expected range of "normal" behavior based on historical metrics.

**EventBridge:**

- Runs scheduled scripts via Lambda.
- Example flow: S3 upload event → optional filter (e.g. an input transformer, or scope to a specific log group in the rule) → EventBridge rule matches the filtered event → JSON payload delivered to a Lambda target.
- A **resource-based policy** on the event bus lets other accounts put events onto it.

**Compliance auditing:** use **AWS CloudTrail** — the event history of API calls across your account.

---

## 10. AWS Account Management

**AWS Health Dashboard:**

- *Service Health Dashboard* — health of all Regions and all AWS services.
- *Your Account Health Dashboard* — performance/availability of the specific AWS services underlying **your** resources; you can set alerts, open cases, view scheduled maintenance activities, etc.
- Example automation: AWS Health event → triggers EventBridge → invokes Lambda/EC2/SNS/SQS/Kinesis Data Streams. E.g., "exposed access keys" event → invoke a Lambda that deletes the exposed IAM access keys.

**AWS Organizations:**

- Global service for managing multiple accounts: one management account plus member accounts, sharing a single payment method (volume discounts on EC2, S3, etc.) and shared Savings Plans.
- **Service Control Policies (SCPs)** are applied to an OU — never to the management account, which retains full power regardless.
- **Bucket-access pattern for "any account in the org, no outsiders":** use a bucket policy with `Principal: "*"`, constrained by the condition `aws:PrincipalOrgID` (or `PrincipalOrgId`) equal to your Organization ID.
  - Listing every account explicitly as principal doesn't scale and is error-prone.
  - Using the management account as principal wouldn't cover other member accounts.
  - `PrincipalOrgId` cannot itself be the *principal* — it must be used as a **condition key**.
  - This pattern lets any user in any member account access the bucket (subject to their own IAM permissions) while blocking outside access.
- **AWS Control Tower** sits on top of Organizations to set up a secure, compliant multi-account environment (e.g. deny certain Regions org-wide).
- **AWS Service Catalog** — a self-service portal for launching predefined, admin-approved products. Products are CloudFormation templates; a *portfolio* is a set of templates with IAM permissions granting access. Users pick from an authorized product list and launch a properly configured, tagged resource.

**Cost management:**

- **Billing Alarms** — first enable them account-wide, then create a CloudWatch alarm on the estimated-charges metric.
- **Cost Explorer** — usage across all accounts, monthly/hourly/resource-level granularity, forecasts up to 12 months based on historical usage.
- **AWS Cost and Usage Reports (CUR)** — the most comprehensive cost/usage dataset, published to an S3 bucket you own; can break down costs by hour/day/month, by product/resource, or by your own tags.
- **AWS Budgets** — e.g. create a monthly budget that notifies you, stops EC2, or triggers an SCP/IAM policy when exceeded.
- **Cost allocation tags** — `aws:` (AWS-generated) or user-defined tags, for detailed cost tracking.
- **AWS Compute Optimizer** — uses ML to recommend better-fit resource configurations to reduce cost.
- If a newly launched EC2 instance doesn't yet appear in an expected report/tool, wait at least ~30 hours before troubleshooting further.

---

## 11. Disaster Recovery

**AWS DataSync:**

- Synchronizes data to/from S3, EFS, or FSx.
- *On-prem → AWS*: over NFS, requires the AWS DataSync agent on-premises.
- *AWS → AWS*: can be scheduled; preserves permissions and metadata.
- **AWS Snowcone** — for edge computing and data transfer in remote, difficult, or limited-connectivity locations; a compact, portable, rugged device for non-datacenter environments (boats, vehicles, industrial sites).
- **Snowball Edge Storage Optimized** — the right choice for securely and quickly moving tens of terabytes to petabytes of data into AWS.

**AWS Backup:**

- Define a backup plan (frequency, retention policy, transition to cold storage, etc.) and assign AWS resources to it — backups land in S3.
- **Vault Lock** makes backups in a vault immutable/undeletable.

---

## 12. Security and Compliance

**Perimeter protection:**

- **AWS WAF** — Layer 7 firewall; deployable on ALB, API Gateway, or CloudFront (same Region as the resource). Supports up to 10,000 IP-based rules; inspects HTTP body/headers/URI strings to block common attacks like SQL injection.
  - WAF does **not support NLB directly** — if you need a fixed IP in front of a WAF-protected ALB, use **Global Accelerator**: `Users → Global Accelerator (fixed IP) → same-Region ALB (with WAF) → EC2 instances`.
- **AWS Shield** — protects against DDoS attacks (Layer 3/4).
- **AWS Firewall Manager** — centrally manages security policies (WAF, Shield, security groups, ENIs, etc.) across all accounts in an organization, at the **Region** level.

**Log analysis:** logs stored in S3 can be queried with **Athena** (move older logs to Glacier for cost savings).

**Detection & data-protection services:**

- **Amazon GuardDuty** — analyzes account/network/DNS logs (not CloudWatch) for threats and suspicious activity; can detect cryptocurrency-mining-related activity via ML. It detects threats/suspicious activity, **not** open ports or security-group misconfigurations.
- **Amazon Macie** — uses ML pattern matching to discover and protect **sensitive data stored in Amazon S3**. It does **not** monitor CloudWatch Logs directly.
  - To find PII/PHI *inside CloudWatch Logs*, use **CloudWatch Logs data protection policies** instead — they automatically detect sensitive-data patterns in log events and can trigger alerts or actions.
- **Amazon Inspector** — scans for vulnerabilities in EC2 workloads and container image repositories.
- **AWS Trusted Advisor** — includes a security check for "Security Groups – Specific Ports Unrestricted" (flags overly open security groups), plus service-limit checks (e.g. an ASG not launching instances during busy periods).
- **AWS Security Hub** — aggregates findings from other services (like Trusted Advisor); conformance packs provide compliance checks/reporting but do **not** enforce restrictions on their own.

> **EXAM TIP — common wrong-answer patterns:**
> - GuardDuty detects threats/suspicious behavior — not open ports or SG misconfiguration.
> - Service Control Policies (SCPs) enforce/restrict cross-account permissions in Organizations — they don't detect open ports.
> - IAM Access Analyzer identifies resource sharing with *external* entities — it's not for network-level exposure like open ports.

**AWS KMS:**

- AWS manages the underlying keys for you; integrated with IAM authorization; all KMS usage is auditable via CloudTrail; usable directly in API calls.
- AWS-managed keys automatically rotate every year.
- **Customer Master Key (CMK):** a logical key representation generated by KMS; a CMK never leaves KMS (or its Region).
  - *CMK with AWS-managed key material* — KMS generates internal key material (never exposed); automatic rotation is simple to enable, and happens yearly.
  - *CMK with imported key material* — you generate the key yourself outside KMS and import it; **automatic rotation is not supported**. There's no built-in 6-month or automatic rotation option — you must create a new CMK, import new key material, and update the alias yourself.
- **Cross-Region snapshot copy pattern:** `EBS (Region 1) → snapshot in S3 (encrypted with Key A) → copy snapshot to Region 2 (re-encrypted with Key B) → new EBS volume (Region 2)`.
- **AWS-managed keys (`SSE-KMS` default keys) cannot be modified for cross-account use** — you cannot add an external account's IAM role to an AWS-managed key's policy. Only a **customer-managed** KMS key can grant cross-account access: create your own CMK, add the external role to its key policy, then share the key's ARN.

**AWS Secrets Manager:**

- Supports automatic rotation of secrets (AWS provides a Lambda that can rotate an RDS DB password, for example); requires KMS; integrates with CloudFormation.
- `RotationSchedule` defaults to 90 days, or a custom interval.
- Preferred over alternatives that: require manual password creation, store secrets as plain CloudFormation parameters (not appropriate for sensitive data), or use SSM Parameter Store without built-in rotation (Parameter Store is cheaper, but to get rotation you'd have to build it yourself with EventBridge + Lambda updating both the parameter and the actual password).

**IAM & MFA:**

- **You cannot enable MFA on an IAM role** — only IAM *users* can have an MFA device.
- To require MFA for API calls: `aws sts get-session-token --serial-number <mfa-device> --token-code <mfa-code>`.

---

## 13. Networking — Route 53

- An **authoritative** zone means the customer can update its own records.
- You **cannot create a CNAME at the zone apex** (e.g. `example.com`).
- **Alias records are preferred over CNAME for AWS resources like load balancers because:**
  - Alias records (A/AAAA) can be used at the zone apex; CNAME cannot, per DNS standards.
  - "Evaluate Target Health" only applies to alias records pointing at AWS resources.

**On-prem failover example:** for an on-premises primary/secondary website pair, Route 53 should route to the primary if its health check returns 2xx/3xx, otherwise route to the secondary.

- ✅ Correct approach: create a standard **A record** for each server and associate each with a Route 53 **HTTP health check** (with the failover routing policy/set IDs already configured).
- For on-premises endpoints, use standard A records — not alias records — because alias targets require AWS resources, and "Evaluate Target Health" only applies to alias targets.

**Lowest-latency multi-Region routing with fast failover:**

- ✅ Use **AWS Global Accelerator**: create a standard accelerator, create an endpoint group per Region, add a listener, and associate the endpoint group with the listener. This routes users to the lowest-latency healthy Region and fails over faster than typical DNS TTL-based approaches.

---

## 14. VPC

- A **Network ACL (NACL)** allows/denies specific inbound/outbound traffic at the **subnet** level.

**VPC Flow Logs:**

- Record all traffic to/from the network interface(s) of an instance — you can see whether packets arrive and are accepted/rejected, or don't arrive at all.
- **You cannot modify an existing Flow Log's filter** — to see REJECTED traffic you must recreate the Flow Log with the "ALL" filter.
- Great for diagnosing: a security group blocking traffic, a NACL blocking traffic, or bad routing.

**Exam-friendly troubleshooting summary:**

- Problem: RDP connection timeout → suspect network/VPC issue.
- Right tool: **VPC Flow Logs** on the instance's ENI.
- Why not CloudWatch Logs: it doesn't capture packet-level network traffic.
- Why not EC2 Instance Connect: that's SSH-only, Linux-only.
- CloudWatch metrics for a **Site-to-Site VPN** connection show tunnel state/metrics — not individual packets.

**VPC basics:**

- You **cannot modify a VPC's IPv4 CIDR** (e.g. if it conflicts with an on-prem client VPN's range) — you must create a new VPC or add an additional CIDR block.
- An **Internet Gateway + route table** allows internet traffic; it's horizontally scaled, redundant, and highly available by default.
- An **egress-only internet gateway** is the IPv6 equivalent for outbound-only access: it allows outbound IPv6 traffic from your VPC to the internet while preventing the internet from initiating an inbound IPv6 connection.
- Instances can live in public or private subnets. Private-subnet instances need a **NAT instance** or **NAT Gateway** to reach the internet.
  - A NAT *instance* must sit in a public subnet, have an Elastic IP, and have **Source/Destination Check disabled**; it's not HA by default (you'd need extra security-group/failover configuration), and it can become outdated/unpatched.
  - **Prefer the NAT Gateway** — it's AZ-specific, managed by AWS, requires an Internet Gateway and an Elastic IP (ENI).
  - Best practice: at least one public and one private subnet (each with its own NAT Gateway) per AZ, in at least 2 AZs.

**DNS resolution attributes:**

- `enableDnsSupport` — enables DNS resolution in the VPC via the Route 53 Resolver (a specific reserved IP is queried).
- `enableDnsHostnames` — assigns public DNS hostnames to instances that have a public IP.
- If you use a custom domain in a **Private Hosted Zone** (e.g. `mycompany.private A 10.0.0.1`), **both attributes must be `true`** for private DNS resolution inside the VPC to work correctly.

**NACL vs. Security Group:**

- A **security group is stateful** — allowing inbound traffic automatically allows the corresponding outbound response.
- A **NACL is stateless** — you need symmetric inbound/outbound rules for traffic to actually pass in both directions (e.g. allow the ephemeral port range outbound on the client-side NACL and inbound on the server-side NACL, since return traffic from a web server uses ephemeral ports).
- **NACL rules are evaluated by number — lower numbers take precedence.**
- Security groups are **allow-only** — they cannot have explicit deny rules.
- A `REJECTED` entry in Flow Logs implies a NACL deny rule (SGs can't produce an explicit "rejected" — they simply don't allow).
- If there's no route to an internet gateway, there will be no corresponding entries in VPC Flow Logs for internet-bound traffic.
- A NACL **allows everything by default** when first created for a VPC (the default NACL); custom NACLs deny everything by default until you add rules.

**Direct Connect (DX):**

> **EXAM TIP:** DX = a dedicated **private** connection from a remote network to your VPC. Look for the word "dedicated"/"private connection" as the trigger phrase in exam questions.

- You need a **Virtual Private Gateway** on your VPC, with a **private virtual interface** for each VPC you want to connect.
- A single DX connection can access both public resources (like S3) and private resources (like EC2) — you need a **public virtual interface** in the gateway to reach S3.
- Increases bandwidth versus a VPN, but has **no encryption by default** — layer a VPN over DX if you need encryption.
- High resiliency for critical workloads: one connection per location. Maximum resiliency: two connections per location, each terminating on a different device.
- **Direct Connect Gateway** lets you connect multiple VPCs across different Regions to a single on-premises network through one DX connection.

**Site-to-Site VPN:**

- Enable **route propagation** for the Virtual Private Gateway in the route tables of the relevant subnets.
- To ping on-prem instances, allow the ICMP protocol inbound in the security group.

**Other VPC tools:**

- **VPC Managed Prefix List** — a shareable, named CIDR block collection (if customer-managed).
- **VPC Reachability Analyzer** — checks whether two configurations *can* communicate, without sending actual packets.
- **VPC Peering** — connects two VPCs over the AWS network. Requires **non-overlapping CIDRs**, is **not transitive**, and requires updating each side's route table. Works across accounts and Regions. If you need transitivity, use a **Transit Gateway** instead.
- **VPC Endpoints** — reach services like S3 and CloudWatch without traversing the public internet.
  - *Interface endpoints* — an ENI with a private IP attached to the service; requires a security group; better suited when accessed over a VPN.
  - *Gateway endpoints* — support S3 and DynamoDB only; free; no security group required — **generally the preferred option** for those two services.
- **AWS PrivateLink** — needs an NLB in your (the provider's) VPC and an ENI in the customer's VPC.

**VPC Flow Log interpretation table:**

| Inbound | Outbound | Likely blocker |
|---|---|---|
| REJECT | — | NACL or Security Group |
| ACCEPT | REJECT | NACL |
| — | REJECT | NACL or Security Group |
| REJECT | ACCEPT | NACL |

- **VPC Traffic Mirroring** — captures traffic from a source and sends it to a target within your VPC (for deep packet inspection tooling).
- **AWS Network Firewall** — inspects traffic in any direction, Layers 3–7, across the whole VPC.

**Subnet sizing:** a `/27` is often too small in practice — it provides 32 total addresses, 5 of which AWS reserves (network, broadcast, and AWS-reserved addresses), leaving only **27 usable IPs** for EC2 instances.

---

## 15. Other Services

**SQS — migrating standard to FIFO (a common scenario question):**

- Standard queues don't guarantee ordering or exactly-once processing — you **cannot convert** a standard queue into a FIFO queue; you must create a **new** FIFO queue.
- ✅ Correct approach: create a new SQS FIFO queue, turn on **content-based deduplication**, and update the application to include a **Message Group ID** on every message (required for in-group ordering).
- Content-based deduplication treats identical-content messages as duplicates within a 5-minute deduplication window.
- `DelaySeconds` is unrelated to this migration — it's optional and orthogonal to FIFO configuration.

**SNS:** every subscriber receives all published messages, but you can apply **filter policies** to scope what each subscriber gets. Supports cross-Region notifications.

**ECS:**

- **Fargate + EFS** = serverless, Multi-AZ, and the easiest combination to auto-scale.
- ECS also supports an **EC2 launch mode** (you manage the underlying instances).
- **Task `networkMode`** options:
  - `bridge` — uses a Docker-managed virtual bridge.
  - `host` — bypasses the Docker network stack, using the host's network directly.
  - `awsvpc` — gives each task its own ENI and IP address, behaving like a mini EC2 instance; **required for Fargate tasks**.
- **ALB does not send data to X-Ray** (you'd need to instrument the application/service itself for tracing).

**AWS Elastic Beanstalk deployment types:**

- **Rolling deployments** — splits environment instances into batches, deploying the new version to one batch at a time while the rest keep serving the old version. A *rolling deployment with an additional batch* launches a new batch first (before taking any instance out of service) to maintain full capacity during the deployment; the extra batch is terminated once the deployment completes.
- **Immutable deployments** — launch a full set of new instances (new ASG) running the new version alongside the old ones. This avoids issues from partially completed rolling deployments — if the new instances fail health checks, Beanstalk terminates them and leaves the original instances untouched.
- To make Beanstalk automatically replace unhealthy instances, change the underlying Auto Scaling group's health check type from **EC2** to **ELB**, via a Beanstalk configuration file.

**AWS Config remediation pattern (review all the exam's Config-rule scenarios):**

- Example: implement an AWS Config rule that flags security groups permitting SSH from `0.0.0.0/0`.
- ✅ Pair it with an **SSM Automation document** that updates the flagged security-group rules — Config detects, SSM Automation remediates.

**Route 53 + SES email deliverability:**

- Scenario: emails sent via Amazon SES from a domain hosted in Route 53 are being rejected by some recipient mail servers.
- ✅ Fix: add a **TXT record with SPF (Sender Policy Framework) information**. SPF tells receiving mail servers which IP addresses are authorized to send mail on behalf of the domain, preventing spoofing and improving deliverability.
