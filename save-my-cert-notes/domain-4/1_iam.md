# IAM, AWS Organizations, and Compliance Tools for CloudOps (SOA-C03)

Security and compliance tooling on AWS is the operational layer that controls who can do what: IAM authenticates and authorizes principals, AWS Organizations and service control policies set guardrails across every account, and AWS Config continuously verifies that resources stay compliant. Task 4.1 of the SOA-C03 exam tests this from the operator's seat — enforcing MFA, replacing long-term access keys with roles, triaging AccessDenied errors layer by layer, proving who did what with CloudTrail, and blocking entire Regions with one policy. Domain 4 carries 16% of your scored content, and these questions are scenario-driven: the exam drops you into a requirement like "no user in any account may create resources outside eu-west-1" and expects you to pick the right tool at the right layer. This lesson covers skills 4.1.1 through 4.1.5, from IAM features through Trusted Advisor remediation and AWS Config conformance packs.

What you’ll learn
Enforce MFA and password policies, and replace long-term access keys with IAM roles for workloads and cross-account access.
Apply policy conditions (aws:SourceIp, aws:PrincipalOrgID, aws:RequestedRegion) and permissions boundaries, and predict outcomes with the policy evaluation order.
Triage AccessDenied errors layer by layer and audit access with CloudTrail, the IAM policy simulator, and IAM Access Analyzer.
Design multi-account guardrails with AWS Organizations, SCPs, and IAM Identity Center, and explain what SCPs can and cannot do.
Remediate Trusted Advisor security findings and enforce continuous compliance with AWS Config rules, auto-remediation, and conformance packs.
Core IAM features in daily operations
Day-to-day IAM operations come down to three controls: the account password policy, MFA enforcement, and replacing long-term access keys with roles.

The account password policy sets minimum length, character requirements, reuse prevention, and optional expiry. It applies only to IAM user console passwords — it does not govern access keys, federated users, or the root user. Configure it once per account and audit it with the credential report.

MFA is registered per user in the console, but registration is not enforcement. To enforce it, attach an IAM policy that denies every action except the MFA-management actions whenever aws:MultiFactorAuthPresent is not true. The AWS-recommended pattern uses a Deny with BoolIfExists, so requests signed with long-term access keys — which never carry MFA context — are denied as well. Always enable MFA on the root user first; a missing root MFA device is one of the most repeated Trusted Advisor security findings.

Roles beat long-term keys everywhere a workload needs AWS access. An EC2 instance gets credentials through an instance profile; ECS tasks and Lambda functions use task and execution roles; principals in other accounts assume a cross-account role with sts:AssumeRole instead of receiving shared keys. Role credentials are short-lived and rotated automatically, so there is nothing to leak into code or AMIs. On the exam, any option that ships access keys onto an instance, bakes them into an AMI, or shares them between accounts is wrong whenever a role is available — which is essentially always.

Federated identity and the two policy families
Federated identity means your people authenticate against an external identity source and receive temporary AWS credentials — no IAM users to create, no passwords to rotate. You have two main routes. IAM Identity Center is the operationally preferred one for multi-account environments: connect it to your identity provider (or use its built-in directory), define permission sets, and assign them to users and groups per account; Identity Center provisions the matching roles into each account and gives users a single sign-on portal. Direct SAML 2.0 federation with IAM establishes a trust between your IdP and an individual account, and users assume specific roles via SAML assertions. It works, but you configure and maintain it account by account, which is why the exam favors Identity Center at scale.

Separately, know the two policy families. Identity-based policies attach to users, groups, and roles and state what that principal can do. Resource-based policies attach to the resource itself — S3 bucket policies, SQS queue policies, SNS topic policies, KMS key policies — and name a Principal that may access it.

The distinction matters most across accounts. Within one account, an allow from either family is enough. Across accounts, both sides must allow: the caller's identity policy must permit the action, and the resource policy in the target account must name the caller (or its account) as a principal. Resource-based access also lets the caller keep its own identity — no role switching — which is why cross-account S3 bucket policies appear so often in ops scenarios.

Policy conditions, permissions boundaries, and evaluation order
Condition keys turn a broad allow or deny into a targeted control, and three of them dominate this exam:

aws:SourceIp restricts calls to a CIDR range — the classic "console access only from the corporate network" requirement. It evaluates the caller's public IP, so requests arriving through a VPC endpoint will not match it.
aws:PrincipalOrgID belongs in resource-based policies: instead of listing every account ID, allow any principal whose account belongs to your AWS Organizations organization. New accounts are covered automatically.
aws:RequestedRegion controls where API calls may land — the building block of the Region-restriction pattern covered later in this lesson.
Permissions boundaries cap the maximum permissions of an individual IAM user or role. The effective permission is the intersection of the identity policy and the boundary; the boundary itself grants nothing. The standard use case is delegation: let a development team create its own roles, but require every role to carry a boundary so the team can never grant beyond it.

Tie it together with the evaluation order: an explicit deny anywhere ends evaluation immediately. Then the gates must all pass — the SCP must allow the action, the permissions boundary (if present) must allow it, and any session policy must allow it. Only then does an allow from an identity-based or resource-based policy take effect. When a question shows a perfectly good IAM allow that still fails, look for an explicit deny or a gate — SCP or boundary — that never allowed the action in the first place.

Walkthrough: triaging an AccessDenied error
An application on EC2 suddenly logs AccessDenied calling s3:PutObject on arn:aws:s3:::app-uploads/reports/daily.csv. Triage it in order and you will locate any IAM failure:

Which principal made the call? Don't assume. Check the instance profile role attached to the instance, or read the userIdentity block of the CloudTrail event. Errors often trace to a different principal than expected — an old role, or a container using the instance role instead of its task role.
Which action on which resource? The exact action and ARN matter: an allow on arn:aws:s3:::app-uploads covers bucket-level actions, but PutObject needs arn:aws:s3:::app-uploads/*. A missing /* on object-level actions is a classic cause.
Does an identity policy allow it? Review the role's attached policies for the action and resource.
Does anything explicitly deny it? Check the bucket policy for denies with conditions — a deny on non-TLS transport or on requests from outside a specific VPC endpoint will reject an otherwise-allowed call.
Do the gates allow it? An SCP on the account or a permissions boundary on the role can silently withhold the allow.
Confirm the diagnosis with the IAM policy simulator before touching production: simulate the exact principal, action, and resource, and it reports allowed or denied — and which statement decided it — without executing anything. Then make the smallest change that fixes the identified layer, and re-run the simulation to verify.

Auditing access: CloudTrail, policy simulator, and IAM Access Analyzer
Auditing answers two different questions: what happened, and what could happen.

CloudTrail answers what happened. Every management API call is recorded with the principal (userIdentity), the action (eventName), the source IP address, and the timestamp. When the exam asks "who deleted the bucket" or "who modified the security group," CloudTrail is the answer. Event history retains 90 days of management events by default; create a trail delivering to S3 for longer retention and multi-account aggregation.

The IAM policy simulator answers what would happen: test a principal against an action and resource without executing anything — ideal for validating a policy change before rollout, or for reproducing an AccessDenied safely.

IAM Access Analyzer answers what could happen from outside. It analyzes resource-based policies and reports external access findings: resources — S3 buckets, IAM role trust policies, KMS keys, and more — accessible from outside your zone of trust (your account or your organization). A finding is not automatically a problem; it is a fact to confirm as intended or to remediate. Access Analyzer also offers unused-access findings, which highlight roles and permissions that have not been exercised — useful for least-privilege cleanup.

Round out the audit kit with two IAM built-ins: the credential report, a downloadable CSV listing every IAM user's password age, access key age, and MFA status, and access advisor, which shows when a principal last used each service — the evidence you need to strip unused permissions with confidence.

Multi-account security: Organizations, SCPs, and IAM Identity Center
AWS Organizations is the multi-account foundation: one management account, member accounts grouped into organizational units (OUs), consolidated billing, and central policy attachment. Structure OUs by function — workloads, sandbox, security — so guardrails can differ per group.

Service control policies (SCPs) are those guardrails, and three rules decide most SCP exam questions:

SCPs never grant permissions. They define the maximum available; a principal still needs an IAM allow. "The SCP allows s3:* but the user still cannot access S3" is normal — the user simply has no identity policy granting it.
An action must be allowed at every level: the SCPs on the account, its OU, and the organization root. A deny anywhere in the chain wins.
SCPs do not affect the management account — one more reason to run no workloads in it.
IAM Identity Center supplies the human access layer on top: workforce users and groups receive permission sets assigned across accounts, provisioned as roles in each one. AWS Control Tower packages the whole pattern — a landing zone with Organizations, Identity Center, and preconfigured guardrails — and you should recognize it by name.

Control	Attaches to	Grants access?	Typical use
IAM identity policy	User, group, or role	Yes — the granting layer	Day-to-day permissions for people and workloads
Service control policy	Organization root, OU, or account	No — caps the maximum	Org-wide guardrails: block Regions, deny risky services
Permissions boundary	Individual IAM user or role	No — caps the maximum	Safe delegation: teams create roles that cannot exceed a limit
Memorize the middle column: only identity and resource policies grant. SCPs and boundaries only cap.

Remediating Trusted Advisor security checks
AWS Trusted Advisor continuously checks your account against best practices, and its security checks are the ones Task 4.1 names. The core security checks — available to every account — include security groups with unrestricted access (0.0.0.0/0 on high-risk ports), IAM use (are you working from the root user instead of IAM principals?), MFA on the root account, and S3 bucket permissions (buckets with open access). Business and Enterprise Support plans unlock the full check set.

Trusted Advisor only flags; remediation is your job, and the exam expects you to match check to fix. An open security group means removing or narrowing the offending inbound rule. A red root-MFA check means enabling an MFA device on the root user. A public S3 bucket usually means turning on Block Public Access and correcting the bucket policy. Old or unused access keys mean rotation or deletion — and preferably replacement with roles.

To make this operational rather than a quarterly chore, wire Trusted Advisor into the event loop: check status changes are emitted as events to EventBridge, where a rule can notify an SNS topic or invoke a Lambda function or Systems Manager Automation runbook that applies the fix. This check → EventBridge → automated remediation pattern is the same loop you will meet again with AWS Config and Security Hub, and recognizing it in any wording is worth easy points on the exam.

Enforcing compliance: Region restrictions and AWS Config
Compliance enforcement has two halves: prevent the violation, and continuously detect the ones that slip through.

Prevention is the SCP's job. The Region-restriction pattern is a deny on all actions when aws:RequestedRegion is not in your approved list — attached at the root or an OU, it stops every principal in every member account, which no per-account IAM policy can do. The standard version uses NotAction to exempt global services such as IAM, CloudFront, and Route 53, whose control planes live in us-east-1 and would otherwise break. Restricting services works the same way: an SCP that denies the unapproved service's actions outright.

Detection is AWS Config. The configuration recorder tracks every supported resource's configuration over time, and Config rules evaluate each resource as compliant or noncompliant. Reach for managed rules — prebuilt checks such as "EBS volumes must be encrypted" or "security groups must not allow unrestricted SSH" — before writing custom rules backed by Lambda. A rule can carry auto-remediation: a Systems Manager Automation runbook that fixes the noncompliant resource, such as revoking an open security group rule, with no human in the loop.

Conformance packs — called out in exam guide v1.1 — bundle Config rules and their remediation actions into one deployable unit, so you can roll an entire compliance framework across accounts and Regions consistently. Config compliance also feeds AWS Security Hub security standards, giving you an account-wide posture score — the continuous-monitoring half of skill 4.1.5.

💡
Tip. Task 4.1 questions are scenario triage: expect an AccessDenied despite a visible allow (find the SCP, permissions boundary, or explicit deny) and multi-account guardrail wording. Map trigger phrases to tools: "prevent any user in any account from using a Region" → SCP with aws:RequestedRegion; "who deleted the bucket" → CloudTrail; "test a policy without executing it" → IAM policy simulator; "bucket accessible outside the organization" → IAM Access Analyzer; "evaluate resources against rules and auto-fix" → AWS Config with SSM remediation. Remember that SCPs never grant and never touch the management account — both are favorite distractors.

Key takeaways
Only IAM identity and resource policies grant access — SCPs and permissions boundaries can only cap it, and an explicit deny anywhere always wins.
Enforce MFA with a Deny policy conditioned on aws:MultiFactorAuthPresent — the account password policy cannot require MFA.
Cross-account access needs an allow on both sides: the caller's identity policy and the target's resource-based policy.
Who did it → CloudTrail. Would this policy allow it → IAM policy simulator. What is exposed outside the account → IAM Access Analyzer.
Block Regions org-wide with an SCP on aws:RequestedRegion (exempting global services via NotAction) — IAM policies cannot reach every account.
SCPs do not affect the management account, and "SCP allows but user still denied" just means no IAM policy grants the action.
AWS Config detects noncompliance and can auto-remediate via SSM Automation runbooks; conformance packs deploy rule sets at scale.
Trusted Advisor flags open security groups, missing root MFA, and public S3 buckets — you remediate, ideally via EventBridge automation.