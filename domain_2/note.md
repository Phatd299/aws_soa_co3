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
