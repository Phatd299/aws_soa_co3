# High Availability: ELB and Route 53 Health Checks, Multi-AZ Design


A highly available environment keeps serving users when instances, Availability Zones, or entire Regions fail, by detecting failure with health checks and shifting traffic to healthy capacity automatically. Task 2.2 of the SOA-C03 exam tests exactly that plumbing: choosing between an Application Load Balancer and a Network Load Balancer, configuring target-group health checks and diagnosing why every target went unhealthy after a deploy, using Route 53 endpoint, calculated, and alarm-based health checks to gate DNS answers, and building Multi-AZ fault tolerance with Auto Scaling groups behind a load balancer and RDS Multi-AZ deployments. These are day-one CloudOps scenarios, and the exam probes them with symptoms — 503s from the load balancer, a standby that mysteriously serves no reads, a failover record that never fails over. This lesson covers each layer in turn: the load balancer inside a Region, DNS across Regions, the Multi-AZ patterns underneath, and the single-AZ dependencies that quietly undermine all of it.

What you’ll learn
Choose between an Application Load Balancer and a Network Load Balancer for a given workload
Configure target-group health checks — path, port, protocol, thresholds, interval — and predict how targets change state
Diagnose all-targets-unhealthy conditions and 502/503 errors at the load balancer
Configure Route 53 endpoint, calculated, and alarm-based health checks and active-passive failover routing
Build Multi-AZ fault tolerance with ASGs behind ELB and RDS Multi-AZ, and distinguish Multi-AZ from read replicas
Identify single-AZ dependencies that break static stability
ALB vs NLB: choosing the right load balancer
The ALB-versus-NLB choice comes down to the layer you need to make decisions at, and the exam hands you the deciding phrase in the scenario.

Application Load Balancer	Network Load Balancer
Layer	7 — HTTP/HTTPS	4 — TCP, UDP, TLS
Routing	Path, host, header, and query-string rules to different target groups; redirects and fixed responses	Flows to a target group by listener port — no content inspection
Addressing	DNS name only, IPs change	Static IP per AZ, Elastic IP support
Performance profile	Rich features at HTTP scale	Extreme throughput, very low latency, millions of requests per second
Client IP	Passed in X-Forwarded-For header	Source IP preserved at the connection level
Typical exam cue	"Route /api and /images to different fleets", "inspect headers"	"Static IP per AZ", "UDP", "non-HTTP protocol", "allowlist a fixed IP"
Operationally: pick the ALB whenever the workload is HTTP and you want routing rules, authentication integration, or Lambda targets. Pick the NLB when the protocol isn't HTTP, when clients or firewalls must target a fixed IP address, or when raw connection performance dominates. An NLB can even use an ALB as a target — the pattern for giving an HTTP stack a static IP while keeping layer-7 routing. Both balancers place a node in each enabled Availability Zone, which is what makes them the traffic-steering half of every Multi-AZ design in the rest of this lesson.

Target groups, cross-zone load balancing, and deregistration delay
A target group is the unit you actually operate: a set of targets (instances, IP addresses, or Lambda functions) plus the health-check configuration that decides which of them receive traffic. One load balancer can route to many target groups — an ALB's routing rules map paths or hosts to different groups — and one target group's health-check settings apply to every target in it, which is why a bad health-check setting takes out the whole group at once.

Cross-zone load balancing controls whether a load balancer node in one AZ can send traffic to targets in another. On the ALB it is always on (enabled by default at the load-balancer level), so requests spread evenly across all registered targets regardless of AZ. On the NLB it is off by default and configurable: leave it off and each NLB node only serves targets in its own AZ — so an AZ with fewer healthy targets sees each of them work harder. Enable it when target counts per AZ are uneven, noting that cross-AZ data transfer charges apply for NLB when it's on.

Deregistration delay — connection draining, in older terminology — is what happens when a target is removed or scaled in. Instead of cutting active connections, the target enters a draining state: no new requests arrive, but in-flight requests get up to the configured delay (default 300 seconds) to complete. Deploys that must cycle targets quickly tune this down; long-polling or slow-upload workloads tune it up. If deployments seem to hang "waiting on the load balancer", this timer is usually why.

ELB health checks: the settings that decide who gets traffic
ELB health checks are periodic probes each load balancer node sends to every registered target, and their settings live on the target group. You configure the protocol, port, and (for HTTP/HTTPS) the path — typically a lightweight endpoint like /health that exercises the app without hammering the database. The interval sets how often probes fire, the timeout how long to wait for a response, and two counters govern state changes: the unhealthy threshold is the number of consecutive failures before a healthy target is marked unhealthy, and the healthy threshold is the number of consecutive successes before it's trusted again. For HTTP checks, success means the response code matches the configured matcher — 200 by default — so an app answering with a 302 redirect or a 404 on the check path is "failing" even though it's up.

What happens to an unhealthy target: the load balancer stops routing new requests to it and keeps probing. If it recovers for enough consecutive checks, traffic returns — the load balancer itself never terminates anything. Replacement is the Auto Scaling group's job, and only if the ASG's health check type includes ELB; with the default EC2-only health checks, an instance with a dead app but a live VM stays in the fleet, unhealthy forever.

One nuance worth knowing: if every target in a group is unhealthy, the ALB fails open and routes requests across all of them anyway — which is why total health-check misconfiguration sometimes shows up as errors-plus-traffic rather than silence.

Debugging: all targets unhealthy after a deploy
The scenario: Friday's deploy completes, alarms fire, and the target group shows every target unhealthy while users get errors. All targets failing at once is almost never an instance problem — it's a systemic mismatch between the health check and the application. Work the list:

Path. Did the new version move or drop the health endpoint? A check against /health that now returns 404 marks every target unhealthy. Curl the path from inside the VPC and look at the actual status code.
Port and protocol. If the app now listens on 8080 but the check targets port 80 — or the app redirects HTTP to HTTPS, returning 301 to an HTTP check expecting 200 — every probe fails despite a working app.
Security groups. The instance security group must allow health-check traffic from the load balancer (reference the ALB's security group as the source) on the health-check port. A tightened SG rule takes out the whole group at once.
Non-2xx from the app itself. A deploy-wide bug — bad config, failed dependency at startup — that makes the app answer 500 everywhere is a legitimate all-unhealthy.
Contrast with some targets unhealthy: that's instance-level — one host's crashed process, full disk, or a problem localized to one AZ — so you investigate the specific instance, not the target-group config.

Load-balancer error codes complete the picture: 502 Bad Gateway means the target sent a malformed response or reset the connection (crashed app, wrong port); 503 means no registered or no healthy targets are available; 504 means the target took longer than the idle timeout to respond.

Route 53 health checks: gating DNS answers
Route 53 health checks decide whether DNS gives out an answer at all: attach a health check to a record, and when the check fails, Route 53 stops returning that record's value and answers with the next-best healthy record instead. There are three types, and choosing among them is a standard exam question:

Endpoint health checks probe an IP or domain name over HTTP, HTTPS, or TCP from a fleet of Route 53 checkers distributed around the world. HTTP(S) checks can also do string matching, requiring a specific string in the response body — catching an app that returns 200 with an error page.
Calculated health checks aggregate other health checks: a parent check is healthy based on how many of its child checks are healthy, letting you encode "the site is up if at least two of three services are up".
Alarm-based health checks take their state from a CloudWatch alarm. This is the answer for private resources: Route 53's checkers live outside your VPC, so they can only probe publicly reachable endpoints — a private instance is monitored via its metrics and an alarm instead.
Failover routing is the active-passive pattern: a PRIMARY record with a health check, and a SECONDARY record — often a static site or a standby stack — that Route 53 serves only while the primary is unhealthy. For alias records pointing at AWS resources like an ALB, enable Evaluate Target Health: the alias inherits the health of the resource itself (for an ALB, whether it has healthy targets) without a separate, chargeable health check. A failover record that "never fails over" is usually missing its health check or its Evaluate Target Health flag.

Two layers of health checking: in-Region and cross-Region
Resilient environments run two layers of health checking, and the exam expects you to know which layer solves which failure. Inside a Region, ELB is the layer. The canonical highly available web tier is an Auto Scaling group spanning multiple Availability Zones, registered with a load balancer whose nodes also span those AZs, with the ASG health check type set to ELB. An instance fails → the balancer stops routing to it within a few failed probes → the ASG terminates and replaces it. An AZ fails → the balancer's nodes in surviving AZs keep serving, and the ASG relaunches capacity in the remaining zones. Detection and rerouting happen in seconds, and no DNS changes are involved.

Across Regions (or across independent sites), Route 53 is the layer. DNS failover — a failover record set aliasing each Region's load balancer, with Evaluate Target Health or an endpoint health check on each — shifts users to the secondary Region when the primary stops answering. It is inherently slower: clients and resolvers cache DNS answers for the record's TTL, so keep TTLs low on failover records and never treat DNS failover as instantaneous.

The two layers stack: ELB health checks decide which targets in a Region receive traffic; Route 53 health checks decide which Region receives users. For operations teams that need Regional failover to be deliberate and rehearsed rather than purely automatic, Amazon Application Recovery Controller is the in-scope service to know by name: it provides readiness checks and routing controls for managing Regional failover safely.

Multi-AZ databases: RDS Multi-AZ, readable standbys, and read replicas
An RDS Multi-AZ instance deployment keeps a standby replica in a different Availability Zone, updated by synchronous replication — the primary doesn't acknowledge a write until the standby has it. When the primary instance or its AZ fails (or during certain maintenance operations), RDS fails over automatically: the standby is promoted and the DB endpoint's DNS is updated to point at it, so applications recover by simply reconnecting to the same endpoint. The trap the exam sets repeatedly: the standby is never readable. It serves no queries, adds no capacity, and exists purely for availability.

That is the exact opposite of a read replica, and the contrast is a reliable exam question:

Multi-AZ standby	Read replica
Purpose	Availability	Read scaling
Replication	Synchronous	Asynchronous
Readable	No	Yes
Failover	Automatic, same endpoint	Manual promotion — an ops action
At concept level, the Multi-AZ DB cluster deployment blends the two: one writer plus two readable standbys in separate AZs, giving automatic failover and read capacity in one deployment. If a scenario demands both HA and reads from the standbys, that's the answer.

The same pattern extends to caching: an ElastiCache Redis replication group with Multi-AZ enabled promotes a replica automatically when the primary node fails, so the cache tier — whose sudden loss would dump full load onto the database — gets the same failover treatment as the database itself.

Static stability: automatic failover vs ops action, and single-AZ traps
Static stability means an environment survives an Availability Zone failure without having to change anything or launch anything at the moment of failure — the surviving capacity and plumbing are already in place. Auditing for it starts with knowing what recovers by itself. Automatic: a load balancer stops routing to failed targets and AZs; an Auto Scaling group replaces failed instances and rebalances across surviving zones; RDS Multi-AZ promotes its standby; a Redis replication group with automatic failover promotes a replica. Ops action required: promoting an RDS read replica to primary; repointing a DNS record that has no health check attached; resizing or launching capacity that wasn't pre-provisioned. On the exam, if the recovery path includes a human step, the design is not fault tolerant as described — look for the option that removes the human.

Then hunt single-AZ dependencies, because one shared component can undo an otherwise Multi-AZ design. The classic trap is a single NAT gateway: a NAT gateway lives in one AZ, so if every private subnet's route table points at one gateway and that AZ fails, instances in the surviving AZs lose outbound internet access even though they're healthy. The statically stable pattern is a NAT gateway per AZ, with each AZ's route table pointing at its local gateway — no cross-AZ dependency to fail. The same logic applies at concept level to VPC interface endpoints: place an endpoint network interface in each AZ your workload uses, so AZ loss doesn't sever access to AWS services. Deeper subnet and routing mechanics belong to the networking domain — here, the skill is spotting the shared single-AZ component in an architecture and duplicating it per AZ.

💡
Tip. Task 2.2 stems describe a failure symptom and ask what's wrong or what design prevents it, so learn the trigger phrases: "all targets unhealthy after a deploy" → health-check path/port or a security group blocking probes; "static IP per AZ" → NLB; "standby is never used for reads" → RDS Multi-AZ (and "readable standbys" → Multi-AZ DB cluster); "fail over to a static site when the app is down" → Route 53 failover routing with a health check. Expect at least one question distinguishing what recovers automatically (ELB routing, ASG replacement, RDS Multi-AZ failover) from what needs an operator (promoting a read replica), and one hiding a single-AZ dependency — usually a lone NAT gateway — inside an otherwise Multi-AZ architecture.

Key takeaways
"Static IP per AZ", UDP, or extreme TCP performance → NLB; path, host, or header routing → ALB.
All targets unhealthy is systemic: health-check path or port, a security group blocking probes from the ALB, or an app-wide non-2xx response.
502 = bad response or reset from a target; 503 = no healthy (or no registered) targets; 504 = target timeout.
Route 53 health checkers live outside your VPC — health-check private resources with alarm-based checks.
Failover routing is active-passive: a PRIMARY record with a health check (or Evaluate Target Health on an alias) and a SECONDARY that serves only when the primary fails.
The RDS Multi-AZ standby is never readable — synchronous, automatic-failover availability; read replicas are asynchronous, readable, and promoted manually.
ELB heals within a Region in seconds; Route 53 fails over across Regions at DNS speed, limited by TTL caching.
One NAT gateway shared across AZs is a single-AZ dependency — deploy one per AZ for static stability.