### CloudFormation
---

CloudFormation is the AWS-native Infrastructure as Code service. You write a template in JSON or YAML describing the resources you want, and CloudFormation creates, updates, and deletes those resources as a unit called a stack. The fundamental value is repeatability and consistency — the same template deployed in two regions produces identical infrastructure.

Templates have several top-level sections. Parameters allow you to pass values at deployment time — instance type, environment name, VPC ID. You define parameter types (String, Number, AWS-specific types like AWS::EC2::KeyPair::KeyName which validates that the value exists in your account), allowed values, and default values. Mappings are static lookup tables — for example, a map from region to AMI ID so the template automatically uses the right AMI for the deployment region. Conditions evaluate to true or false based on parameter values and control whether resources are created. Outputs export values from the stack — like an ALB DNS name or an RDS endpoint — that other stacks or users can reference.

The Resources section is the only mandatory section. Every resource has a Type (like AWS::EC2::Instance), Properties (the configuration), and optional attributes: DependsOn (explicit ordering), DeletionPolicy, UpdateReplacePolicy, Metadata, and Condition.

DependsOn forces CloudFormation to create one resource before another. CloudFormation already infers dependencies from references — if resource B uses Ref to reference resource A, CloudFormation knows to create A first. DependsOn is for cases where there's a dependency that CloudFormation can't infer, like a database needing to be fully available before an application server starts.

DeletionPolicy is one of the most important and most tested attributes. It controls what happens to a resource when its stack is deleted or the resource is removed from a template. Delete is the default — the resource is deleted. Retain keeps the resource but removes it from CloudFormation management. Snapshot takes a final snapshot before deleting — available for RDS instances, RDS clusters, ElastiCache, Redshift, and EBS volumes. The exam scenario is almost always: "A developer deleted a CloudFormation stack and lost the database data. How do you prevent this?" The answer is DeletionPolicy: Snapshot (or Retain) on the RDS resource.

UpdateReplacePolicy behaves the same way as DeletionPolicy but applies when a resource must be replaced during a stack update rather than deleted entirely. If an update requires CloudFormation to create a new resource and delete the old one (a replacement), UpdateReplacePolicy controls what happens to the old resource.

Intrinsic functions are how you make templates dynamic. Ref returns the value of a parameter or the ID of a resource. Fn::GetAtt returns a specific attribute of a resource — for example, Fn::GetAtt: [MyLoadBalancer, DNSName] gets the DNS name of an ALB. Fn::Sub substitutes variables in a string — useful for constructing ARNs or commands. Fn::ImportValue imports an output exported by another stack. Fn::Select picks an element from a list. Fn::If returns one value or another based on a condition. Fn::Join concatenates strings. Fn::Split splits a string into a list.

Change sets solve the problem of uncertainty during stack updates. Before executing an update, you create a change set — CloudFormation calculates what changes would be made (resources added, modified, replaced, or deleted) and shows you a preview. You review the change set and then either execute it or discard it. The critical detail is resource replacement: some property changes require CloudFormation to delete the old resource and create a new one. For an RDS instance, changing the engine version might trigger replacement, which means downtime and potential data loss without a DeletionPolicy. Change sets surface this before you commit.

Stack drift detection compares the current actual configuration of stack resources against what the template specifies. Resources can drift when someone modifies them directly through the console or API outside of CloudFormation. Drift detection reports DRIFTED (the stack has at least one drifted resource), IN_SYNC (all resources match the template), or NOT_CHECKED (detection hasn't been run). For individual resources it reports MODIFIED, DELETED (resource was deleted outside CloudFormation), IN_SYNC, or NOT_CHECKED. Drift detection is passive — it tells you what drifted but does not fix it. To remediate, you either update the stack to re-apply the template configuration, or import the drifted resource back under CloudFormation management.

Nested stacks embed one stack as a resource within another using the AWS::CloudFormation::Stack resource type. The child stack's template is referenced by URL (S3 location). This lets you modularize large templates — a network stack, a compute stack, a database stack — and compose them into a parent stack. Updates to the parent can cascade to children. The exam pattern for nested stacks is: "Your CloudFormation template is too large" (template size limit is 51,200 bytes when uploaded directly, 1MB when uploaded to S3) or "you want to reuse common infrastructure components across multiple stacks."

StackSets let you deploy the same stack to multiple accounts and regions from a single operation. There are two permission models. Self-managed permissions require you to create IAM roles in each target account. Service-managed permissions integrate with AWS Organizations — you grant CloudFormation permission to operate across the org, and it handles role creation automatically. Service-managed StackSets also support automatic deployment: when a new account joins an OU that has a StackSet targeting it, the stack is automatically deployed to that account. This is the exam pattern for "enforce a security baseline across all accounts in the organization."

StackSet operations have a concurrency model. You specify how many accounts or regions to deploy to simultaneously and how many failures to tolerate before stopping. MaxConcurrentPercentage and FailureTolerancePercentage control this. Low concurrency + low failure tolerance = cautious rollout. High concurrency + high failure tolerance = fast rollout that keeps going even if some accounts fail.

Stack policies are JSON documents that protect specific resources from being updated or deleted. Once a stack policy is set, you cannot update protected resources unless you explicitly override the policy during that update operation. This is different from IAM — stack policies govern what CloudFormation itself can do to resources in the stack.

cfn-init and cfn-signal are helper scripts on EC2 instances. cfn-init reads a Metadata section in the template (AWS::CloudFormation::Init) and configures the instance — installs packages, writes files, starts services. cfn-signal sends a signal back to CloudFormation when the instance has finished initialization. CloudFormation waits at a WaitCondition or CreationPolicy resource until it receives the expected number of signals (or times out). Without cfn-signal, CloudFormation marks the EC2 resource as CREATE_COMPLETE as soon as the instance passes its status checks, even if the application inside hasn't finished installing. This causes the stack to appear complete while the application is still initializing — a common production problem.

The CreationPolicy attribute is the modern replacement for WaitCondition. You attach it directly to an EC2 instance or Auto Scaling group and specify how many success signals you need and the timeout. cfn-signal on the instance sends the signal. If the timeout expires without enough signals, CloudFormation rolls back.

Stack rollback behavior: if any resource fails during CREATE or UPDATE, CloudFormation rolls back all changes by default. This is usually what you want, but during debugging it can be frustrating — the rollback deletes resources that did succeed, making it hard to investigate the failure. You can disable rollback (Rollback on failure: Disabled) during development to preserve the partial stack for inspection. You can also skip specific resources during rollback if you know they didn't cause the problem.

CloudFormation Registry allows you to use third-party and private resource types as first-class CloudFormation resources. Instead of using a Lambda-backed custom resource to manage a Datadog monitor or a Kubernetes namespace, you can register a resource type and use it directly in templates.

Custom resources (AWS::CloudFormation::CustomResource) let you extend CloudFormation with arbitrary logic by invoking a Lambda function or sending a request to an SNS topic. Use cases: provisioning resources that CloudFormation doesn't natively support, calling external APIs during deployment, validating configuration before proceeding. The Lambda function receives a CloudFormation event with RequestType (Create, Update, Delete) and must send a response to a pre-signed S3 URL indicating success or failure.

### Systems Manager
---

Systems Manager is a collection of capabilities under one service umbrella. The unifying theme is managing EC2 instances and on-premises servers at scale without SSH. Everything requires the SSM Agent to be installed and running on the managed instance, and the instance must have an IAM instance profile with the AmazonSSMManagedInstanceCore policy (or equivalent permissions).

Session Manager provides browser-based and CLI-based interactive shell sessions. There are no inbound firewall rules required, no SSH keys to manage, and no bastion hosts. The session traffic flows through the SSM service — the instance makes an outbound connection to the SSM endpoint, and your session rides that connection. Session Manager supports port forwarding, which means you can tunnel RDP, database connections, or any TCP service through a Session Manager session without opening any inbound ports. Sessions can be logged — the complete session transcript goes to S3 and/or CloudWatch Logs for audit purposes. IAM policies control who can start sessions and to which instances.

Run Command executes predefined or custom commands across a fleet of instances. You select targets by instance ID, resource group, or tag-based filter. Commands run asynchronously — you don't hold an interactive session. Output (stdout/stderr) can be sent to S3 or CloudWatch Logs. Rate controls prevent thundering herd: you can specify MaxConcurrency (how many instances run the command simultaneously — absolute number or percentage) and MaxErrors (how many instance failures to tolerate before stopping). Run Command uses SSM Documents (also called SSM Documents or runbooks) — JSON or YAML definitions of actions to perform.

SSM Documents define actions. AWS provides hundreds of pre-built documents (AWS-RunShellScript, AWS-RunPowerShellScript, AWS-ApplyPatchBaseline, etc.). You can also create custom documents. Documents have parameters, steps, and preconditions. Step types include aws:runShellScript, aws:runPowerShellScript, aws:downloadContent, aws:executeStateMachine, and others. Documents are versioned and can be shared across accounts.

Patch Manager automates OS patching. You define patch baselines that specify which patches are approved — by severity, by CVE, by product, or by explicit patch ID. AWS provides default patch baselines for each OS. You create custom baselines to enforce stricter or more lenient rules. Patch groups let you associate instances with a specific baseline using a tag (Key=Patch Group, Value=<group name>). Maintenance windows define when patching runs — you specify a schedule, duration, and allowed unregistered targets. Compliance reporting in Patch Manager (and the broader SSM Compliance feature) shows you which instances are compliant with their patch baseline, which are non-compliant, and what specific patches are missing.

State Manager enforces desired state. An association defines a target (instances by tag or ID), a document (what to run), a schedule (how often to enforce), and parameters. Examples: ensure the CloudWatch Agent is always installed and running, ensure a specific configuration file always has specific content, ensure antivirus is always active. State Manager runs the document on the schedule and reports compliance. This is the mechanism for configuration drift prevention.

Parameter Store is a hierarchical key-value store for configuration data. The hierarchy uses forward slashes: /myapp/prod/db-password, /myapp/dev/db-password. You can apply IAM policies to entire path prefixes — grant an application access to /myapp/prod/* without exposing other paths. Standard tier is free, supports up to 4KB per parameter, and has no parameter policies. Advanced tier costs $0.05 per parameter per month, supports up to 8KB, and supports parameter policies — you can set an expiration date on a parameter and configure EventBridge notifications when it's about to expire or has expired. SecureString parameters are encrypted with KMS. The application must have both IAM permission to get the parameter AND access to the KMS key to decrypt it.

Parameter Store versus Secrets Manager: both store secrets. Secrets Manager adds automatic rotation — it runs a Lambda function on a defined schedule to rotate the secret value (database passwords, API keys). Secrets Manager costs $0.40 per secret per month. Parameter Store SecureString has no built-in rotation. If the exam asks about automatically rotating database credentials, the answer is Secrets Manager with automatic rotation, not Parameter Store.

Secrets Manager integrates natively with RDS, Redshift, and DocumentDB for automatic password rotation. The rotation Lambda updates the secret value and simultaneously updates the database password. Applications retrieve the current secret value from Secrets Manager at runtime rather than having the password hardcoded or in an environment variable.

Inventory collects metadata from managed instances: installed applications, AWS components, network configuration, Windows updates, instance details, and custom inventory you define. Inventory data is stored in SSM and can be queried. You can aggregate inventory across accounts with Resource Data Sync, which ships inventory to a central S3 bucket for analysis with Athena.

OpsCenter is a case management system within SSM. OpsItems represent operational issues — created automatically from EventBridge rules, CloudWatch alarms, or Security Hub findings, or manually. Each OpsItem has a title, severity, source, related resources, and runbooks. You can run SSM Automation documents directly from an OpsItem to remediate the issue.

Automation documents run multi-step operational workflows. Unlike Run Command (which executes a single document on instances), Automation documents can interact with the AWS API — create snapshots, start/stop instances, update AMIs, invoke Lambda functions, call Step Functions. Automation is how you implement self-healing: Config detects a non-compliant resource, triggers an EventBridge rule, which triggers an SSM Automation document that fixes the problem.

Maintenance Windows are schedules for running disruptive tasks — patching, restarts, AMI creation — during a defined window when impact is acceptable. You define the window schedule, duration, and cutoff (how long before the end of the window to stop starting new tasks). Tasks within a window can be Run Command executions, Automation documents, Lambda invocations, or Step Functions executions.

### Elastic Beanstalk
---

Elastic Beanstalk is a PaaS layer on top of EC2, Auto Scaling, ELB, RDS, and CloudWatch. You provide application code; Beanstalk handles the infrastructure. You retain full access to the underlying resources but don't have to manage them directly.

An application is the top-level container. An environment is a running deployment of a specific application version — web server tier (serves HTTP traffic behind an ELB) or worker tier (processes jobs from an SQS queue). You can have multiple environments per application — production, staging, development. Each environment has its own resources.

The saved configuration is a snapshot of environment settings that can be applied to create or update environments. It's how you promote configuration from staging to production — save the staging configuration, apply it to production.

Deployment policies control how a new application version is deployed to the instances in an environment:

- All at Once stops all instances, deploys the new version to all of them, then restarts them. The entire deployment takes the minimum amount of time. There is a brief period where all instances are down — this causes downtime. Only appropriate for development environments where downtime is acceptable.

- Rolling deploys the new version to a subset (batch) of instances at a time. The batch size can be a fixed number or a percentage. While a batch is being updated, those instances are removed from the load balancer — total capacity is reduced by the batch size during the deploy. No extra cost because no new instances are launched. If a deployment fails mid-way, you have instances running the old version and instances running the new version simultaneously.

- Rolling with Additional Batch launches a new batch of instances first before updating any existing instances. During the entire deployment, full capacity is maintained because the extra batch fills the gap. After the new version is fully deployed and healthy, the extra batch is terminated. Slightly more expensive due to the temporary extra instances.

Immutable launches a completely new Auto Scaling group with new instances running the new version. The new ASG sits alongside the existing one. Traffic still goes entirely to the old ASG. Health checks run on the new instances. If they pass, traffic is shifted to the new ASG (by attaching the new instances to the load balancer and the old ASG is removed). If health checks fail, the new ASG is simply terminated — the old ASG has been serving traffic the entire time and is unaffected. This is the safest deployment policy with the easiest rollback (terminate the new ASG). It temporarily doubles the instance count, so it's more expensive during the deployment window.

Traffic Splitting (Canary) sends a configurable percentage of traffic to the new version while the rest continues to the old version. If the new version passes health metrics during the evaluation period, traffic is shifted 100% to it. If it fails, traffic shifts back to 100% old version automatically. This is the only Beanstalk deployment policy that natively implements canary testing.

Blue/Green in Elastic Beanstalk is not a deployment policy in the drop-down menu — it's a manual process. You clone the environment, deploy the new version to the clone (green), test it, then use the Swap Environment URLs feature to atomically swap the CNAMEs of the two environments. Production traffic hits green; the old blue environment is preserved for rollback. Swap Environment URLs takes effect within seconds at the DNS layer, but DNS TTL means some clients will hit blue for a few minutes after the swap.

.ebextensions let you customize the environment via configuration files placed in a .ebextensions directory in your application bundle. Files are processed in alphabetical order. You can install packages, create files, run commands, set environment variables, configure the load balancer, and define CloudWatch alarms. This is how you configure things that the Beanstalk console doesn't expose directly.

Beanstalk stores environment configuration in saved configurations and .ebextensions. The underlying resources (EC2 instances, ELB, ASG) are CloudFormation stacks managed by Beanstalk. You can see these stacks in the CloudFormation console. You should not modify Beanstalk-managed resources directly — Beanstalk will overwrite changes on the next deploy.

RDS in Beanstalk: you can create an RDS instance inside the Beanstalk environment or outside it. Inside is convenient but dangerous — deleting the environment deletes the database. Outside (decoupled) is the production pattern: create RDS separately, provide the connection string as an environment variable to Beanstalk. This way the database lifecycle is independent of the Beanstalk environment lifecycle.

Managed platform updates: Beanstalk can automatically apply platform updates (OS patches, runtime patches) during a maintenance window you define. You control whether to allow minor version updates, patch version updates, or both. Major version updates require manual action.

### CodeDeploy
---

CodeDeploy automates application deployments to EC2, on-premises servers, Lambda functions, and ECS services. It integrates with CodePipeline for CI/CD but can also be triggered directly via the API or CLI.

For EC2 and on-premises deployments, the CodeDeploy agent runs on each instance and polls for deployment instructions. The agent needs outbound internet access (or a VPC endpoint) to reach the CodeDeploy service. The deployment group specifies target instances by tag, Auto Scaling group, or instance ID.

The application revision is the deployable artifact — application code plus an AppSpec file (appspec.yml). The AppSpec file defines the deployment lifecycle and hooks.

For EC2 deployments, the lifecycle event order is:
- ApplicationStop runs on the existing application to gracefully stop it before deploying the new version.
BeforeInstall runs before files are copied — pre-installation tasks like decrypting files, creating backups of the current version.
- Install is when CodeDeploy copies files from the revision to the instance. You don't write a hook for this — CodeDeploy handles it based on the files section of AppSpec.
- AfterInstall runs after files are copied — post-installation configuration, changing file permissions, setting environment variables.
- ApplicationStart starts the new application version.
- ValidateService runs after the application starts — health checks, smoke tests. If this hook exits with a non-zero code, the deployment fails and rolls back.
- BeforeBlockTraffic and AfterBlockTraffic bracket the step where the instance is deregistered from the load balancer before deployment. BeforeAllowTraffic and AfterAllowTraffic bracket re-registration after deployment. These hooks give you control over the exact moment traffic is shifted.

In-place deployment for EC2: CodeDeploy stops the existing application, deploys the new version on the same instances, restarts. During deployment, the instances being updated are deregistered from the load balancer. If you have one instance, there's downtime. If you have multiple instances and configure a deployment configuration that updates one at a time, there's no downtime but reduced capacity.

Blue/Green deployment for EC2: CodeDeploy provisions a new set of instances (either from an ASG launch template or from existing instances you specify), deploys to them, waits for them to pass health checks, then re-routes traffic from the old instances to the new ones by updating the load balancer target group. The old instances remain running until you choose to terminate them (you configure how long to wait before termination — this is your rollback window).

Deployment configurations define how many instances can be updated simultaneously. Built-in configurations: CodeDeployDefault.OneAtATime (one instance at a time — lowest risk, longest deploy), CodeDeployDefault.HalfAtATime (half simultaneously), CodeDeployDefault.AllAtOnce (all simultaneously — fastest, highest risk). You can also create custom configurations specifying a minimum number or percentage of healthy instances during deployment.

For Lambda deployments, the AppSpec specifies the Lambda function name, the current version alias, and the target version. Deployment configurations control traffic shifting:

Canary configurations shift a small percentage of traffic to the new version for a specified time, then shift the remainder. CodeDeployDefault.LambdaCanary10Percent5Minutes shifts 10% for 5 minutes, then 100%.

Linear configurations shift traffic in equal increments at regular intervals. CodeDeployDefault.LambdaLinear10PercentEvery1Minute adds 10% every minute over 10 minutes.

AllAtOnce shifts all traffic immediately. CodeDeployDefault.LambdaAllAtOnce — immediate cutover, no canary period.

Lambda hooks run Lambda functions. BeforeAllowTraffic runs before traffic shifts to the new version — use for pre-flight validation. AfterAllowTraffic runs after the shift — use for smoke tests. If either hook returns a failure, CodeDeploy rolls back to the previous version.

For ECS deployments, CodeDeploy manages the transition from one ECS task definition revision to another using the ALB's weighted target group routing. The deployment group specifies the ECS cluster, service, and load balancer configuration (listener, production target group, test target group). CodeDeploy registers the new task definition revision with the test target group, runs the BeforeInstall hook (Lambda), performs the install (registers with production target group at the configured weight), runs AfterInstall, runs AfterAllowTestTraffic (after test listener sends traffic to new version), shifts production traffic, runs BeforeAllowTraffic, completes the traffic shift, runs AfterAllowTraffic.

ECS deployments only support Blue/Green — there is no in-place ECS deployment in CodeDeploy because ECS manages task replacement internally (ECS rolling update is a separate mechanism, not CodeDeploy).

Automatic rollback can be configured at the deployment group level: roll back on deployment failure (any lifecycle event hook fails or a health check fails) or roll back when alarms are triggered (CloudWatch alarms you specify — if they go to ALARM state during or shortly after deployment, CodeDeploy triggers rollback). For Lambda and ECS, rollback means shifting traffic back to the previous version. For EC2 in-place, rollback means re-deploying the last known good revision.

### EC2 Image Builder and AMI Management
---

An AMI (Amazon Machine Image) is a template for launching EC2 instances. It contains the OS, installed software, configuration, and one or more EBS volume snapshots. Golden AMI is the pattern of pre-building AMIs with all required software, patches, agents, and configuration baked in, so instances launch in a fully configured state with no userdata installation required. This reduces launch latency and eliminates the risk of packages failing to install at launch time.

EC2 Image Builder automates the golden AMI creation process. A pipeline has four stages: source, build, test, and distribute.

The source is a base AMI — either an AWS-provided AMI or a previously built custom AMI.

The build stage applies components — units of build logic. A component is an SSM document that installs software, applies configuration, or runs scripts. AWS provides managed components for common tasks (installing CloudWatch Agent, applying CIS hardening, installing the SSM Agent, enabling FIPS mode). You write custom components in YAML with phases (build, validate, test) and steps.

The test stage launches the built AMI and runs test components against it. Tests can verify that services are running, packages are installed, ports are open, or any custom validation you define. If tests fail, the AMI is not distributed.

The distribute stage copies the tested AMI to specified regions and accounts. You can share AMIs with specific accounts, with an entire Organization, or make them public. Encryption settings are configured per-distribution — you can specify a KMS key for each target region.

Pipelines run on a schedule or are triggered manually. When a new version of the base AMI is available (e.g., AWS releases a new Amazon Linux 2 AMI), Image Builder can automatically start a new pipeline run to rebuild and retest the golden AMI with the latest patches.

AMI sharing across accounts: when you share an AMI encrypted with a customer-managed KMS key, the recipient account must also have access to the KMS key (via the key policy). Otherwise they cannot launch instances from the AMI. If you want to share an encrypted AMI without sharing the key, you copy the AMI to the recipient account with a new KMS key — the copy is encrypted with the recipient's key, so they have full control.

AMI deprecation marks an AMI as deprecated — it no longer appears in default launch wizard searches but can still be used. Existing running instances are unaffected. Deregistration removes the AMI from the launchable list entirely. Deregistration does not delete the underlying EBS snapshots — you must delete those separately. If you want to fully clean up an AMI, deregister it first, then delete the associated snapshots.

AMI lifecycle in the context of ASGs: after building a new golden AMI, you update the launch template (or launch configuration) with the new AMI ID, then trigger an instance refresh to rolling-replace running instances with ones launched from the new AMI. This is the fleet patching pattern — no SSH required, fully automated.

Launch templates are the modern replacement for launch configurations. Key advantages: versioning (you can have multiple versions and specify a default), support for mixed instance types and purchase options (Spot and On-Demand in the same ASG), and support for all current EC2 features. Launch configurations are legacy and do not support these features. The exam favors launch templates.

### Service Catalog
---

Service Catalog lets you create and manage a catalog of approved IT products — infrastructure as code templates — that end users can deploy through a self-service portal without needing CloudFormation expertise or broad IAM permissions.

The administrator side: you create portfolios (collections of products), define products (CloudFormation templates), and share portfolios with IAM principals (users, groups, roles) or with entire OUs via Organizations integration.

The end user side: users browse the catalog, see the products they have access to, choose one, fill in any parameters, and provision it. Under the hood, Service Catalog runs CloudFormation on their behalf. The user never interacts with CloudFormation directly.

The launch constraint is the key exam concept. When you add a launch constraint to a product, you specify an IAM role. When a user provisions that product, Service Catalog assumes the launch role to run CloudFormation. The launch role has the permissions needed to create the product's resources — EC2, RDS, VPC, whatever the template needs. The end user's IAM permissions only need to allow servicecatalog:ProvisionProduct — they don't need ec2:RunInstances, rds:CreateDBInstance, or any of the underlying resource permissions. The launch role provides those. This is how Service Catalog enables least-privilege deployment: users can deploy complex infrastructure with minimal IAM permissions.

Notification constraints let you specify an SNS topic that receives CloudFormation stack events for provisioned products. Useful for operational visibility.

TagOptions library enforces tagging standards. You define TagOptions (valid tag keys and values) and associate them with portfolios or products. When a user provisions a product, they must select from approved tag values. This ensures consistent tagging for cost allocation, security policies, and compliance.

Stack constraints restrict which CloudFormation template parameters an end user can see or modify. You can hide parameters (so the user never sees them, and the administrator-defined default is always used) or lock values (so the user sees the parameter but cannot change it).

Service actions let end users perform operational tasks on provisioned products — like restarting an EC2 instance or updating a parameter — without giving them direct EC2 or SSM permissions. The service action maps to an SSM document that runs with a defined IAM role.

Portfolios can be shared with other accounts or OUs. The receiving account can add their own constraints on top of the shared portfolio's constraints, but cannot remove the original constraints. This supports hub-and-spoke governance: a central team maintains the portfolio, subsidiary teams access it and add their own local controls.

Provisioned products are the running instances of a product — a specific user's deployment of a specific product version. They appear in the Service Catalog console with status (available, under change, error, plan in progress). The user who provisioned a product can update it (deploy a new product version) or terminate it (delete the CloudFormation stack) from the Service Catalog console.

Record history tracks all actions taken on a provisioned product — provisioning, updates, terminations, failed operations. Useful for audit.

The exam pattern to recognize: any question involving self-service infrastructure deployment with governance, approved templates, and least-privilege access → Service Catalog with a launch constraint. The alternative wrong answers usually involve giving users CloudFormation permissions directly (too permissive) or building a custom internal portal (too much operational overhead).