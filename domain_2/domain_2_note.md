# High Availability, Fault Tolerance, Disaster Recovery

## 2.1 High Availability (HA)

High Availability (HA) ensures a system remains operational and accessible for a very high percentage of time, measured by **uptime**.

Examples:

- **99.9% uptime** → ~8.77 hours of downtime per year
- **99.99% uptime** → ~52.6 minutes of downtime per year

> **Goal:** Minimize downtime.


## 2.2 Fault Tolerance (FT)

Fault Tolerance (FT) allows a system to continue operating **without interruption** even when one or more components fail.

### Example

- **HA:** A server fails, a replacement is launched, the application is restored in **10 minutes**.
  - ✔ High Availability
  - ❌ Not Fault Tolerant (10 minutes of downtime)

- **FT:** Two servers run simultaneously and continuously synchronize data.
  - If one fails, the other immediately takes over (automatic failover).
  - ✔ No downtime
  - ✔ Fault Tolerant

> FT requires redundant components, data replication, and automatic failover, making it more complex and expensive than HA.

## 2.3 Disaster Recovery (DR)

Disaster Recovery (DR) is the process of restoring systems after major disasters such as natural disasters, regional power outages,...

A DR plan typically includes:

- Regular data backups
- Standby infrastructure
- Documented recovery procedures
- Backups stored in different locations (preferably different Regions)

> **Goal:** Restore service as quickly as possible after a disaster.


# Scaling mechanisms in compute environments

## EC2 Auto Scaling Group (ASG)

- **Cooldown:** Default **300s**; applies to **Simple Scaling** only.
- **Health Checks:** `EC2` or `ELB`; **Grace Period = 300s**.
- **Termination Policies:** `OldestLaunchTemplate`, `OldestInstance`, `NewestInstance`, `ClosestToNextInstanceHour`, `Default`, `AllocationStrategy` (Spot).
- **Lifecycle Hooks:** `Pending:Wait`, `Terminating:Wait`; **1h** timeout (extendable to **48h**); timeout action = `CONTINUE` or `ABANDON`.
- **Warm Pools:** Pre-initialized **Stopped** or **Running** instances (outside **InService**) for faster scale-out.
- **Suspend Processes:** `Launch`, `Terminate`, `HealthCheck`, `ReplaceUnhealthy`, `AZRebalance`, `AlarmNotification`, `ScheduledActions`, `AddToLoadBalancer`, `InstanceRefresh`.
- **Instance Refresh:** Rolling replacement after launch template changes; respects **MinHealthyPercentage**.
- **Mixed Instances (Spot):** `LowestPrice`, `CapacityOptimized`, `CapacityOptimizedPrioritized`, `PriceCapacityOptimized` *(recommended)*.

## Lambda Concurrency

Each AWS account has a **default regional concurrency limit of 1,000** shared across all Lambda functions. If the total concurrent executions exceed this limit, additional invocations are **throttled**.

To guarantee capacity for a critical function, configure **Reserved Concurrency**. This reserves a fixed number of concurrent executions exclusively for that function while reducing the shared pool available to other functions.

> AWS requires leaving at least **100 unreserved concurrency** for functions without reserved concurrency.

### Example

- Regional concurrency limit: **1,000**
- Function **A**: Reserved Concurrency = **400**
- Function **B**: Reserved Concurrency = **400**

The remaining **200** concurrent executions are shared by all other Lambda functions. If those functions collectively exceed **200** concurrent executions, **throttling** occurs.

---

# Implement caching by using AWS services to enhance dynamic scalability

## CloudFront

**Default values on a Cache Policy**: Min TTL = 0, Default TTL = 86400 sec (24 hr), Max TTL = 31536000 sec (1 yr) — all editable.


