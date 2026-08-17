Picking up Domain 4 where we left off — skipping 4.1.1 (IAM) since that's already locked in, covering **4.1.2 through 4.2.5** at full technical depth.

# Domain 4: Security & Compliance (16%)

## Task 4.1 (continued)

### Skill 4.1.2 — Troubleshoot and audit access issues (CloudTrail, IAM Access Analyzer, IAM Policy Simulator)

| Tool | What it does | Key mechanics |
|---|---|---|
| **CloudTrail** | Records API activity (who did what, when, from where) | Management events (control-plane, e.g. `CreateBucket`) logged by default; Data events (object-level S3, Lambda invoke, DynamoDB item ops) NOT logged by default — must opt in, cost per event; Insights events detect unusual API call volume/rate |
| **Event history** | Last **90 days** of management events, viewable in console/CLI with **no trail required** | Read-only, can't be exported directly to S3 — need a Trail for that |
| **Trail** | Persists events beyond 90 days to S3, optionally to CloudWatch Logs | Single-Region vs multi-Region (multi-Region recommended, captures events from all Regions into one S3 location); Organization trail — created in management account, auto-applies to all member accounts, member accounts can't disable/delete it |
| **Log file integrity validation** | Detects tampering | SHA-256 hashing + digest files delivered to a separate S3 prefix hourly |
| **IAM Access Analyzer** | Finds resources shared with entities **outside** your account/org | Uses automated reasoning (Zelkova/provable security), not just pattern matching. Analyzes resource-based policies on: S3, IAM roles, KMS keys, Lambda, SQS, Secrets Manager, EFS, ECR repos, RDS snapshots, SNS topics, S3 access points |
| Access Analyzer — **policy generation** | Generates least-privilege IAM policy from actual CloudTrail activity | Looks back up to **90 days** of access history |
| Access Analyzer — **policy validation** | Checks syntax + security best practices before you save a policy | Flags errors, security warnings, suggestions |
| Access Analyzer — **custom policy checks** | Validates a policy change doesn't grant new/unintended access | Compares against a reference policy using natural language |
| Access Analyzer — **unused access findings** | Flags unused IAM roles, unused permissions, unused access keys, unused console passwords | Separate analyzer type from external-access analyzer |
| **IAM Policy Simulator** | Tests whether a policy *would* allow/deny an action | **Does NOT make a real API call.** Evaluates identity-based policies, resource-based policies, and permissions boundaries you select |

🚩 **Distractor trap:** Policy Simulator does **not** reflect real-time context — no SCP evaluation reliably, no MFA status, no session tags at call time. If a question involves "will this actually work given SCPs," the exam wants Access Analyzer or an actual test call, not the Simulator.

---

### Skill 4.1.3 — Multi-account strategies

| Concept | Detail |
|---|---|
| **AWS Organizations** | Management account + member accounts, grouped into OUs (Organizational Units, nestable) |
| **SCPs (Service Control Policies)** | **Do not grant permissions** — set the *maximum available* permissions (a filter/ceiling). Attached identity/resource policy is still required to actually grant access. Applied at OU or account level. **Never affects the management account.** Default: `FullAWSAccess` SCP attached everywhere. |
| **RCPs (Resource Control Policies)** | Newer sibling to SCPs — restrict maximum permissions **granted by resource-based policies** across accounts (e.g., stop an S3 bucket policy from granting external access even if the bucket owner tries). Doesn't affect identity-based policies. |
| **Explicit deny** | Always wins across SCP + identity policy + resource policy + permissions boundary evaluation |
| **AWS Control Tower** | Automated landing zone. **Preventive guardrails** = SCPs (block actions). **Detective guardrails** = Config rules (flag non-compliance, don't block). Account Factory automates new-account provisioning with baseline config. |
| **AWS RAM** | Share resources cross-account within an Organization *without* needing resource policies on each resource — e.g., subnets, Transit Gateway attachments, License Manager configs, Route 53 Resolver rules. Shares within an Organization don't require manual invitation acceptance. |
| **Cross-account roles** | Trust policy (who can `sts:AssumeRole`) + permission policy (what they can do once assumed). **External ID** required for third-party cross-account access to prevent the confused deputy problem. |
| **IAM Identity Center** | Successor to AWS SSO. Centralized human access across accounts via **permission sets** (templated IAM roles pushed to each account) |
| **Delegated administrator** | Lets a chosen member account administer an org-wide service (GuardDuty, Security Hub, Config, Macie, IAM Access Analyzer) without needing management-account credentials |

🚩 **Distractor trap:** "User has full IAM permissions in the account but the action still fails" → check for an SCP deny at the OU/account level, not the IAM policy. This is one of the single most common SOA-C03 scenario patterns.

---

### Skill 4.1.4 — Trusted Advisor security checks

| Category | Examples of Security checks |
|---|---|
| Security | Root account MFA, IAM use (at least one IAM user exists besides root), S3 bucket permissions (public read/write), Security Groups — unrestricted ports (22, 3389, 0.0.0.0/0), EBS public snapshots, RDS public snapshots, ELB listener security (weak SSL/TLS ciphers), CloudTrail logging enabled, KMS CMK key rotation status, exposed access keys (scans public repos) |

| Support plan | What you get |
|---|---|
| Basic / Developer | Only **Service Quotas** checks + a small free set of core security checks (S3 bucket permissions, security groups — specific ports, IAM use, MFA on root, EBS/RDS public snapshots — the "7 core checks") |
| Business / Enterprise | **Full check catalog** (100+ checks across all 6 categories) + Trusted Advisor **API** access + programmatic refresh |

- Trusted Advisor integrates with **EventBridge** — trigger notifications/automation when a check status changes (e.g., new red/yellow finding).
- Trusted Advisor is **detect + advise**, not automatic remediation — pair with Lambda/SSM Automation via EventBridge for auto-remediation.

---

### Skill 4.1.5 — Enforce compliance requirements

| Mechanism | Use case |
|---|---|
| **SCP — Region restriction** | Deny actions unless `aws:RequestedRegion` is in an approved list |
| **SCP — service restriction** | Deny access to non-approved services org-wide (e.g., block EC2 classic, block specific unapproved services) |
| **Config Conformance Packs** | Bundle of Config rules + optional auto-remediation actions, deployed as a single unit via CloudFormation/YAML template; deployable org-wide via Config's organization conformance pack feature |
| **Config Aggregators** | Single pane of glass for compliance status across multiple accounts/Regions |
| **Config custom policy rules (Guard)** | Write rules in the **AWS CloudFormation Guard** DSL — no Lambda function needed (contrast with older custom Lambda-backed Config rules) |
| **AWS Artifact** | Self-service access to AWS compliance reports (SOC 1/2/3, PCI DSS, ISO) and to accept AWS agreements (BAA for HIPAA, etc.) |
| **Tag Policies (Organizations)** | Enforce consistent tag keys/values org-wide; non-compliant tags are flagged, not blocked, by default |
| **Backup Policies (Organizations)** | Enforce AWS Backup plans centrally across accounts |

---

## Task 4.2: Protect data and infrastructure

### Skill 4.2.1 — Data classification scheme

| Tool | Detail |
|---|---|
| **Amazon Macie** | ML + managed data identifiers to discover/classify sensitive data in **S3 only** (PII, credentials, financial data). Produces **sensitive data findings** (what was found) and **policy findings** (e.g., bucket became public, bucket unencrypted). Integrates with Security Hub and EventBridge. Requires S3 as the source — does not scan RDS, DynamoDB, EBS, etc. |
| **Tag-based classification** | Manual/organizational scheme (Confidential/Internal/Public tags) — Macie output can drive automated tagging via Lambda |

🚩 **Distractor trap:** Macie is S3-specific. If a question describes sensitive data in RDS or DynamoDB, Macie is not the (direct) answer.

---

### Skill 4.2.2 — Encryption at rest (KMS)

| Concept | Detail |
|---|---|
| **Key types** | AWS managed keys (`aws/service`, free, rotated automatically every year, no control over policy); **Customer managed keys (CMK)** — full control of key policy, rotation, deletion; AWS owned keys — invisible, not in your account, used internally by some services |
| **Envelope encryption** | KMS `GenerateDataKey` returns a plaintext + encrypted data key; plaintext data key encrypts your data locally (fast, no size limit like direct KMS encrypt which caps at 4KB); encrypted data key stored alongside ciphertext |
| **Key policy** | Resource-based policy attached to the CMK itself — **required**, default policy grants root account full access and enables IAM policies to also govern access (both must allow) |
| **Grants** | Temporary, programmatic delegation of key usage — how AWS services get permission to use your CMK on your behalf |
| **Multi-Region keys** | Same key material replicated as independent keys in multiple Regions (not literally global) — used for DynamoDB Global Tables, cross-Region encrypted snapshot copy, cross-Region disaster recovery without re-encrypting |
| **Key rotation** | Automatic annual rotation is **optional** for CMKs (must be enabled); rotating **does not** re-encrypt existing ciphertext — AWS keeps old key material versions to decrypt old data. On-demand rotation also available. |
| **Imported key material (BYOK)** | Key material origin = `EXTERNAL`; you control expiration/deletion of material; if you delete the imported material, all data encrypted under that CMK becomes unrecoverable |
| **CloudHSM vs KMS** | CloudHSM = dedicated, single-tenant HSM cluster, **FIPS 140-2 Level 3**, full control of key material, used to back a **KMS Custom Key Store**. KMS (standard) = shared multi-tenant HSMs, FIPS 140-2 Level 2 for the general service (Level 3-validated HSMs underneath, but no direct key material access) |
| **S3 encryption options** | SSE-S3 (AES-256, AWS managed, no API call limits), SSE-KMS (`aws/s3` or CMK — **subject to KMS request throttling quotas**, can bottleneck high-throughput workloads), SSE-C (you supply the key per-request, AWS never stores it), DSSE-KMS (dual-layer, for compliance requiring two independent encryption layers) |
| **EBS encryption** | "Encrypted by default" is a **per-Region account setting**. Snapshots of an encrypted volume are always encrypted. **Cannot** enable encryption on an existing unencrypted volume in place — must snapshot → copy snapshot with encryption enabled → create new volume |
| **RDS encryption** | Must be enabled **at creation time**. Cannot enable on an existing unencrypted instance directly — snapshot → copy snapshot with encryption enabled → restore new instance from encrypted snapshot |

---

### Skill 4.2.3 — Encryption in transit (ACM)

| Concept | Detail |
|---|---|
| **ACM public certs** | Free, for use with **integrated AWS services only** (CloudFront, ALB/NLB, API Gateway, etc.) — private key **never leaves ACM**, cert cannot be downloaded/exported |
| **Validation** | DNS validation (CNAME record — recommended, supports **automatic renewal**) vs Email validation (manual approval, renewal can lapse if nobody responds) |
| **Renewal** | Auto-renews ~60 days before expiry **only if** DNS validation record still resolves and the cert is in active use by an integrated service |
| **Region requirement** | Cert for **CloudFront** must be requested in **us-east-1**, regardless of where your origin or viewers are (CloudFront is global but ACM integration is us-east-1 only). Cert for **ALB/NLB** must be in the **same Region** as the load balancer. |
| **ACM Private CA** | For internal/private PKI — issues certs for internal services, mutual TLS, IoT devices. Unlike public ACM certs, **these CAN be exported** for use outside AWS-integrated services. Paid per-CA-per-month plus per-cert issuance. |

---

### Skill 4.2.4 — Securely store secrets

**The #1 distractor pair in this domain:**

| Feature | Parameter Store | Secrets Manager |
|---|---|---|
| Cost | Standard tier: free. Advanced tier: paid per parameter/month | Paid per secret/month + API call charges |
| Size limit | Standard: 4 KB. Advanced: 8 KB | 64 KB (65,536 bytes) |
| **Automatic rotation** | **No native rotation** — must build your own via Lambda + EventBridge schedule | **Built-in rotation** with managed Lambda rotation functions; native rotation support for RDS, Redshift, DocumentDB |
| Cross-Region | Not natively replicated — needs custom sync | **Native multi-Region secret replication** (replica secrets) |
| Cross-account sharing | Via IAM only (no resource policy on Standard params); Advanced parameters can use RAM in limited cases | **Native resource-based policies** for direct cross-account access |
| Versioning | Supported | Supported, via version **stages**: `AWSCURRENT`, `AWSPENDING`, `AWSPREVIOUS` |
| Encryption | `SecureString` type → KMS. Plain `String`/`StringList` types are unencrypted | Always encrypted with KMS |
| Typical use | App config values, non-secret + secret parameters, direct references in CloudFormation/ECS task defs/Lambda env vars | Database credentials, API keys, anything needing automatic rotation |

🚩 **Distractor signal:** "requires automatic rotation" or "database credentials" → Secrets Manager. "Least cost, no rotation needed, general config" → Parameter Store.

---

### Skill 4.2.5 — Security Hub, GuardDuty, Config, Inspector

| Service | What it detects | Detect vs Remediate |
|---|---|---|
| **Security Hub** | Aggregates findings from GuardDuty, Inspector, Macie, IAM Access Analyzer, Config, and 3rd-party tools into standardized **ASFF** (AWS Security Finding Format). Runs compliance checks: CIS AWS Foundations Benchmark, PCI DSS, AWS Foundational Security Best Practices, NIST 800-53 | **Detect + aggregate only.** No native remediation — findings must trigger EventBridge → Lambda/SSM Automation |
| **GuardDuty** | Threat detection using CloudTrail (mgmt + S3 data events), VPC Flow Logs, DNS logs, EKS audit logs, RDS login activity, Lambda network activity; **EBS malware protection** (agentless on-demand/automatic scanning) | **Detect only.** Set up EventBridge rule → Lambda for auto-remediation (isolate instance, revoke creds, quarantine) |
| **Inspector** | Automated vulnerability scanning: EC2 (via SSM Agent — network reachability + software CVEs), ECR images (on push + continuous rescan as new CVEs are published), Lambda functions (code + dependencies) | **Detect only.** Requires SSM Agent + managed instance for EC2 coverage |
| **Config** | Records configuration changes/history and evaluates compliance via Config Rules (AWS managed or custom) | **Detect AND remediate** — Config Rules can have a **remediation action** (SSM Automation document) attached directly in the console, auto-triggered on non-compliance |

🚩 **The core exam distractor:** GuardDuty/Inspector/Macie = **detect only**, need external EventBridge+Lambda wiring for remediation. **Config is the one service in this group with native, built-in auto-remediation** via SSM Automation attached to a rule. If a question wants "least operational overhead" auto-remediation of a misconfiguration, the answer is almost always **Config + SSM Automation remediation**, not a custom Lambda triggered by GuardDuty.

---

## Domain 4 — Key Numbers Reference

| Item | Value |
|---|---|
| CloudTrail Event History retention (no trail needed) | 90 days, management events only |
| CloudTrail log delivery latency to S3 | ~15 minutes typical |
| Access Analyzer policy generation lookback | Up to 90 days of CloudTrail activity |
| Parameter Store Standard param size | 4 KB |
| Parameter Store Advanced param size | 8 KB |
| Secrets Manager secret size | 64 KB (65,536 bytes) |
| KMS CMK automatic rotation period | 1 year (annual), opt-in |
| KMS direct `Encrypt` API payload limit | 4 KB (hence envelope encryption for larger data) |
| CloudFront ACM cert Region requirement | us-east-1 only |
| CloudHSM/KMS FIPS levels | CloudHSM: FIPS 140-2 Level 3. KMS service: Level 2 (Level 3 HSMs underneath) |
| Trusted Advisor free-tier core security checks | 7 checks (root MFA, IAM use, S3 permissions, security groups specific ports, EBS/RDS public snapshots) |

---

That closes out Domain 4. Want scenario-based practice questions on this material now, or should we move straight into the Domain 5 deep pass per the study plan?