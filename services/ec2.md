## EC2 Esential

## ENI and ENA

ENA stands for Elastic Network Adapter - In AWS, most instance family support ENA - ENA = the networking technology/device at hypervisor hardware.

ENI stands for Elastic Network Interface - This is the network identity attached to the network card.


## Purchase Options

1. On-Demand

2. Reserved Instances (Standard vs. Convertible)

Note:
- User commit to use/pay an instance for long term (1-3 years)

3. Savings Plans

Note:
- User commit to pay x price per hour for long term (1-3 years)
- Apply for other instance service like Lambda, Fargate,...

4. Spot Instances

5. Dedicated Hosts vs. Dedicated Instances

Note:
- Dedicated Host - User control entire physical host/hardware + Used for BYOL, socket control
- Dedicated Instance - Host not shared with anyone else

6. On-Demand Capacity Reservations (ODCR)

Note:
For DR event - User reserve an instance capability for failover.

## EC2 Instance Status

**System status check failure** → underlying hardware/hypervisor/network issue on AWS's side. Remediation: stop/start (moves to new hardware), not reboot.

**Instance status check failure** → OS-level issue (corrupted file system, kernel panic, exhausted memory, misconfigured network). Remediation: reboot, or fix via user data / EC2 Serial Console / EC2Rescue.

Note:
- CloudWatch has a default `StatusCheckFailed_System` and `StatusCheckFailed_Instance` metric

## EC2 Placement Group

**Cluster** — single AZ, low-latency/high-throughput (HPC, tightly coupled MPI workloads). Can't span AZs.

**Spread** — up to 7 instances per AZ, each on distinct underlying hardware — for small numbers of critical instances needing isolation from correlated hardware failure.

**Partition** — up to 7 partitions per AZ, each partition = distinct racks with own network/power; instances in one partition don't share hardware with another. Used for HDFS, Cassandra, Kafka — distributed systems that handle their own replication and want partition-level fault isolation.