# Domain 3: Deployment, Provisioning, and Automation — Full Reference

Domain 3 is done at deep-dive level below across both tasks (3.1.1–3.1.6, 3.2.1–3.2.2). Distractor callouts and key numbers are marked separately at the end.

---

## Task 3.1: Provision and Maintain Cloud Resources

### Skill 3.1.1 — AMIs & Container Images (EC2 Image Builder)

| Concept | Detail |
|---|---|
| AMI creation | `CreateImage` API reboots the instance by default to guarantee filesystem consistency; `--no-reboot` skips this but risks an inconsistent image |
| EBS-backed vs instance store–backed | EBS-backed: can stop/start, boots faster, supports `CreateImage` directly. Instance store–backed: root volume is ephemeral, must be *bundled* (`ec2-bundle-vol`) and uploaded to S3, can only be terminated (never stopped) |
| AMI copy | Cross-Region and cross-account copy both supported; copy is asynchronous (returns immediately, new AMI ID, status `pending`); can add/change encryption during copy (unencrypted → encrypted, or re-encrypt with different CMK) |
| AMI sharing | `ModifyImageAttribute` (launch permissions) by account ID; if AMI is encrypted, the KMS key policy must also grant the target account `kms:CreateGrant`, `kms:Decrypt`, `kms:DescribeKey` — sharing the AMI alone is not sufficient |
| EC2 Image Builder — Recipe | Base image (AMI ID or SSM parameter for "latest") + ordered list of components |
| EC2 Image Builder — Components | Build phase + Test phase, defined as YAML/JSON documents; can reference public Amazon-managed components or custom ones |
| EC2 Image Builder — Image pipeline | Ties recipe + infrastructure config + distribution config together; scheduled via cron expression or run manually |
| EC2 Image Builder — Infrastructure configuration | Instance type(s), IAM instance profile, SNS topic for pipeline notifications, "terminate instance on failure" toggle (default true — turn off to debug a failed build) |
| EC2 Image Builder — Distribution configuration | Target Regions/accounts, AMI launch permissions, license configuration, container image target (ECR repo + tags) |
| Image Builder — Test phase | Launches a fresh test instance from the just-built AMI, runs test components; pipeline fails if tests fail (image is not distributed) |
| Image lifecycle policies | Native to Image Builder (separate from ECR lifecycle policies); Deprecate → Disable → Delete stages based on **age** or **count** of images/AMIs produced |
| ECR — image scanning | *Basic scanning* (on push or manual, uses Clair-based CVE DB) vs *Enhanced scanning* (continuous, powered by Amazon Inspector, also scans for OS + language package vulnerabilities) |
| ECR — lifecycle policies | Rule-based: expire untagged images after N days, or keep only N images matching a `tagPrefixList` |
| ECR — image tag immutability | Per-repository setting; prevents overwriting a tag (e.g., `latest` can still be mutable if excluded) |
| ECR — cross-account/cross-region | Repository policy for cross-account pull/push; **Replication configuration** for automatic cross-Region/cross-account copies |

**Distractor pair:** EC2 Image Builder pipeline vs. a manual `CreateImage` + custom script — "least operational overhead" / "standardize golden AMI process across teams" → Image Builder. Manual scripting is the wrong-answer trap.
**Distractor pair:** ECR basic scanning (on-push, CVE only) vs. enhanced scanning (continuous, Inspector-powered, OS + package-level) — exam likes to test "which finds a *newly disclosed* CVE in an image you haven't rebuilt" → enhanced scanning (continuous rescans).

---

### Skill 3.1.2 — CloudFormation & CDK

| Concept | Detail |
|---|---|
| Stack states | `CREATE_IN_PROGRESS`, `CREATE_COMPLETE`, `CREATE_FAILED`, `ROLLBACK_IN_PROGRESS`, `ROLLBACK_COMPLETE`, `UPDATE_ROLLBACK_COMPLETE`, `DELETE_FAILED` |
| **`ROLLBACK_COMPLETE` trap** | A stack in this state **cannot be updated** — it must be deleted and recreated. This is a very common exam distractor: "how do you fix a stack stuck in ROLLBACK_COMPLETE?" → delete and redeploy, not "retry update" |
| Change sets | Preview of what an update would do before executing; doesn't modify resources until executed; useful for validating no unintended replacement/deletion |
| Stack policies | JSON document that protects specific resources from **updates** (not creation/deletion of the stack itself); default behavior with no policy = all updates allowed; once *any* policy is set, everything not explicitly `Allow`ed is denied |
| Nested stacks | `AWS::CloudFormation::Stack` resource — child stack is fully owned/managed by parent, updates cascade, useful for reusable templates |
| Cross-stack references | Parent exports a value with `Outputs` + `Export`, other stacks import with `Fn::ImportValue`; an exported value **cannot be deleted or changed** while another stack imports it |
| Nested stacks vs cross-stack refs | Nested = tight coupling, single unit of lifecycle. Cross-stack = loose coupling, independent lifecycles, but import lock-in on changes |
| Drift detection | Detects resources that were changed **outside** CloudFormation (manual console/CLI edits); does not auto-remediate, only reports |
| `DeletionPolicy` | `Delete` (default), `Retain` (resource survives stack deletion), `Snapshot` (RDS, EBS, ElastiCache, Neptune, Redshift only — snapshot taken before deletion) |
| `UpdateReplacePolicy` | Same values as above but governs what happens when an update forces **replacement** of a resource |
| `CreationPolicy` + `cfn-signal` | Used with `AWS::CloudFormation::Init` / EC2 or Auto Scaling Group — stack creation waits for a success signal from the instance before marking `CREATE_COMPLETE`; prevents premature "success" when app bootstrap hasn't finished |
| Rollback behavior | Default: automatic rollback on failure. `--disable-rollback` (CLI) or "Preserve successfully provisioned resources" (console) leaves resources in place for debugging |
| Rollback triggers | CloudWatch alarms attached to a stack update — if alarm goes into ALARM state during the monitoring period, CFN rolls the update back automatically |
| Capabilities | `CAPABILITY_IAM` (template creates IAM resources with auto-generated names), `CAPABILITY_NAMED_IAM` (template creates IAM resources with **custom** names), `CAPABILITY_AUTO_EXPAND` (template uses macros/nested transforms like SAM) |
| CDK — Constructs | L1 (Cfn-prefixed, 1:1 with CFN resources), L2 (curated, sensible defaults, e.g. `s3.Bucket`), L3/patterns (multi-resource architectures, e.g. `ApplicationLoadBalancedFargateService`) |
| CDK — Bootstrap | `cdk bootstrap` provisions the CDK Toolkit stack (S3 staging bucket, ECR repo, IAM roles) — required **once per account/Region** before first deploy |
| CDK — Synth & deploy | `cdk synth` renders CloudFormation template from code (doesn't touch AWS); `cdk deploy` synths then executes a CloudFormation deployment (change set under the hood) |
| StackSets vs. Stacks | Single stack = one account/Region. StackSet = same template deployed/updated across **multiple accounts and Regions** from one admin operation |

**Distractor pair:** Stack stuck `ROLLBACK_COMPLETE` → delete & redeploy (not "update" or "continue rollback").
**Distractor pair:** `DeletionPolicy: Retain` vs `Snapshot` — Snapshot only applies to a specific subset of stateful resources (RDS, EBS, ElastiCache, Neptune, Redshift); for everything else (e.g., S3 bucket), only `Retain` or `Delete` are valid — S3 buckets must also be **empty** to delete via CFN regardless of policy.

---

### Skill 3.1.3 — Identify & Remediate Deployment Issues

| Issue Type | Root Cause / Fix |
|---|---|
| Subnet sizing | VPC/subnet CIDR exhausted mid-deployment (Auto Scaling can't launch, or CFN fails). Remember AWS **reserves 5 IPs per subnet** (network addr, VPC router, DNS, future use, broadcast) — undersized subnets fail earlier than expected. Fix: add a secondary CIDR block to the VPC and new subnets, or resize before deployment (can't shrink an existing subnet) |
| CloudFormation `ROLLBACK_COMPLETE`/`DELETE_FAILED` | Stuck stack → delete and recreate; `DELETE_FAILED` often caused by a resource with `DeletionPolicy: Retain` that CFN still tries to interact with, or a non-empty S3 bucket/ECR repo blocking deletion |
| CFN permissions issues | Missing `CAPABILITY_IAM`/`CAPABILITY_NAMED_IAM` in the deploy call; or the CFN execution role itself lacks permissions for the resource being created — check the **stack event** for the exact `AccessDenied` action |
| CreationPolicy timeout | `cfn-signal` never received (app bootstrap failed, or `Timeout` value too short) → stack creation fails after timeout even though resources exist |
| Circular dependency | Two resources reference each other's attributes — CFN can't determine creation order; fix by decoupling with `DependsOn` restructuring or separating into nested stacks |
| Resource limit/quota errors | e.g. EIP limit, VPC limit per Region — surfaces as `CREATE_FAILED` with a quota message in stack events |

**Exam signal:** "deployment fails intermittently only during high-traffic scaling events" → almost always subnet IP exhaustion, not an AMI or IAM issue.

---

### Skill 3.1.4 — Multi-Region / Multi-Account Provisioning

| Concept | Detail |
|---|---|
| AWS RAM | Shares resources you own with other accounts/OUs without duplicating them. Shareable: subnets, Transit Gateways, License Manager configurations, Route 53 Resolver rules, RDS DB Clusters (Aurora), License Manager, Outposts, and more. Shared subnets: participant accounts can launch resources into them but **cannot modify the subnet itself** (route tables, NACLs stay owned by the sharer) |
| CloudFormation StackSets | Deploys/updates/deletes stacks across many accounts & Regions from one operation |
| StackSets — permission models | **Self-managed**: you manually create `AWSCloudFormationStackSetAdministrationRole` (admin account) and `AWSCloudFormationStackSetExecutionRole` (target accounts) with trust relationships. **Service-managed**: requires AWS Organizations, auto-deploys to accounts as they join an OU, no manual role setup |
| StackSets — deployment controls | `MaxConcurrentCount`/`MaxConcurrentPercentage` (parallelism), `FailureToleranceCount`/`FailureTolerancePercentage` (how many account/Region failures before StackSet operation stops), region deployment order is sequential per account, concurrent across accounts (or reversed depending on settings) |
| StackSets — service-managed auto-deployment | New accounts added to a target OU automatically get stack instances created; removing an account from OU auto-deletes its stack instance (configurable) |

**Distractor pair:** AWS RAM (share existing resource, e.g. subnet or TGW attachment) vs. StackSets (deploy/replicate a **template** that creates new resources in each account) — RAM shares one resource; StackSets creates independent resource copies everywhere.

---

### Skill 3.1.5 — Deployment Strategies and Services

| Concept | Detail |
|---|---|
| CodeDeploy — EC2/on-prem: In-place | Deploys directly onto existing instances (stopped from traffic via ELB deregistration during deploy); AllAtOnce, HalfAtATime, OneAtATime, or custom |
| CodeDeploy — EC2/on-prem: Blue/Green | Provisions a **new** Auto Scaling Group/fleet, deploys, then reroutes traffic (via ELB) and terminates originals (per configurable termination wait time) |
| CodeDeploy — Lambda deployment configs | `AllAtOnce`, `Canary10Percent5Minutes` / `Canary10Percent10Minutes` / `Canary10Percent15Minutes` / `Canary10Percent30Minutes`, `Linear10PercentEvery1Minute` / `Linear10PercentEvery2Minutes`/`3Minutes`/`10Minutes` — traffic shift is via Lambda **alias** weighted routing |
| CodeDeploy — ECS deployment configs | Blue/Green only via CodeDeploy integration (not in-place); similar Canary/Linear percentage options; uses two target groups (blue = current, green = new) behind the same ALB, with a **test listener** available for pre-shift validation |
| CodeDeploy — rollback | Automatic rollback on deployment failure **or** on CloudWatch alarm trigger (must specify alarm(s) in the deployment group) — this is the canonical cross-domain pattern (CloudWatch alarm → CodeDeploy rollback) |
| AppSpec file | Defines deployment lifecycle hooks (`BeforeInstall`, `AfterInstall`, `ApplicationStart`, `ValidateService` for EC2/on-prem; `BeforeAllowTraffic`/`AfterAllowTraffic` for Lambda/ECS) |
| AWS AppConfig | Deploys **configuration/feature-flag changes** (not code) with the same deployment-strategy concept (linear/canary rollout) + automatic rollback via CloudWatch alarms; decouples config changes from code deployments |
| Elastic Beanstalk deployment policies | All at once, Rolling, Rolling with additional batch, Immutable, Traffic splitting (canary) — EB-native equivalent, distinct from CodeDeploy |

**Distractor pair:** ECS deployment types — native **rolling update** (ECS service scheduler, no CodeDeploy needed, minimum/maximum healthy percent controls) vs. **blue/green via CodeDeploy** (needed for canary/linear traffic shifting + automatic alarm-based rollback). If the question mentions "gradually shift traffic" or "automatic rollback on alarm" for ECS → CodeDeploy blue/green, not native rolling update.
**Distractor pair:** AppConfig vs Parameter Store for feature flags — Parameter Store just stores a value; AppConfig adds **validated, gradual, monitored rollout with rollback** — "least risk" config change signals AppConfig.

---

### Skill 3.1.6 — Third-Party Tools (Terraform, Git)

| Concept | Detail |
|---|---|
| Terraform state | `.tfstate` tracks real-world resource mapping; for team use, store remotely (S3 + DynamoDB for state locking) rather than locally, to avoid conflicting concurrent applies |
| Terraform vs CloudFormation | Terraform is multi-cloud, HCL syntax, explicit `plan`/`apply` workflow; CFN is AWS-native, no separate state file (AWS manages state), tighter integration with StackSets/Service Catalog |
| CodeConnections (formerly CodeStar Connections) | Used to integrate CodePipeline with third-party Git providers (GitHub, GitHub Enterprise, Bitbucket, GitLab) via OAuth-based connection ARN, rather than storing credentials directly |
| CodeCommit | AWS-native Git repo service — note: **as of July 2024, CodeCommit is no longer available to new customers**; existing customers can continue use. Exam may still reference it, but new architectures typically point to GitHub/GitLab via CodeConnections |
| CodePipeline source stage | Can trigger on Git push (via CodeConnections webhook) or on S3 object upload (versioned bucket) |

---

## Task 3.2: Automate the Management of Existing Resources

### Skill 3.2.1 — Systems Manager Automation

| Concept | Detail |
|---|---|
| SSM Automation documents (runbooks) | JSON/YAML documents defining a sequence of steps (`aws:runCommand`, `aws:invokeLambdaFunction`, `aws:executeAwsApi`, `aws:branch`, `aws:approve`, etc.); can be AWS-predefined (`AWS-*`) or custom |
| Automation execution | Can run standalone (`StartAutomationExecution`), be invoked from EventBridge rules, CloudWatch alarms (via EventBridge), or Systems Manager Maintenance Windows |
| Rate/error controls | `MaxConcurrency` and `MaxErrors` govern how automation runs across multiple targets simultaneously |
| Approve/reject steps | `aws:approve` step pauses execution pending manual (or SNS-notified) approval — useful for production remediation requiring human sign-off |
| State Manager | Enforces a **desired, recurring state** on managed instances via association (e.g., ensure an agent is installed, run a document on a schedule) — different from Automation which is typically a one-shot or event-triggered remediation |
| Maintenance Windows | Defines a recurring time window + targets + tasks (Run Command, Automation, Lambda, Step Functions) — used to control **when** disruptive operations (patching, reboots) happen |
| Patch Manager | Patch baselines (approval rules, auto-approval delay, rejected patches list), patch groups (tag-based), compliance reporting; predefined baselines per OS or custom |
| Run Command | Ad hoc or scripted execution of commands on managed instances without SSH/RDP; requires SSM Agent + IAM instance profile with SSM permissions |
| Session Manager | Interactive shell access without opening inbound ports (no bastion, no SSH key) — requires SSM Agent, instance profile with `AmazonSSMManagedInstanceCore` (or equivalent), and outbound HTTPS (443) connectivity (via NAT/IGW or SSM VPC endpoints) |
| **SSM four-layer troubleshooting chain** | (1) SSM Agent installed & running, (2) IAM instance profile attached with correct SSM permissions, (3) network path to SSM endpoints (NAT gateway/IGW, or VPC endpoints for `ssm`, `ssmmessages`, `ec2messages`), (4) Security Group/NACL not blocking outbound 443 — this exact chain shows up across Domains 1, 3, and 5 |
| Parameter Store | Standard (free, 4 KB max, no auto-rotation) vs Advanced (paid, 8 KB max, parameter policies for expiration/notification) tiers; supports SecureString (KMS-encrypted) |
| Parameter Store vs Secrets Manager | Secrets Manager: **automatic rotation** (native Lambda rotation functions for RDS/Redshift/DocumentDB, or custom), built-in versioning with staging labels (`AWSCURRENT`/`AWSPENDING`), higher cost per secret/API call. Parameter Store: cheaper, good for config values and secrets **without** built-in rotation need (rotation must be self-built via Lambda + EventBridge schedule) |
| Automation document actions | Common exam actions: stop/start/reboot instances, create AMI/snapshot before patching, attach/detach EBS volumes, update Auto Scaling Group capacity, remediate Config non-compliant resources |

**Distractor pair:** Config + SSM Automation ("detect AND remediate") vs. GuardDuty/Inspector alone ("detect only") — a finding from GuardDuty needs to be **routed** (typically via EventBridge) to an SSM Automation document or Lambda to actually fix anything; GuardDuty itself takes no remediation action.
**Distractor pair:** State Manager (recurring desired-state enforcement) vs. Automation runbook (triggered task/remediation, often one-time or event-driven) — "ensure this agent config is always present" → State Manager association; "when this alarm fires, do X" → Automation triggered via EventBridge.

---

### Skill 3.2.2 — Event-Driven Automation

| Concept | Detail |
|---|---|
| S3 Event Notifications (native) | Configured directly on the bucket; destinations: SNS topic, SQS queue, Lambda function; simpler but limited filtering (prefix/suffix only), fewer destinations |
| EventBridge (via S3 → EventBridge integration) | Must be explicitly enabled on the bucket ("Amazon EventBridge" setting); once enabled, **all** S3 events flow to the default event bus, enabling advanced content-based filtering, multiple/fan-out targets (Step Functions, Kinesis, API destinations, cross-account buses, etc.), and rule-based routing |
| S3 native notifications vs EventBridge | Native = simpler, lower latency, fewer targets/filter options. EventBridge = richer filtering (event pattern matching on any field), many more target types, easier fan-out to multiple consumers, cross-account event bus routing — "route the same event to multiple different services" or "filter on object metadata/tags" → EventBridge |
| Lambda event source mappings | For **poll-based** sources (SQS, Kinesis, DynamoDB Streams) Lambda polls the source using an event source mapping — batch size, batch window, and (for streams) parallelization factor are tunable; for **push-based** sources (S3, SNS, API Gateway, EventBridge) the source invokes Lambda directly, no polling config needed |
| EventBridge rules — general (recap, also in Domain 1) | Event pattern matching (JSON) vs. schedule expressions (`rate()`/`cron()`); can target Lambda, SSM Automation, Step Functions, SNS/SQS, Kinesis, ECS RunTask, CodePipeline, and cross-account/cross-Region buses |
| Dead-letter queues / retry | Both S3-native (SNS/SQS DLQ semantics apply to those services) and EventBridge (rule-level DLQ config) support capturing failed event deliveries after retries are exhausted |

**Distractor pair:** "Trigger a workflow only when an object with a specific tag/metadata is uploaded, and notify two independent teams' systems" → this needs **EventBridge** (content filtering + fan-out); plain S3 event notifications can't filter on tags and support only limited destinations per configuration.

---

## Domain 3 — Key Numbers Recap

| Item | Value |
|---|---|
| Reserved IPs per subnet | 5 (network, VPC router, DNS, future use, broadcast) |
| CodeDeploy Lambda canary options | 10% traffic shift, then wait 5/10/15/30 min before completing |
| CodeDeploy Lambda linear options | 10% every 1/2/3/10 min |
| Parameter Store Standard max size | 4 KB |
| Parameter Store Advanced max size | 8 KB |
| Secrets Manager staging labels | `AWSCURRENT`, `AWSPENDING` (and `AWSPREVIOUS` after rotation) |
| CodeCommit new-customer availability | Discontinued for new customers as of July 2024 |
| CFN stuck state requiring delete+recreate | `ROLLBACK_COMPLETE` |

---

Domain 3 is now fully re-consolidated. Since Domain 4 (beyond IAM) is still open and Domain 5 only has survey-level coverage, want to move to closing out the remaining Domain 4 skills next, or start the deep Domain 5 pass?