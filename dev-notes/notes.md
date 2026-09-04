# AWS CloudOps Engineer - Associate (SOA-C03)

## Identity & Access Management (IAM)

### OIDC and Single Sign-On
- **IAM OIDC Identity Providers**: Connect external OIDC-compatible IdPs to AWS resources
- **IAM Identity Center**: Provides single sign-on (SSO) access for external users to AWS applications
  - Uses the SAML protocol
  - Recommended for external user access to AWS services

### Lambda Concurrency
- **Reserved concurrency**: Maximum number of concurrent Lambda function instances that can run simultaneously
- **Provisioned concurrency**: Number of pre-initialized execution environments
  - Pre-initialized code reduces cold start latency
  - Speeds up Lambda function execution

## Networking & DNS

### VPC Endpoints
- **Gateway VPC Endpoints** (S3 and DynamoDB):
  - Allow EC2 instances in private subnets to access S3 without internet
  - No data processing charges
  - No need for NAT gateways

### Route 53

#### DNS Record Types
- **A record**: Maps domain to IPv4 address
- **AAAA record**: Maps domain to IPv6 address
- **Alias record**: Routes to zone apex or ALB DNS
  - Can point a subdomain (www.example.com) to root domain (example.com)
  - Enables routing to different service providers

#### Routing Policies
- **Geolocation**: Route traffic based on user location
- **Geoproximity**: Route traffic based on resource location
- **Latency-based**: Route to lowest latency resources
- **Multivalue answer**: Return multiple IP addresses

### Route 53 Resolver
- **Inbound Endpoints**: Allow DNS queries from on-premises network or another VPC to your VPC
- **Outbound Endpoints**: Allow DNS queries from your VPC to on-premises network or another VPC

### Email Delivery
- **SPF Records**: TXT record specifying which IP addresses can send email for a domain
  - Prevents email spoofing
  - Improves email deliverability

## Compute

### EC2 Instance Placement Groups
- **Cluster Placement Group**:
  - Packs instances close together in an Availability Zone
  - Low-latency network performance
  - Ideal for high-performance computing (HPC) and tightly-coupled applications

- **Partition Placement Group**:
  - Spreads instances across logical partitions
  - Prevents groups from sharing underlying hardware
  - Used for distributed and replicated workloads (Hadoop, Cassandra, Kafka)

- **Spread Placement Group**:
  - Places small group of instances across distinct hardware
  - Reduces correlated failures

- **Reference**: [EC2 Placement Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html)

### EC2 Instance States
- **Stop**: Shuts down the instance
  - Migrated to new underlying host on restart
  - Assigned new public IPv4 address

- **Start**: Powers on the instance and migrates to new host
  - [Reference](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Stop_Start.html)

- **Reboot**: OS-level reboot (typically a few minutes)
  - Keeps: Public DNS name, Private IPv4, Public IPv4, IPv6, instance store data

- **Terminate**: Permanently shut down the instance
  - Cannot be recovered
  - Attached EBS volumes marked for deletion are permanently deleted

## Storage

### Elastic File System (EFS)
- Can create only one mount target per Availability Zone
- [Reference](https://docs.aws.amazon.com/efs/latest/ug/accessing-fs.html)

### Elastic Block Store (EBS)
- **Snapshots**:
  - Incremental backups (only changed blocks saved)
  - Minimizes creation time and storage costs
  
- **Fast Snapshot Restore (FSR)**:
  - Creates volume fully initialized at creation
  - Eliminates I/O latency on first access
  - Delivers full provisioned performance instantly

### Storage Gateway
- **Gateway-stored volumes**:
  - Full data copy stored locally
  - Asynchronously backed up to AWS
  - Behaves like local block storage device

- **Gateway-cached volumes**:
  - Most data stored in AWS
  - Recently accessed data cached locally

## Database Services

### RDS (Relational Database Service)
- **RDS Proxy**:
  - Pools and shares database connections
  - Improves connection efficiency
  - [Reference](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)

- **Performance Insights**:
  - Visualize database load on RDS instances
  - Filter by waits, SQL statements, hosts, or users
  - [Reference](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html)

### Aurora
- **PITR (Point-in-Time Recovery)**:
  - Robust, long-term backup and restore solution
  - Creates new database cluster

- **Backtracking**:
  - Quick, in-place "undo" feature for Aurora MySQL
  - Recovers from recent, minor errors
  - Low Recovery Time Objective (RTO)

## Caching & Performance

### ElastiCache
- **Memcached**:
  - Lightweight and simple
  - Best for read-heavy, non-persistent caching

- **Redis**:
  - Feature-rich with high availability
  - Supports persistence and advanced data types
  - Ideal for mission-critical caching and real-time applications

## Infrastructure as Code

### CloudFormation
- **Stack Sets**: Create stacks across AWS accounts and regions with single template

- **Custom Resources**:
  - Provisions requirements with complex logic or workflows
  - Use when built-in resource types are insufficient
  - [Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)

## Monitoring & Logging

### CloudWatch
- **Logs Data Protection**:
  - Identifies and protects sensitive data (PII, PHI)
  - Automatically detects sensitive data patterns
  - Invokes alerts or initiates actions when sensitive data is logged

- **CloudWatch Synthetics Canaries**:
  - Configurable scripts that run on schedule
  - Monitor endpoints and APIs
  - [Reference](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html)

- **Process Metrics**:
  - Collect process metrics using procstat plugin

### Cost Allocation
- **User-defined Cost Allocation Tags**:
  - Must be activated to track costs
  - [Reference](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/activating-tags.html)

## Deployment Strategies

### Rolling Updates
- Gradually replaces tasks with new versions
- Maintains application availability during updates
- Updates in small batches

### Canary Deployments
- Shifts small percentage of traffic for validation
- Requires additional infrastructure for traffic splitting
- Leads to higher costs

## Security & Compliance

### Trusted Advisor
- Inspects AWS environment
- Makes recommendations for:
  - Cost savings
  - System availability and performance improvements
  - Security gap closure
- Available checks depend on support plan level

## Governance & Management

### AWS Control Tower
- **Automated landing zone setup**: Well-architected multi-account environment with log archive and audit accounts
- **Pre-configured controls**: Library of governance rules (guardrails)
  - Preventive, detective, or proactive controls
  - Enforce security, compliance, and operational policies
- **Account Factory**: Provisions new AWS accounts with automatic policy compliance

### AWS Service Catalog
- **Portfolio Sharing**:
  - Share via account-to-account or Organizations
  - Shares reference to portfolio (not copy)
  - Products and constraints stay in sync with shared portfolio
  - Recipients cannot modify products/constraints but can add IAM access
  - [Reference](https://docs.aws.amazon.com/servicecatalog/latest/adminguide/catalogs_portfolios_sharing_how-to-share.html)

### AWS Personal Health Dashboard
- Personalized view of AWS service events that may affect your resources

### OpsWorks
- Supports Chef and Puppet configuration management