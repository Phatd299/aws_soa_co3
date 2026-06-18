### IAM Advanced Concepts
---

IAM policy evaluation is the foundation of everything in this domain. When an AWS API call is made, IAM evaluates all applicable policies and decides whether to allow or deny. The evaluation order is: explicit Deny wins above everything else, then service control policies (SCPs), then resource-based policies, then identity-based policies, then permission boundaries, then session policies. The effective permission is the intersection of all these layers — not the union.

The default is implicit deny. If no policy explicitly allows an action, it is denied. An explicit Deny in any policy overrides any number of explicit Allows. This is the most important rule in IAM policy evaluation.

Identity-based policies attach to IAM users, groups, and roles. They define what the principal can do. Managed policies (AWS-managed or customer-managed) can be attached to multiple principals. Inline policies are embedded in a single principal and deleted when the principal is deleted. AWS-managed policies are maintained by AWS and updated when services add features. Customer-managed policies give you full control.

Resource-based policies attach to resources — S3 bucket policies, KMS key policies, SQS queue policies, Lambda resource policies, SNS topic policies. They define who can do what to that resource. Resource-based policies can grant access to principals in other accounts without requiring the cross-account principal to assume a role — this is the distinction between resource-based cross-account access and role-based cross-account access.

For same-account access: if an identity-based policy or resource-based policy allows an action, access is granted. Both don't need to allow it — either one is sufficient for same-account access (unless there's a Deny somewhere).

For cross-account access: both the identity-based policy in the calling account AND the resource-based policy on the resource must allow the action. Either alone is insufficient. Exception: if the resource-based policy grants access to an account (not a specific principal within it), then principals in that account only need their identity-based policy to allow it.

Permission boundaries are managed IAM policies attached to a user or role that define the maximum permissions that principal can ever have. The principal's effective permissions are the intersection of the boundary and the identity-based policy. If the boundary allows S3 and EC2, but the identity policy only allows S3, the principal can only use S3. If the identity policy allows S3 and RDS, but the boundary only allows S3, the principal can only use S3. The boundary does not grant permissions — it only limits them.

The canonical use case for permission boundaries: you want to allow developers to create IAM roles for their applications, but you don't want them to be able to create roles with more permissions than they themselves have. You attach a permission boundary to the developer's IAM user that limits what they can do, and you add a condition to their IAM policy that requires any roles they create to also have that permission boundary. This is called delegated administration.

SCPs are Organization-level policies applied to OUs or accounts that restrict what principals in those accounts can do. An SCP is not a grant — it's a ceiling. Even if an IAM policy in an account allows an action, if the SCP doesn't allow it, the action is denied. The management account of the organization is never affected by SCPs — SCPs only apply to member accounts.

SCPs use the same JSON policy language as IAM policies. You can write allow-list SCPs (deny everything except what's listed — requires an explicit Deny * with exceptions) or deny-list SCPs (deny specific actions — simpler, AWS applies a FullAWSAccess SCP by default). The exam favors deny-list SCPs because they're additive — you can stack multiple deny SCPs without risk of accidental lockout.

Session policies are passed when you assume a role or federate a user. They further restrict the permissions of the resulting session beyond what the role's identity policy allows. Used in scenarios where a broker assumes a role and then passes a subset of that role's permissions to the end user based on context.

IAM Roles for EC2 are delivered via the instance metadata service (IMDS). The application calls http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name> to get temporary credentials. IMDSv2 requires a session token obtained via a PUT request before metadata can be accessed — this prevents SSRF attacks from accessing the metadata service. You should enforce IMDSv2 at the account level via a Service Control Policy or at the instance level via launch template settings.

IAM Access Analyzer has two major functions. First, it identifies resources that are accessible from outside your zone of trust (the account or Organization you define). It analyzes resource-based policies on S3 buckets, KMS keys, IAM roles, Lambda functions, SQS queues, and Secrets Manager secrets, and generates findings when it detects external access. Second, it generates least-privilege IAM policies based on CloudTrail activity. You specify a time window, Access Analyzer analyzes what API calls were actually made by a principal, and generates a policy granting exactly those permissions. This is the automated least-privilege workflow.

IAM policy conditions are key-value pairs that add context to policy evaluation. Common condition keys: aws:RequestedRegion (restrict actions to specific regions), aws:SourceIp (restrict to IP ranges), aws:MultiFactorAuthPresent (require MFA), aws:CurrentTime (time-based access), s3:prefix (restrict S3 access to key prefixes), kms:ViaService (restrict KMS key usage to specific services). Condition operators: StringEquals, StringLike (supports wildcards), ArnLike, IpAddress, DateLessThan, Bool, and Null (check whether a key is present).

Service-linked roles are IAM roles predefined by AWS services. They have exactly the permissions the service needs to function. You can't modify the trust policy or the permissions policy. They're created automatically when you enable the service (or manually if automatic creation isn't supported). Examples: AWSServiceRoleForElasticLoadBalancing, AWSServiceRoleForAutoScaling.

### KMS and Encryption

KMS manages cryptographic keys used to encrypt AWS resources and application data. The service is regional — keys are specific to a region and never leave the region (unless you explicitly use multi-region keys).

Key types: AWS-managed keys are created automatically by services when you first use encryption with that service (aws/s3, aws/rds, aws/ebs, etc.). You cannot manage these keys directly — no custom rotation schedule, no custom key policy, no cross-account sharing. They rotate automatically every year. Customer-managed CMKs (Customer Master Keys) are keys you create. You control the key policy, rotation, and usage. They cost $1/month plus $0.03 per 10,000 API calls. Imported key material: you generate key material outside AWS and import it into KMS. AWS stores and uses the key material but you control it and can delete it by deleting the key material (the key remains but is unusable until you re-import). Multi-region keys: the same key material in multiple regions, with the same key ID. Allows you to encrypt in one region and decrypt in another without re-encryption. Used for global applications and disaster recovery.

The KMS key policy is a resource-based policy on the key. Unlike most AWS resource policies, the KMS key policy is required — you cannot use IAM policies alone to grant access to a KMS key. The key policy must explicitly allow access. If you accidentally lock yourself out of a key by writing a key policy with no valid principals, the key becomes unrecoverable (unless you have the root account listed, which is why best practice is always to include the root account).

The standard key policy pattern: grant the AWS account root user full access (this enables IAM policies in the account to control access — root access in key policy is not the same as directly granting root user permissions), then grant specific IAM roles or users the ability to use the key (kms:Encrypt, kms:Decrypt, kms:GenerateDataKey, kms:DescribeKey) and separately grant key administrators the ability to manage the key (kms:Create*, kms:Describe*, kms:Enable*, kms:List*, kms:Put*, kms:Update*, kms:Revoke*, kms:Disable*, kms:Delete*, kms:TagResource, kms:UntagResource, kms:ScheduleKeyDeletion, kms:CancelKeyDeletion).

Cross-account KMS usage: the key policy in the key-owning account must explicitly allow the external account or specific principals in it. The external account's IAM policy must also allow the KMS actions. Both conditions must be met. When you use an encrypted resource across accounts (like sharing an RDS snapshot encrypted with a CMK), the recipient must have access to the CMK used for encryption. The clean solution is to re-encrypt with a key in the recipient's account.

Envelope encryption is the pattern KMS uses and recommends for encrypting large amounts of data. KMS has a 4KB limit on data it can encrypt directly. For anything larger, you use envelope encryption: call GenerateDataKey to get a plaintext DEK (Data Encryption Key) and an encrypted DEK. Use the plaintext DEK to encrypt your data with AES-256 locally. Discard the plaintext DEK immediately. Store the encrypted DEK alongside the encrypted data. To decrypt, call KMS Decrypt with the encrypted DEK to get the plaintext DEK back, then use it to decrypt the data. The plaintext DEK is never stored anywhere persistently — it only exists in memory during the encryption/decryption operation. KMS never sees your actual data.

GenerateDataKeyWithoutPlaintext generates a DEK but returns only the encrypted version. Use this when you want to prepare encrypted DEKs in advance without immediately having the plaintext key in memory — for example, generating encrypted DEKs to distribute to workers.

KMS key rotation: for customer-managed CMKs, you enable automatic annual rotation. When the key rotates, KMS generates new key material and uses it for all new encryption operations. The old key material is retained and used to decrypt data encrypted with it previously. The key ID and ARN remain the same — applications don't need to be updated. Rotation is transparent to applications. Imported key material cannot be automatically rotated — you must manually delete and re-import new material, which changes the key.

Key deletion: you cannot immediately delete a CMK. You schedule deletion with a waiting period of 7 to 30 days. During the waiting period, the key is disabled and cannot be used for cryptographic operations. You can cancel the deletion during the waiting period. Once deleted, all data encrypted solely with that key becomes permanently unrecoverable. Before deleting a key, use CloudTrail to verify nothing is using it and enable CloudWatch alarms on the kms:Decrypt and other API calls for that key ID to detect any usage.

KMS grants are an alternative to key policy updates for delegating key usage. A grant gives a specific principal permission to perform specific KMS operations, optionally with constraints (like requiring an encryption context value). Grants are useful for temporary delegations — a service like EBS creates a grant when it needs to use your CMK to encrypt a volume, and can retire the grant when done. Grants don't modify the key policy, so they don't require key administrator permissions to create (just kms:CreateGrant in the caller's IAM policy).

Encryption context is a set of key-value pairs you pass to KMS encryption operations. It's not secret — it's logged in CloudTrail and can be used in key policy conditions. If you encrypt with a specific encryption context, you must provide the same encryption context to decrypt. This binds ciphertext to its context — a ciphertext encrypted for one environment cannot be decrypted with an encryption context for a different environment. It's an additional integrity check, not a confidentiality mechanism.

S3 server-side encryption options: SSE-S3 uses AWS-managed keys, fully managed, no additional cost, no key policy control. SSE-KMS uses a KMS CMK, gives you key policy control, access logging via CloudTrail, and the ability to require MFA to delete objects. SSE-C uses a key you provide with each request — S3 performs the encryption but doesn't store your key. Client-side encryption means you encrypt before uploading — S3 stores ciphertext and knows nothing about the encryption. The exam usually asks about SSE-KMS when the requirement mentions key control, audit logging of decryption events, or cross-account encryption.

EBS encryption: when you enable encryption for an EBS volume, all data at rest, data in transit between the volume and the instance, snapshots, and volumes created from those snapshots are encrypted. There's no performance impact in modern instance families — encryption is handled in hardware. You can enforce EBS encryption at the account level so all new volumes are automatically encrypted.

### GuardDuty

GuardDuty is a threat detection service that uses ML and threat intelligence to identify malicious or anomalous activity. It analyzes data sources you don't have to configure — it reads VPC Flow Logs, CloudTrail management events, CloudTrail S3 data events (if enabled), DNS query logs, and Kubernetes audit logs (if enabled for EKS) automatically. You enable GuardDuty with one click and it starts analyzing within minutes.

GuardDuty findings have a severity (Low, Medium, High) and a finding type structured as ThreatPurpose:ResourceTypeAffected/ThreatFamilyName.DetectionMechanism!Artifact. For example: UnauthorizedAccess:EC2/SSHBruteForce, CryptoCurrency:EC2/BitcoinTool.B, Trojan:EC2/BlackholeTraffic, Recon:IAMUser/UserPermissions.

Common finding categories the exam references: cryptocurrency mining activity on EC2 (high CPU, connections to known mining pools), unauthorized API calls from unusual locations, EC2 instances communicating with known malicious IPs or domains, credential exfiltration (API calls from unusual geographic locations or at unusual times), S3 bucket reconnaissance, compromised IAM credentials being used from a TOR exit node.

GuardDuty does not block anything. It generates findings. To respond to findings, you use EventBridge to catch GuardDuty finding events and trigger automated responses — Lambda to isolate an instance (remove from ASG, apply restrictive security group), SNS to notify the security team, Step Functions for a multi-step response workflow.

Multi-account GuardDuty: you designate an administrator account (via Organizations or manual invitation). The administrator account can see all findings from all member accounts in a centralized view. Member accounts cannot disable GuardDuty if the administrator has enabled it for the organization. This prevents attackers who compromise a member account from disabling threat detection.

GuardDuty has optional protection plans: S3 Protection (analyzes S3 data plane events), EKS Protection (analyzes Kubernetes audit logs), Malware Protection (scans EBS volumes attached to flagged EC2 instances for malware), RDS Protection (analyzes RDS login activity), Lambda Protection (analyzes Lambda network activity), and Runtime Monitoring (uses an agent to detect threats at the OS level on EC2 and ECS).

Suppression rules let you automatically archive findings that match a filter. Use this to suppress known-safe findings — for example, suppress port scan findings from your security scanner's IP address. Suppressed findings are not deleted — they're archived and still available for review.

### Inspector

Inspector performs automated vulnerability assessments. It scans for software vulnerabilities (CVEs) and unintended network exposure. There are two assessment types.

EC2 instance scanning uses the SSM Agent to scan the instance's package list against the CVE database. It requires the SSM Agent to be installed and the instance to be registered as a managed instance. It identifies which packages have known vulnerabilities and reports the severity (Critical, High, Medium, Low, Informational) with the CVE ID, CVSS score, and remediation guidance.

ECR container image scanning integrates with Amazon ECR. When you push an image to ECR, Inspector can automatically scan it for OS package vulnerabilities and programming language package vulnerabilities. Findings appear in both Inspector and ECR. This is the exam pattern for "scan container images for vulnerabilities before deployment."

Inspector findings include a risk score that accounts for the vulnerability severity, the network exposure of the resource (is this EC2 instance publicly accessible?), and whether exploits exist in the wild. An EC2 instance with a critical CVE but no internet exposure scores lower risk than the same instance with public internet access.

Inspector does not remediate. It generates findings with remediation recommendations (update this package to version X). Acting on those findings is your responsibility — you might use SSM Patch Manager to apply patches, or rebuild a container image with the fixed base image.

Multi-account Inspector: delegate an administrator account via Organizations. The administrator gets aggregated findings across all member accounts.

### AWS Config

Config is a configuration history and compliance service. It records the configuration state of AWS resources over time and evaluates those configurations against rules you define.

The Config recorder captures configuration changes to supported resources. When you enable Config, you choose which resource types to record (all supported resources, or a specific list). Config creates configuration items — snapshots of resource configuration at a point in time — whenever a resource changes. These items are stored in an S3 bucket (the delivery channel). Config also maintains a configuration history for each resource and a configuration snapshot of all resources at a point in time.

Config rules evaluate whether resources comply with your desired configuration. AWS managed rules cover common compliance requirements: ec2-instances-in-vpc (ensures all EC2 instances are in a VPC), encrypted-volumes (ensures EBS volumes are encrypted), s3-bucket-public-read-prohibited, rds-instance-public-access-check, iam-password-policy, mfa-enabled-for-iam-console-access, and hundreds more. Custom rules are Lambda functions that evaluate resource configuration and return COMPLIANT or NON_COMPLIANT.

Rules can be triggered in two ways: configuration change (evaluates the rule whenever the relevant resource type changes) or periodic (evaluates all resources of the relevant type on a schedule — every hour, every 3 hours, every 6 hours, every 12 hours, or every 24 hours). Change-triggered rules provide near-real-time compliance feedback.

Remediation actions are the critical feature that distinguishes Config from the detection-only services. You associate a remediation action with a rule — typically an SSM Automation document. When Config marks a resource as NON_COMPLIANT, you can manually trigger the remediation or configure auto-remediation. With auto-remediation, Config automatically runs the SSM Automation document to fix the non-compliant resource without human intervention. Example: the s3-bucket-public-read-prohibited rule marks a bucket as NON_COMPLIANT; the auto-remediation runs the AWS-DisableS3BucketPublicReadWrite automation document to make the bucket private.

Conformance packs are collections of Config rules and remediation actions packaged together. AWS provides sample conformance packs for common compliance frameworks: CIS AWS Foundations Benchmark, PCI DSS, HIPAA, NIST 800-53. You deploy a conformance pack to apply all the rules in one operation. Conformance packs can be deployed across an Organization using the Organizations integration.

Config aggregators collect Config data from multiple accounts and regions into a single view. An aggregator is created in a central account and pulls configuration items, compliance data, and rule results from member accounts. Member accounts must authorize the aggregator account. With Organizations integration, you can aggregate all accounts automatically.

Resource relationships: Config tracks relationships between resources — for example, an EC2 instance is associated with a security group, which is associated with a VPC. You can view the complete resource relationship graph in the Config console and query it via the Config API. This is useful for impact analysis — "which EC2 instances use this security group?"

Config timeline: for any resource, you can view its complete configuration history — every configuration change recorded as a configuration item with a timestamp. You can compare configurations at two points in time to see exactly what changed. Paired with CloudTrail, you can correlate configuration changes with the API calls that caused them.

### Macie

Macie uses ML to automatically discover, classify, and protect sensitive data in S3. It identifies PII (names, addresses, social security numbers, passport numbers, credit card numbers), financial data, credentials (access keys, passwords), protected health information, and custom data types you define.

Sensitive data discovery jobs scan S3 buckets on a schedule or one-time. You specify which buckets to scan, which objects to include (by prefix, file extension, or object size), and the sampling rate. Macie stores findings in an S3 bucket and reports them in the console and to EventBridge.

Macie also provides bucket-level security findings without running a discovery job. It continuously monitors bucket policies, ACLs, and encryption settings and reports: unencrypted buckets, publicly accessible buckets, buckets shared with external accounts, and buckets accessible from outside the Organization. These are policy findings, separate from the sensitive data findings.

Custom data identifiers let you define your own patterns for sensitive data — regular expressions with optional keywords and ignore words. For example, you might define a pattern for your company's internal employee ID format.

Multi-account Macie: delegate an administrator account via Organizations. Centralized finding management across all accounts. The administrator can create discovery jobs that scan buckets across all member accounts.

Macie is specifically for S3. If a question involves sensitive data detection on EC2, RDS, or any non-S3 service, Macie is not the answer. For those scenarios, you'd use application-level controls, database activity monitoring, or third-party DLP tools.

### Security Hub
---

Security Hub is the aggregation and normalization layer for security findings across AWS security services and third-party tools. It ingests findings from GuardDuty, Inspector, Macie, Config, IAM Access Analyzer, Firewall Manager, Systems Manager Patch Manager, and third-party integrations (CrowdStrike, Palo Alto, Splunk, etc.).

Security Hub normalizes all findings into the AWS Security Finding Format (ASFF) — a standard JSON schema. This means regardless of which service generated the finding, it has the same fields (title, severity, resource, remediation, etc.) in Security Hub.

Security standards are pre-packaged sets of security checks. Available standards: AWS Foundational Security Best Practices, CIS AWS Foundations Benchmark v1.2 and v1.4, PCI DSS, NIST SP 800-53. Enabling a standard automatically creates Config rules for the checks in that standard and scores your compliance as a percentage.

Security score: Security Hub computes a security score per standard (and an overall score) as a percentage of checks that are passing. This gives you a high-level posture view without having to look at individual findings.

Automated response and remediation: Security Hub sends all findings to EventBridge. You create EventBridge rules that match specific finding types or severities and trigger automated responses — Lambda, Step Functions, SNS, SSM Automation. AWS provides Security Hub Automated Response and Remediation (SHARR) — a solution with pre-built response playbooks for common finding types.

Insights are correlated views of findings — filtered, grouped, and aggregated. AWS provides managed insights (top products generating findings, EC2 instances with most findings, etc.). You can create custom insights.

Multi-account Security Hub: designate an administrator account. The administrator sees all findings from all member accounts. Findings from member accounts appear with the member account ID so you can identify which account they came from. Cross-region aggregation lets you view findings from all regions in a single Region.

### WAF and Shield
---

WAF is a web application firewall that operates at Layer 7 — it inspects HTTP/HTTPS request content. You attach WAF to CloudFront, ALB, API Gateway, AppSync, or Cognito user pools via Web ACLs.

A Web ACL contains rules and rule groups, each with a priority. Rules are evaluated in priority order (lowest number first). The first matching rule's action is applied — Allow, Block, Count, or CAPTCHA. Count mode is crucial for testing: it evaluates the rule and increments a metric but doesn't block, letting you see what would be blocked without affecting traffic.

Rule types: IP set rules (match against a list of IP addresses or CIDR ranges), geo-match rules (match requests from specific countries), rate-based rules (match requests from a single IP that exceed a threshold in 5 minutes — automatic IP-level rate limiting), regex pattern set rules (match request components against regular expressions), and managed rule groups.

AWS Managed Rule Groups are pre-built rule sets maintained by AWS: Core Rule Set (OWASP Top 10 attacks), Known Bad Inputs, SQL Database (SQLi protection), Linux Operating System, PHP Application, WordPress Application, Amazon IP Reputation List (known malicious IPs), Anonymous IP List (TOR, VPN, hosting providers). AWS Marketplace managed rule groups are provided by security vendors (F5, Imperva, etc.).

WAF Bot Control is a managed rule group specifically for bot traffic. It classifies bots as verified (Googlebot, Bingbot — legitimate crawlers), unverified, and malicious, and lets you allow/block/challenge each category.

Fraud Control (Account Takeover Prevention and Account Creation Fraud Prevention) are specialized managed rule groups that detect credential stuffing, account takeover attempts, and fake account creation using ML.

WAF Logging: you can log all requests (or a sample) to CloudWatch Logs, S3, or Kinesis Data Firehose. Logs include the rule that matched, the action taken, and the full request details. Essential for forensics and tuning.

Shield Standard is automatically applied to all AWS customers at no cost. It protects against common, volumetric network and transport layer DDoS attacks — SYN/ACK floods, UDP reflection attacks, DNS amplification. This protection is always on and transparent.

Shield Advanced provides enhanced protection with additional features: more sophisticated DDoS detection and mitigation, near-real-time attack visibility and notifications via CloudWatch, access to the Shield Response Team (SRT) 24/7 for assistance during attacks, automatic application layer (Layer 7) DDoS mitigation (with WAF), DDoS cost protection (AWS credits you for scaling costs incurred during a DDoS attack), and attack diagnostics with detailed post-attack reports. Shield Advanced is $3,000/month with a 1-year commitment, plus data transfer out costs. You attach it to specific protected resources: CloudFront distributions, Route 53 hosted zones, ELB load balancers, EC2 Elastic IPs.

Firewall Manager is the centralized management service for WAF rules, Shield Advanced protections, Security Groups, and Network Firewall policies across an Organization. You define security policies in the management account and Firewall Manager automatically applies them to all existing and new resources in member accounts. The key value: when a new account joins the Organization or a new ALB is created, Firewall Manager ensures the WAF Web ACL is automatically attached without any manual action.

### Secrets Manager

Secrets Manager stores and manages secrets — database credentials, API keys, OAuth tokens, SSH keys. Unlike Parameter Store SecureString, Secrets Manager is purpose-built for secrets and adds automatic rotation.

Automatic rotation uses a Lambda function that Secrets Manager invokes on a schedule. AWS provides rotation Lambda templates for RDS (MySQL, PostgreSQL, Oracle, SQL Server), Redshift, DocumentDB, and generic secrets. The rotation Lambda updates both the secret value in Secrets Manager and the underlying credential (e.g., the database password) atomically. During rotation, the old secret remains valid until the rotation is confirmed complete — this prevents application downtime during rotation.

The rotation strategy: single-user rotation updates one database user's password. Multi-user rotation maintains two users (one active, one pending rotation) and alternates between them, ensuring there's always a valid set of credentials available even mid-rotation.

Secrets are versioned. Each rotation creates a new version. Versions are labeled with staging labels: AWSCURRENT (the active version), AWSPENDING (being rotated to), AWSPREVIOUS (just rotated from — retained briefly for fallback). Applications should always retrieve the AWSCURRENT version. Secrets Manager caches the secret in the client SDK for a configurable period to reduce API call frequency.

Resource-based policies on secrets control cross-account access. You can grant another account's IAM principals access to retrieve a secret without requiring them to assume a role in your account.

Secrets Manager also integrates with CloudFormation via dynamic references — you reference a secret in a template as {{resolve:secretsmanager:MySecret:SecretString:password}} and CloudFormation retrieves the current value at deployment time. This means you don't hardcode secrets in templates.

### Network Security: Security Groups and NACLs
---

Security Groups are stateful virtual firewalls attached to ENIs (Elastic Network Interfaces). Every EC2 instance, RDS instance, ELB, Lambda function (in a VPC), and other ENI-backed resource has security groups.

Stateful means the connection tracking table maintains state. When you allow an inbound connection on port 443, the response traffic is automatically allowed regardless of outbound rules. You don't need to write a rule for the return traffic. This simplifies rule management — you only write rules for the traffic you initiate.

Security Groups only have allow rules. You cannot write a Deny rule. Default behavior: all inbound traffic is denied, all outbound traffic is allowed. When you modify a Security Group, changes take effect immediately for all associated ENIs.

Security Group referencing: instead of specifying an IP range in a Security Group rule, you can reference another Security Group as the source or destination. This means "allow traffic from any resource that has security group SG-ABC attached." This is the pattern for internal service-to-service communication — your application servers' Security Group references the load balancer's Security Group as the inbound source, rather than using an IP range that might change.

The limit is 60 inbound and 60 outbound rules per Security Group (default, can be increased), and 5 Security Groups per ENI (default, can be increased to 16).

NACLs operate at the subnet level. Every subnet has exactly one NACL; one NACL can be associated with multiple subnets. NACLs are stateless — each packet is evaluated independently. You must write rules for both inbound and outbound traffic, including the response.

For TCP connections, the client uses a random ephemeral source port (1024–65535). The server responds to that ephemeral port. In a NACL, you must allow outbound traffic to the ephemeral port range for responses to inbound connections, and allow inbound traffic from the ephemeral port range for responses to your outbound connections. This is the most common NACL misconfiguration.

NACLs have numbered rules evaluated in order from lowest to highest. The first matching rule is applied. Rule 100 is evaluated before rule 200. The * rule at the end is the default deny — if no numbered rule matches, the packet is denied. You should leave gaps in rule numbering (100, 200, 300) to allow inserting rules later without renumbering.

NACLs support explicit Deny rules. This is the critical distinction from Security Groups. If you need to block a specific IP address from reaching your VPC, the NACL Deny rule is the mechanism. Place the Deny rule with a lower number than any Allow rule that might match the same traffic so the Deny is evaluated first.

The exam scenario is typically: "A specific IP address is repeatedly attacking your application. How do you block it without modifying the application or Security Groups?" The answer is a NACL Deny rule for that IP in the subnet's NACL. For global blocking across a CloudFront distribution, the answer is a WAF IP set Block rule.

### AWS Organizations Security Controls
---

Organizations provides hierarchical account management. The root is the management account. Organizational Units (OUs) group accounts. SCPs apply to OUs or individual accounts.

The preventive controls pattern: SCPs deny actions that would create security risks regardless of what IAM policies within the account say. Common SCP patterns:

Deny disabling CloudTrail — prevents anyone in the account from running cloudtrail:StopLogging or cloudtrail:DeleteTrail. Even an account administrator cannot disable the audit log.

Deny leaving the Organization — prevents accounts from calling organizations:LeaveOrganization. Keeps accounts under centralized management.

Deny disabling GuardDuty — prevents member accounts from disabling threat detection.
Restrict regions — deny all actions not in an approved list of regions. Prevents resource creation in unapproved regions.

Require encryption — deny creation of unencrypted EBS volumes, unencrypted RDS instances, or S3 buckets without encryption.

Deny root user actions — deny all actions when the principal is the root user (except specific emergency actions). Forces use of IAM identities.

The deny-list approach to SCPs is operationally simpler than the allow-list approach. With the default FullAWSAccess policy on every OU, you layer additional Deny SCPs for specific restricted actions. With the allow-list approach, you replace FullAWSAccess with an explicit allow-list, which requires careful maintenance as AWS releases new services and features.

AWS Control Tower builds on Organizations to provide a landing zone — a well-architected multi-account environment with guardrails. Preventive guardrails are SCPs. Detective guardrails are Config rules. Control Tower sets up logging accounts, audit accounts, and enforces guardrails across the organization with minimal manual configuration.