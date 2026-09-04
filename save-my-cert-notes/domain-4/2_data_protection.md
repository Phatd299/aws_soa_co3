Data Protection on AWS: KMS, ACM, Secrets, and Security Services (SOA-C03)

Data and infrastructure protection is the operational discipline of classifying what you store, encrypting it at rest and in transit, keeping secrets out of code, and remediating the findings your security services raise. Task 4.2 of the SOA-C03 exam tests all of it from the operator's seat: enabling default EBS encryption, working the snapshot-copy-restore sequence for an unencrypted RDS instance, diagnosing a KMS AccessDenied, keeping ACM certificates renewing, choosing Secrets Manager over Parameter Store when rotation matters, and telling GuardDuty, Inspector, Macie, Config, and Security Hub apart in one read of the question stem. Domain 4 carries 16% of your scored content, and this task supplies its most mechanical, most predictable questions — the encryption sequences and the service-chooser items are near-guaranteed marks once you know the patterns. This lesson covers skills 4.2.1 through 4.2.5.

What you’ll learn
Implement a data classification scheme with tagging and discover misplaced sensitive data in S3 with Amazon Macie.
Configure encryption at rest with KMS — default EBS encryption, key policies versus IAM, rotation — and troubleshoot KMS AccessDenied errors.
Encrypt existing unencrypted EBS volumes and RDS instances with the snapshot, copy-with-encryption, restore sequence.
Manage ACM certificates with DNS validation and auto-renewal, and enforce TLS with aws:SecureTransport and ALB HTTPS redirects.
Select the right secrets store, distinguish GuardDuty, Inspector, Macie, Config, and Security Hub, and wire findings into automated remediation.
Classify first, then protect
A data classification scheme decides how much protection each dataset gets, and every control in this lesson hangs off it. A typical three-tier scheme — public, internal, and confidential — maps directly to controls: public data needs integrity but not secrecy; internal data gets baseline encryption and private access; confidential data (PII, credentials, financial records) gets customer managed KMS keys, tight resource policies, access logging, and active monitoring.

Operationally, classification lives in tags. A consistent tag such as DataClassification=confidential on buckets, volumes, and databases lets you audit coverage, scope AWS Config rules to sensitive resources, and write policies that treat tiers differently. Tagging is cheap; retrofitting classification after an incident is not.

Amazon Macie is the discovery tool that keeps the scheme honest for S3. Macie uses machine learning and pattern matching to find sensitive data — names, credentials, card numbers, and other PII — inside S3 objects, and it reports where your classification is wrong: the "internal" bucket that actually holds customer PII. Enable automated sensitive data discovery to sample objects across your buckets continuously, or run targeted discovery jobs against specific buckets when you need full coverage. Macie also evaluates bucket-level posture, flagging buckets that are publicly accessible or unencrypted. Findings flow to EventBridge and Security Hub, so a Macie discovery can trigger the same remediation loop as any other finding. When the exam says "identify sensitive data stored in S3," the answer is Macie — not GuardDuty, which watches for threats, and not Inspector, which scans for vulnerabilities.

Encryption at rest with AWS KMS
KMS underpins encryption at rest across AWS, and the ops questions focus on configuration and access, not cryptography. Conceptually, services use envelope encryption: your KMS key encrypts data keys, and data keys encrypt the actual data. That is as deep as the exam goes.

Your first account-level move is enabling default EBS encryption. It is a per-Region, per-account setting; once on, every new volume and snapshot copy in that Region is encrypted with the key you choose (the AWS managed key by default), with no application changes required. It does not touch existing volumes.

Key policies versus IAM is the control point to internalize. Every KMS key has a resource-based key policy, and it is the primary authority: if the key policy does not permit an action — directly, or by delegating to IAM with the standard "enable IAM policies" statement — no IAM policy can override it. Cross-account key use requires both sides: the key policy must allow the external principal, and that principal's IAM policy must allow the KMS actions. Grants provide temporary, programmatic access and are commonly created by AWS services on your behalf.

Enable automatic key rotation on customer managed keys and KMS rotates the key material yearly, retaining old material so existing ciphertext still decrypts.

When a workload throws a KMS AccessDenied, triage in order: the key policy (does it allow this principal, or delegate to IAM?), the caller's IAM policy (does it allow kms:Decrypt or kms:GenerateDataKey on this key?), then grants. An EC2 instance failing to use an encrypted volume is almost always missing KMS permissions on the volume's key, not EBS permissions.

Walkthrough: encrypting resources that already exist
You cannot switch on encryption in place for an existing EBS volume or RDS instance — the exam loves this fact. The fix is always the same shape: snapshot, copy with encryption, restore.

Scenario: a production RDS MySQL instance is unencrypted and a new compliance requirement says every database must be encrypted with a customer managed key. The sequence:

Take a snapshot of the unencrypted instance.
Copy the snapshot, enabling encryption on the copy and selecting the KMS key. The encrypted copy is the pivot — an unencrypted snapshot cannot be restored directly into an encrypted instance.
Restore a new DB instance from the encrypted snapshot copy.
Cut the application over — update the connection endpoint or rename the instances — during a maintenance window, because writes made after the snapshot will not carry over unless you re-sync them.
Verify the new instance, then delete the old one.
EBS follows the same pattern: snapshot the volume, copy the snapshot with encryption enabled, create a new volume from the encrypted copy, then stop the instance and swap the volumes. Enable default EBS encryption first so nothing new is ever created unencrypted again.

S3 is gentler: every bucket applies default encryption to new objects, with SSE-S3 as the baseline. Choose SSE-KMS when you need your own key, key-level access control, and a CloudTrail record of every key use. Changing the default only affects new writes — existing objects keep their original encryption until you copy them over themselves.

Encryption in transit with ACM
AWS Certificate Manager issues free public TLS certificates and deploys them to the services that terminate TLS for you: Elastic Load Balancing, CloudFront, and API Gateway. You never handle the private key; ACM manages storage and deployment, which is exactly what makes it operationally safe.

Request certificates with DNS validation: ACM gives you a CNAME record, you publish it in Route 53 (one click when the hosted zone is in the same account), and the certificate validates and then renews automatically for as long as the record stays in place. Email validation requires a human to respond at renewal time — avoid it. The classic expired-certificate incident has two root causes worth memorizing: the DNS validation record was deleted, so auto-renewal silently failed; or the certificate was imported into ACM rather than issued by it — imported certificates never auto-renew, so you must re-import before expiry and monitor expiration with CloudWatch or an AWS Config rule.

Enforcing TLS is the other half. On S3, add a bucket policy that denies all actions when aws:SecureTransport is false — plain HTTP requests are rejected outright. On an Application Load Balancer, keep port 80 open but configure its listener as a redirect to HTTPS, so clients are upgraded rather than stranded.

Finally, know the trade-off between terminating TLS at the load balancer — simpler certificate management and offloaded crypto, but plaintext between the LB and targets — and end-to-end encryption, where the ALB re-encrypts to HTTPS targets. A requirement that traffic be "encrypted in transit at every hop" is pointing you at end-to-end.

Storing secrets: Secrets Manager versus Parameter Store
Secrets never belong in code, user data scripts, AMIs, or plaintext environment files — you cannot protect what has already leaked into an artifact. The exam's wrong answers put database passwords in user data; the right answers fetch them at runtime, over an IAM-authorized API call, from one of two services.

AWS Secrets Manager is purpose-built for secrets: KMS-encrypted storage, fine-grained IAM and resource-policy control, and — its defining feature — automatic rotation. For Amazon RDS, rotation is a managed integration: Secrets Manager changes the database password on schedule and updates the stored secret, while applications simply fetch the current value at connect time. It charges per secret per month.

Systems Manager Parameter Store stores configuration, and its SecureString type is a KMS-encrypted parameter — a perfectly good home for a static secret. The standard tier is free, which matters at fleet scale. What it lacks is native rotation: nothing changes the underlying credential unless you build that machinery yourself.

The selection logic the exam probes: credentials that must rotate automatically — database passwords, API keys under a rotation policy — go to Secrets Manager; static configuration secrets where cost matters and rotation is not required go to Parameter Store SecureString. Both integrate with IAM for least-privilege reads, both encrypt with KMS keys you control, and both are retrieved at runtime by the workload's role — an instance profile or task role — so nothing sensitive is ever baked into the deployment artifact.

The findings services: GuardDuty, Inspector, Config, and Security Hub
Skill 4.2.5 is about configuring reports and remediating findings, so know exactly what each service finds.

Amazon GuardDuty is threat detection. It continuously analyzes CloudTrail events, VPC Flow Logs, and DNS query logs — no agents, no traffic interception — and raises findings for active threats: an EC2 instance communicating with a cryptocurrency-mining pool, credentials used from an anomalous location, port scanning, or data exfiltration patterns. GuardDuty detects; it does not prevent or block — remediation is a separate step you automate.

Amazon Inspector is vulnerability management. It continuously scans EC2 instances (via the Systems Manager agent), ECR container images, and Lambda functions for known CVEs and unintended network reachability, prioritizing findings by severity. Inspector tells you what could be exploited; GuardDuty tells you what is being attacked.

AWS Config contributes configuration compliance: rule evaluations — unencrypted volumes, open security groups, drifted settings — surface as findings alongside the rest.

AWS Security Hub is the aggregation layer. It ingests findings from GuardDuty, Inspector, Macie, Config, and partner tools into one normalized format, runs its own security standards (such as AWS Foundational Security Best Practices) with a posture score, and aggregates across accounts and Regions through a delegated administrator account — the "single pane of glass" phrasing in exam questions. The v1.1 exam guide also names AWS Security Agent alongside these services as an example for configuring reports and remediating security findings; recognize it by name as part of this family.

Choosing the right security service
Most Task 4.2 questions reduce to picking the service whose job matches the symptom in the stem. Anchor on what each one examines and what it produces:

Question you are answering	Service	What it produces
Is something attacking or abusing my account right now?	Amazon GuardDuty	Threat findings from CloudTrail, VPC Flow Logs, and DNS logs
Do my instances, images, or functions carry known vulnerabilities?	Amazon Inspector	CVE and network-reachability findings for EC2, ECR images, and Lambda
Is sensitive data sitting in my S3 buckets?	Amazon Macie	Sensitive-data findings (PII, credentials) for S3 objects
Are my resources configured the way policy requires?	AWS Config	Compliance evaluations against managed or custom rules
Where do I see all of it in one place?	AWS Security Hub	Aggregated findings, security standards, and scores across accounts
The distractors work because the services overlap in vocabulary, not in function. GuardDuty and Inspector both mention EC2, but only Inspector reports CVEs and only GuardDuty reports active compromise. Macie and GuardDuty both produce "findings," but Macie only ever examines data in S3. Security Hub generates standards findings of its own, yet it is never the primary detector — if the stem describes one specific symptom, pick the specialist; if it asks for one consolidated view, a security score, or cross-account aggregation, pick Security Hub.

From finding to fix: the automated remediation loop
A finding that sits unread in a console is not remediation, and the exam expects you to close the loop. Every service in this lesson — GuardDuty, Inspector, Macie, Config, and Security Hub — publishes its findings as events to Amazon EventBridge, which makes the operational pattern uniform:

Match: an EventBridge rule filters findings by source, type, and severity — for example, GuardDuty findings above a severity threshold.
Notify: route matched findings to an SNS topic so the on-call engineer always hears about it, whatever else happens.
Remediate: invoke a Lambda function or a Systems Manager Automation runbook that performs the fix — swap a compromised instance's security group for an isolation group and snapshot its volumes for forensics, revoke an open security group rule, or enable a bucket's Block Public Access.
AWS Config adds its own shortcut — a rule can attach an SSM remediation runbook directly, no EventBridge required — and Security Hub adds custom actions, which let an analyst send selected findings to EventBridge with one click for semi-automated response.

Two operational judgments matter. First, automate reversible containment freely — isolation, disabling credentials, notifications — but keep a human approval step in front of destructive actions like terminating instances. Second, remediation without a record is a lost audit trail: have the runbook tag what it touched and feed your ticketing flow. Detection → EventBridge → runbook is the same loop whether the trigger is a threat, a vulnerability, sensitive data, or configuration drift — learn it once, apply it everywhere.

💡
Tip. Task 4.2 questions lean on trigger phrases: "compromised instance mining cryptocurrency" → GuardDuty; "CVEs on EC2 instances or container images" → Inspector; "sensitive data in S3" → Macie; "single view of all findings" or "security score" → Security Hub; "resources must be configured according to policy" → AWS Config. Encryption items are procedural: expect the snapshot → copy-with-encryption → restore sequence for existing EBS volumes and RDS instances, aws:SecureTransport for forcing TLS to S3, and an expired certificate traced to a deleted DNS validation record or an imported (never auto-renewing) certificate. When a KMS AccessDenied appears, check the key policy before the IAM policy.

Key takeaways
You cannot encrypt an existing EBS volume or RDS instance in place — snapshot, copy the snapshot with encryption, restore, and cut over.
The KMS key policy is the primary authority: an IAM allow is useless if the key policy neither permits the principal nor delegates to IAM.
ACM certificates auto-renew only while their DNS validation records exist; imported certificates never auto-renew.
Enforce TLS on S3 with a bucket-policy Deny when aws:SecureTransport is false; on an ALB, redirect the HTTP listener to HTTPS.
Secrets Manager when you need automatic rotation (managed for RDS); Parameter Store SecureString for free, static, KMS-encrypted secrets.
GuardDuty = active threats, Inspector = CVEs and vulnerabilities, Macie = sensitive data in S3, Config = configuration compliance, Security Hub = aggregation and standards.
GuardDuty analyzes CloudTrail, VPC Flow Logs, and DNS logs — it detects threats; it never blocks them.
Findings → EventBridge rule → SNS plus Lambda/SSM runbook is the standard automated remediation loop for every findings service.