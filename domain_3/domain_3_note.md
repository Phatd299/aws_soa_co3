# Skill 3.1.1: AMIs and Container Images

## AMI Fundamentals

| Component | Detail |
|------------|--------|
| **AMI =** | EBS snapshot(s) + block device mapping + launch permissions + metadata (kernel, architecture, virtualization type) |
| **Instance store-backed AMI** | Root volume from instance store. **Cannot stop** (only reboot or terminate). Cannot create using EBS snapshots. |
| **EBS-backed AMI** | Root volume stored on EBS. Supports **stop/start**, `CreateImage`, and snapshots. Most common AMI type. |
| **Cross-account sharing** | Modify **launch permissions** by adding the target AWS account ID. If the AMI uses **encrypted EBS snapshots** protected by a customer-managed KMS key (CMK), you must also share the **KMS key** or re-encrypt with a key the target account can use. |
| **Cross-region** | AMIs are **Region-scoped**. Use `CopyImage` to copy an AMI to another Region. Encrypted snapshots require a KMS key in the destination Region. |
| **Deregister vs. Delete** | `DeregisterImage` removes the AMI so it can no longer launch instances. **Snapshots remain** until explicitly deleted. This is a common exam trap that leads to orphaned snapshot costs. |

## Core Components

## EC2 Image Builder

| Component | Purpose |
|------------|---------|
| **Purpose** | Automates AMI/container image creation, patching, testing, distribution pipeline — replaces manual "launch, configure, snapshot" workflows |
| **Core Components** | Image recipe (base image + components/build steps), Components (build/test scripts, YAML-based, from Component Manager or custom), Infrastructure configuration (instance type, IAM role, VPC/subnet for build), Distribution configuration (target regions/accounts, launch permissions, tags) |
| **Image Lifecycle** | Build → test (in isolated instance) → distribute (copy to regions/accounts) → optionally deprecate/deregister old versions automatically via lifecycle policies |
| **Output Types** | AMI or container image (pushes to ECR) — same pipeline concept for both |
| **Testing** | Built-in test component phase runs before distribution; failed tests halt distribution |


## Image Lifecycle

```
Build
   │
   ▼
Run Build Components
   │
   ▼
Run Test Components
   │
   ▼
Tests Pass?
   │
 ┌─┴─────────────┐
 │               │
Yes             No
 │               │
 ▼               ▼
Distribute     Stop Pipeline
   │
   ▼
Lifecycle Policy
(deprecate/deregister old images)
```

## Architecture Overview

```text
                    Image Builder Pipeline

             +-------------------------+
             |      Image Recipe       |
             |-------------------------|
             | Base Image              |
             | Components              |
             +------------+------------+
                          |
                          |
             +------------v------------+
             | Infrastructure Config   |
             |-------------------------|
             | EC2 Instance Type       |
             | IAM Role                |
             | VPC/Subnet              |
             +------------+------------+
                          |
                          |
                   Build Temporary EC2
                          |
          +---------------+----------------+
          |                                |
          ▼                                ▼
   Build Components                 Test Components
          |                                |
          +---------------+----------------+
                          |
                    Tests Successful?
                          |
                     Yes / No
                          |
                          ▼
             +-------------------------+
             | Distribution Config     |
             |-------------------------|
             | Regions                 |
             | AWS Accounts            |
             | Launch Permissions      |
             | Tags                    |
             +------------+------------+
                          |
                          ▼
                    Publish AMI
                 or Push Image to ECR
                          |
                          ▼
                 Lifecycle Policies
```


## Skill 3.1.2: AWS CloudFormation and AWS CDK

