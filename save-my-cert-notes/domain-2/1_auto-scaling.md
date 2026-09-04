# Auto Scaling and Elasticity: EC2, Caching, and Database Scaling

Elasticity is the ability of a workload to add capacity automatically when demand rises and shed it when demand falls, keeping performance steady without paying for idle resources. Task 2.1 of the SOA-C03 exam tests whether you can operate that machinery: EC2 Auto Scaling groups and their five policy types, warm-up and cooldown behavior, the caching layers — CloudFront and ElastiCache — that absorb load before it reaches your compute, and the scaling controls built into RDS, Aurora, and DynamoDB. The questions are operational, not architectural: a group that launches and terminates instances in a loop, capacity stuck at its maximum, a read replica falling behind, a DynamoDB table throttling writes. Each scenario has one best-fit control, and the exam expects you to pick it from a one-line requirement. This lesson works through the scaling mechanisms for the compute, cache, and database tiers, compares every scaling policy type, and walks through the troubleshooting patterns the exam draws its scenarios from.

What you’ll learn
Configure EC2 Auto Scaling groups with launch templates, min/max/desired capacity, and health-check settings
Choose the right scaling policy — target tracking, step, simple, scheduled, or predictive — for a given demand pattern
Diagnose flapping, stuck-at-max capacity, and suspended-process problems in an Auto Scaling group
Apply CloudFront and ElastiCache caching to offload traffic before it reaches the compute tier
Scale RDS, Aurora, and DynamoDB with vertical resizing, storage autoscaling, read replicas, and capacity modes
Interpret ReplicaLag, cache hit ratio, and throttling metrics as scaling signals
How an EC2 Auto Scaling group keeps capacity right
An EC2 Auto Scaling group (ASG) maintains a fleet of instances between three numbers: min, max, and desired capacity. Scaling policies move desired up and down within the min/max bounds, and the group launches or terminates instances until reality matches it. What gets launched is defined by a launch template — AMI, instance type, security groups, key pair, user data — which is versioned, so you can roll a new AMI forward by pointing the group at a new template version and roll back just as easily. Launch templates replaced the older launch configurations, which are immutable and no longer receive new features; new groups should always use templates.

The group also polices health. By default it uses EC2 status checks: an instance that fails hardware or system checks is terminated and replaced. If the group sits behind a load balancer, enable the ELB health check type as well, so an instance whose application is broken — but whose VM is fine — also gets replaced. The health-check grace period gives a newly launched instance time to boot and start its application before health evaluation begins; set it too short and the group kills instances that were still starting up, then launches replacements that suffer the same fate.

Two more behaviors matter operationally. AZ rebalancing keeps instances spread evenly across the group's Availability Zones, launching a replacement in an underweighted AZ before terminating from an overweighted one. Scale-in protection — settable on the group or per instance — shields instances doing long-running work from being selected for termination when the group scales in.

Scaling policy types compared
Five policy types adjust desired capacity, and the exam expects you to pick the best fit from a one-line requirement.

Policy type	How it works	When it's the answer
Target tracking	You set a target value for a metric (for example, average CPU at 50%); AWS creates and manages the CloudWatch alarms and scales both directions to hold the target	"Keep a metric near a value" — the default answer, least configuration
Step scaling	Your CloudWatch alarm triggers adjustments sized to how far the metric breached — small breach adds one instance, large breach adds four	You need different responses at different severity levels
Simple scaling	One alarm, one fixed adjustment, then the whole policy waits out a cooldown before it can act again	Rarely — it's the legacy option, slow to react to fast-moving load
Scheduled scaling	Sets min/desired/max at specific times or on a recurring schedule	Predictable cycles: weekday 9 a.m. ramp-up, overnight scale-down
Predictive scaling	Machine-learning forecast of historical load launches capacity ahead of expected demand	Recurring patterns where instances need lead time to be ready
Target tracking is the modern default. Simple scaling is legacy because it makes exactly one adjustment per cooldown window, so it chronically lags a spike that step scaling or target tracking would meet in one move. Policies can coexist on one group — target tracking for reactive scaling plus scheduled scaling for the known morning ramp is a common, exam-friendly combination. When multiple policies act at once, the group honors the one calling for the most capacity.

Cooldowns, instance warmup, and lifecycle hooks
Cooldowns and warm-up exist because a freshly launched instance contributes nothing for the first minute or two, and scaling decisions made during that window overreact. The classic cooldown (default 300 seconds) applies to simple scaling: after an activity, the policy refuses to act again until the cooldown expires. That is exactly why simple scaling is slow — during a real spike it takes one step, then sits on its hands.

Target tracking and step scaling use instance warmup instead. During warmup, a new instance's metrics are not yet mixed into the group's aggregate, and the instance is already counted toward the capacity the policy thinks it has added — so the group neither scales out again for load the new instance is about to absorb, nor treats the booting instance's near-zero CPU as evidence it should scale in. Set warmup to roughly the time an instance takes to boot and start serving. It's a different knob from the health-check grace period: grace delays health evaluation; warmup delays metric participation.

Lifecycle hooks, at concept level, pause an instance mid-transition: Pending:Wait after launch, so you can run configuration, register with tooling, or pre-warm a local cache before the instance goes into service; Terminating:Wait before termination, so you can drain work and ship logs off the box. Your automation completes the hook (or it times out) and the transition continues. If a group seems to launch instances that hang before entering service, an uncompleted lifecycle hook is a prime suspect.

Troubleshooting: flapping, stuck at max, and suspended processes
Walk through the classic flapping scenario. A group runs a step policy scaling out when average CPU exceeds 70% and a second policy scaling in below 30%. Load rises, two instances launch. While they boot, their near-idle CPUs drag the group average under 30%, so the scale-in policy terminates instances. Load reconcentrates on fewer instances, CPU spikes past 70%, and the cycle repeats — the group launches and terminates every few minutes. The fixes, in exam-preferred order: replace both policies with a single target tracking policy, which manages scale-out and scale-in together and scales in conservatively; lengthen instance warmup so booting instances stop distorting the average; or, if step policies must stay, widen the gap between the out and in thresholds. "Tighten the cooldown" is the distractor — it changes timing, not the feedback loop.

Capacity stuck at max: the alarm stays in breach but nothing launches because desired already equals max. Raise the maximum if the load is legitimate — and check the group's activity history first, because launch failures (an EC2 instance quota reached, an instance type unavailable in an AZ, a deleted launch-template AMI) produce the same "nothing is launching" symptom with a different fix.

Suspended processes: processes such as Launch, Terminate, HealthCheck, and AZRebalance can be suspended — manually for maintenance, or by the group itself after repeated failures. A group that simply does nothing despite breaching alarms may have Launch suspended. The activity history and the suspended-processes list are always your first two stops.

Scaling beyond EC2: ECS services and Lambda
The same scaling vocabulary carries to containers. ECS service auto scaling adjusts a service's desired task count through Application Auto Scaling, and target tracking is again the default answer: hold ECSServiceAverageCPUUtilization or ECSServiceAverageMemoryUtilization at a target, or scale on ALBRequestCountPerTarget when request volume tracks load better than CPU does. Step and scheduled policies exist here too, with the same trade-offs as on EC2. One operational wrinkle: scaling tasks only helps if there is somewhere to place them. On Fargate that's AWS's problem; on EC2-backed clusters, capacity providers with managed scaling grow the underlying Auto Scaling group as tasks demand it — at concept level, know that task scaling and instance scaling are two separate layers that must both work.

Lambda needs only a mention at this depth: it scales concurrency automatically with incoming requests, so there is no policy to configure. The operational knobs are reserved concurrency, which caps a function (and guarantees it that share of the account pool), and provisioned concurrency, which keeps initialized execution environments warm for latency-sensitive workloads. When invocations are throttled, the function has hit its reserved cap or the account concurrency limit — the fix is raising the limit or the reservation, not a scaling policy.

The exam signal to read for: if the scenario says "tasks" or "service", reach for ECS service auto scaling; if it says "function throttled", think concurrency limits, not Auto Scaling groups.

Caching for dynamic scalability: CloudFront and ElastiCache
Caching is a scaling mechanism: every request served from a cache is a request your fleet and database never see. CloudFront in front of a web tier serves static assets and cacheable dynamic responses from edge locations, cutting origin load — often the cheapest fix for a struggling web tier, because raising the cache hit ratio shrinks the fleet the Auto Scaling group must maintain. Operationally, tune cache behaviors and TTLs so dynamic-but-cacheable responses (product pages, API reads) are actually cached, not just images and CSS.

ElastiCache sits between application and database. The Redis-versus-Memcached choice is a recurring exam pattern:

Redis: replication with automatic failover, persistence options, rich data structures. Choose it when the cache must survive node failure or when losing it would crush the database.
Memcached: simple, multi-threaded, scales horizontally by adding nodes — but no replication and no persistence. Choose it for a plain, ephemeral object cache where a lost node just means re-fetching from the database.
Redis cluster mode, at concept level, shards the keyspace across multiple node groups so write throughput and total memory scale horizontally rather than being limited to one primary.

The ops metric is the cache hit ratio (for Redis, the CacheHitRate metric). A falling hit ratio means load is leaking back to the database — check evictions and memory pressure, since an undersized cache evicting keys early is the usual cause. For DynamoDB specifically, DAX plays the same read-caching role without application-managed cache logic.

Scaling RDS and Aurora
RDS scales vertically by modifying the DB instance class. The modification restarts the instance, so applied immediately it means a brief outage; applied in the maintenance window it waits. The Multi-AZ trick, at concept level: with a Multi-AZ deployment, RDS modifies the standby first, fails over to it, then updates the old primary — shrinking the outage to roughly the failover window instead of a full resize-and-restart. Storage autoscaling handles the other vertical axis: when free storage drops below its threshold, RDS grows the volume automatically, up to a maximum you set. It only grows — storage never shrinks back.

For read-heavy load, add read replicas. Replication is asynchronous, so replicas serve eventually consistent reads, and your application must route read traffic to them explicitly. The ops metric is ReplicaLag: rising lag means the replica is falling behind — stale reads now, and a longer catch-up if you ever promote it. Alarm on it.

Aurora restructures the problem. Its cluster storage volume is shared by all instances and grows automatically, so there is no storage provisioning to manage. A cluster supports up to 15 replicas, all reading from that shared volume with typically minimal lag, and the reader endpoint load-balances read connections across them — no application-side replica lists. Aurora Serverless v2 scales an instance's capacity continuously in ACUs (Aurora Capacity Units) between a floor and ceiling you set, making it the answer for spiky or unpredictable database load where a fixed instance class is always either too big or too small.

Scaling DynamoDB: capacity modes and throttling
DynamoDB scales through its two capacity modes, and picking the wrong one is what the exam tests. On-demand mode charges per request and absorbs traffic without any capacity management — the answer for spiky, unpredictable, or brand-new workloads where you can't forecast throughput. Provisioned mode sets read and write capacity units explicitly, and is cheaper for steady, predictable traffic — especially when paired with auto scaling, which is Application Auto Scaling target tracking on consumed capacity: you set a target utilization and floor/ceiling, and DynamoDB adjusts provisioned throughput to follow the load curve.

Throttling is the symptom that tells you the mode or its limits are wrong. When requests exceed provisioned throughput, DynamoDB rejects them with ProvisionedThroughputExceededException, visible in the ThrottledRequests and read/write throttle-event metrics. The diagnosis flows from the traffic shape: a steady workload throttling means provisioned capacity (or the auto scaling ceiling) is simply too low — raise it; a bursty workload throttling between auto scaling adjustments is the cue to switch to on-demand, because auto scaling reacts to consumed-capacity metrics and cannot keep up with instant spikes.

One more pattern at concept level: throttling with plenty of unused table-level capacity points to a hot partition — one partition key receiving a disproportionate share of traffic. Adaptive capacity softens this automatically, but a skewed key design can still overwhelm a single partition; the durable fix is a higher-cardinality partition key, and for hot reads, fronting the table with DAX.

💡
Tip. Task 2.1 questions hand you a demand pattern or a misbehaving fleet and ask for the best-fit control, so map trigger phrases to mechanisms: "keep CPU near 50%" → target tracking; "every weekday at 9 a.m." → scheduled scaling; "launches and terminates repeatedly" → flapping, fixed with target tracking or a longer instance warmup. Database stems lean on symptoms — rising ReplicaLag points at asynchronous read replicas, DynamoDB throttling points at the wrong capacity mode or limits, and spiky database load points at Aurora Serverless v2 or on-demand mode. Watch for distractors that solve availability (Multi-AZ) when the question asks for scalability: read whether the goal is more capacity or more resilience before choosing.

Key takeaways
"Keep a metric near a value" (for example, average CPU at 50%) → target tracking, the default scaling-policy answer.
Simple scaling is legacy: one adjustment per cooldown; prefer target tracking or step scaling for reactive load.
Fix a flapping ASG with target tracking, longer instance warmup, or wider thresholds — not a shorter cooldown.
Alarms in breach but nothing launching → desired equals max, or launches are failing; read the activity history and check quotas.
Redis when the cache needs replication, failover, or persistence; Memcached for simple horizontal ephemeral caching.
RDS read replicas are asynchronous — alarm on ReplicaLag; Aurora runs up to 15 low-lag replicas behind a reader endpoint.
DynamoDB throttling means the wrong capacity mode or limits: on-demand for spiky traffic, provisioned plus auto scaling for steady traffic.
CloudFront and ElastiCache absorb load before it reaches compute — raise the cache hit ratio before resizing the fleet.

