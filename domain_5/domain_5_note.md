Domain 5 in full — all three tasks, all twelve skills, exam-depth. This is long; I've kept it dense and table-driven per your standard format. One heads-up before we start: **AWS raised the default VPC route table quota from 50 to 500 routes in June 2025** — if you've seen "50" in older material, that's now outdated. I've used current numbers throughout.

---

# Task 5.1 — Networking Features & Connectivity

## Skill 5.1.1: VPC Configuration

| Component | Key Behavior | Exam-Critical Numbers |
|---|---|---|
| **Subnets** | AZ-scoped, can't span AZs | 5 reserved IPs per subnet (network addr, VPC router at +1, DNS at +2, future-use at +3, broadcast at last — even though VPC doesn't support broadcast) |
| **Route tables** | Most-specific-match wins; local route always present, can't be edited/deleted | **500 routes/table default (raised from 50, June 2025)**; 200 route tables/VPC |
| **NACLs** | Stateless — must allow both directions explicitly; evaluated in rule-number order, lowest first, first match wins; subnet-level | 20 rules/NACL default (adjustable); default NACL allows all; custom NACL denies all by default |
| **Security Groups** | Stateful — return traffic auto-allowed regardless of outbound rules; allow-only (no explicit deny); instance/ENI-level; all rules evaluated (no ordering) | 60 inbound + 60 outbound rules/SG default; 5 SGs/ENI default (adjustable to 16); 1,000 total rules/ENI hard cap |
| **NAT Gateway** | Managed, AZ-scoped (deploy one per AZ for HA), needs EIP, sits in **public** subnet | 5 Gbps baseline, auto-scales to 100 Gbps; 55,000 concurrent connections per unique destination; no SG attachable |
| **NAT Instance** | Legacy, EC2-based, you patch/scale it | Can have a SG; must disable source/dest check |
| **Internet Gateway** | Horizontally scaled, redundant, highly available | 1 IGW per VPC |
| **Egress-only IGW** | IPv6-only equivalent of NAT Gateway (stateful egress, no inbound) | N/A for IPv4 |

**Distractor pair:** *SG vs. NACL.* "Instance can't be reached despite SG allowing the port" → check NACL for the **return traffic** ephemeral port range (1024–65535 on Linux) — a classic SOA-C03 trap because NACLs are stateless and people forget the inbound rule needs a matching outbound ephemeral-port allow (and vice versa).

## Skill 5.1.2: Private Networking Connectivity

| Method | Transitivity | Key Numbers | Distractor Signal |
|---|---|---|---|
| **VPC Peering** | Non-transitive — no routing through a peer to a third VPC | No overlapping CIDR allowed; cross-region and cross-account supported | "Need many-to-many VPC connectivity" → peering doesn't scale (N² connections) → **Transit Gateway** |
| **Transit Gateway** | Hub-and-spoke, transitive across VPC/VPN/DX attachments | 5 TGWs/account; **5,000 attachments/TGW**; 20 route tables/TGW; 50 Gbps burst bandwidth per VPC/DX/peering attachment; MTU 8500 bytes | **TGW peering itself is non-transitive** — routing across a TGW peering connection to a VPC on the far side requires explicit routes on both TGWs, and you can't transit through a peered TGW to reach a *third* TGW |
| **Direct Connect** | N/A (physical link) | Dedicated: 1/10/100/400 Gbps; Hosted (via partner): 50 Mbps–25 Gbps; MACsec available on 10/100/400 Gbps dedicated; DX Gateway lets one DX connection reach VGWs/TGWs across Regions | "Need private, consistent, high-bandwidth connectivity" → DX. "Need it fast, no lead time" → VPN (DX provisioning takes weeks + needs LOA-CFA cross-connect) |
| **Site-to-Site VPN** | N/A | 2 tunnels/connection for HA (~1.25 Gbps each); static or BGP dynamic routing; ECMP across multiple VPN connections (TGW) for more bandwidth (BGP required, not static); VPN CloudHub for hub-spoke across multiple on-prem sites via one VGW | Distractor: VPN as **DX backup** needs BGP path preference (AS-path prepending / local pref) so DX is preferred when up |
| **PrivateLink (Interface Endpoint)** | ENI-based, one endpoint per subnet/AZ | SG applies to the endpoint ENI; private DNS optional (needed if app uses default AWS service DNS name); backed by NLB or GWLB on provider side | "Access an AWS service or third-party SaaS privately without traversing internet/NAT" → Interface Endpoint |
| **Gateway Endpoint** | Route-table based, no ENI | **S3 and DynamoDB only**; free; regional (can't be referenced by a peered VPC's route table) | Distractor: someone asks for a private endpoint to RDS/EC2 API/anything else — must be Interface Endpoint, not Gateway |

## Skill 5.1.3: Auditing Network Protection Services (Single Account)

| Service | Scope | Function | Key Distractor |
|---|---|---|---|
| **Route 53 Resolver DNS Firewall** | VPC-level, outbound DNS only | Domain lists + ALLOW/BLOCK/ALERT rule groups | Stops DNS exfiltration/DGA malware — doesn't inspect payload traffic, only DNS queries |
| **AWS WAF** | Attached to CloudFront, ALB, API Gateway, AppSync, Cognito, App Runner, Verified Access | Web ACLs with rate-based rules, IP sets, managed rule groups, WCU-metered | **CLOUDFRONT-scope Web ACLs must be created in us-east-1**, even though CloudFront is global — a recurring exam trap |
| **AWS Shield Standard** | Automatic, free, all customers | L3/L4 DDoS protection | Always on — no config needed |
| **AWS Shield Advanced** | Paid, opt-in per resource or org-wide | L3/L4/L7 DDoS, cost protection, DDoS Response Team (DRT) access | Distractor: "need cost protection against scaling due to a DDoS attack" → Shield **Advanced**, not Standard |
| **AWS Network Firewall** | VPC-level (via Gateway Load Balancer endpoints or subnet routing) | Stateful/stateless rule groups, Suricata-compatible syntax, domain filtering, TLS inspection | "Need deep packet inspection / IDS-IPS-style filtering inside a VPC" → Network Firewall, not SG/NACL/WAF |

## Skill 5.1.4: Network Architecture Cost Optimization

| Lever | Cost Mechanic | Exam Signal |
|---|---|---|
| NAT Gateway | Hourly charge + **per-GB data processing charge** (this is the expensive part at scale) | High NAT data-processing cost for S3/DynamoDB-bound traffic → replace with **Gateway VPC Endpoint** (free) |
| Interface Endpoint | Hourly + per-GB, but usually cheaper than routing through NAT+IGW for AWS API traffic at scale | Still cheaper than NAT for private API access even though it's not free like Gateway endpoints |
| Data transfer | Same-AZ private IP = free; cross-AZ = charged both directions; cross-region = charged; internet egress = charged; internet ingress = free | "Reduce cost of chatty microservices across AZs" → co-locate in same AZ or accept the cross-AZ cost for HA — a real tradeoff question |
| Direct Connect vs. VPN | DX = flat port-hour + lower per-GB egress rate than internet; VPN = no fixed infra cost but IPsec overhead and lower throughput ceiling | High, steady, predictable data transfer volume → DX pays off despite upfront cost |
| Transit Gateway | Per-attachment hourly + per-GB data processing charge | Many small VPCs paying TGW processing fees → consider fewer attachments/consolidation |

---

# Task 5.2 — Domains, DNS, and Content Delivery

## Skill 5.2.1: DNS / Route 53 Resolver

| Concept | Detail |
|---|---|
| VPC Resolver | Every VPC gets a Resolver at the **base CIDR + 2** address (e.g., `10.0.0.2`) |
| Hard limit | **1,024 packets/sec per ENI to the .2 address — cannot be increased** (a genuinely hard quota, not adjustable via Support) |
| Inbound Resolver Endpoint | Lets **on-prem** DNS servers query your VPC Resolver — needed for on-prem clients to resolve private hosted zone records |
| Outbound Resolver Endpoint | Forwards VPC DNS queries **to on-prem** DNS via Resolver Rules |
| Resolver Rules | **Forward rule** (specific domain → target on-prem IPs); **System rule** (default AWS resolution — can't be overridden for reserved AWS domains) |
| Rule sharing | Share Resolver rules across accounts/VPCs via **AWS RAM** |

**Distractor pair:** "On-prem resource can't resolve a private hosted zone name" → check for an **Inbound Resolver Endpoint**, not an outbound one (people mix these up constantly).

## Skill 5.2.2: Route 53 Routing Policies, Health Checks, Query Logging

| Policy | Behavior | Distractor Signal |
|---|---|---|
| **Simple** | One record (or multiple values returned together, client picks) — no health check support | "Basic DNS, no failover needed" |
| **Weighted** | 0–255 weight, proportional traffic split; weight 0 = stop routing there (unless all weights are 0) | A/B testing, canary traffic shifting |
| **Latency** | Routes to Region with lowest measured latency, not geographic proximity | Distractor: latency ≠ geolocation — a nearby Region isn't always lowest-latency |
| **Failover** | Primary/secondary; **requires a health check on the primary** | "Automatic DR failover to standby Region" |
| **Geolocation** | Routes by user's continent/country/state; requires a default record for unmatched locations; most specific match wins | Compliance/content-licensing routing by country |
| **Geoproximity** | Traffic Flow only; supports **bias** to expand/shrink a region's geographic "pull" | Distractor vs. Geolocation — geoproximity uses distance + bias, geolocation uses fixed geographic rules |
| **Multivalue Answer** | Returns up to 8 healthy records at random; health-check aware | Distractor: **not a substitute for a load balancer** — it's still just DNS-level distribution, no connection draining, no L7 features |

**Health checks:**

| Type | Detail |
|---|---|
| Endpoint (HTTP/HTTPS/TCP) | Default interval 30s, optional **fast interval 10s** (extra charge); default failure threshold 3 consecutive (range 1–10); HTTP(S) requires TCP connect within 4s + 2xx/3xx response within 2s; TCP requires connect within 10s; overall status flips unhealthy if >18% of global checkers report failure |
| Calculated health check | Combines up to 256 child health checks with AND/OR/NOT logic |
| CloudWatch alarm health check | Used for resources with no public endpoint (e.g., RDS) — health tied to an alarm state, set `InsufficientDataHealthStatus` explicitly |

**Query logging:** Public hosted zone query logging **requires the CloudWatch Logs log group to be created in us-east-1**, regardless of where your hosted zone traffic originates — a well-known exam trap.

**Alias vs. CNAME:** Alias records are free, work at the **zone apex** (CNAME can't), have no settable TTL (inherit the target's), and can point to ELB/CloudFront/S3 website endpoints/other R53 records.

## Skill 5.2.3: CloudFront & Global Accelerator

| Feature | CloudFront | Global Accelerator |
|---|---|---|
| Layer | Application (L7) — caching CDN | Network (L4) — TCP/UDP, no caching |
| IPs | Edge domain (dynamic) | **2 static anycast IPs** |
| Use case | HTTP(S) content delivery, caching | Non-HTTP protocols (gaming, VoIP), or when you need static IPs / fast failover without DNS TTL delay |
| Failover speed | DNS-based, subject to TTL/resolver caching | Fast — IP doesn't change, health-check-driven endpoint group failover |

**CloudFront cache TTL defaults** (when origin sends no Cache-Control/Expires headers): min TTL 0, **default TTL 86,400s (24h)**, max TTL 31,536,000s (1 year).

**Origin access:** Use **Origin Access Control (OAC)** for S3 origins — it replaced the legacy OAI and adds SSE-KMS support.

**Distractor pair: CloudFront Functions vs. Lambda@Edge**

| | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Language | JavaScript only | Node.js / Python |
| Trigger points | Viewer request/response only | All 4: viewer request/response, origin request/response |
| Execution location | At the edge (every POP) | Nearest regional edge cache |
| Latency | Sub-millisecond | Higher |
| Use case | Header manipulation, URL rewrites, simple auth | Origin selection logic, external calls, heavier compute |

**Other exam-relevant CloudFront items:** signed URLs (single file) vs. signed cookies (multiple files under one policy); field-level encryption for sensitive form data end-to-end; **Origin Shield** as an additional caching tier that reduces origin load and improves cache hit ratio for multi-region distributions.

---

# Task 5.3 — Troubleshoot Network Connectivity

## Skill 5.3.1: VPC Configuration Troubleshooting

| Symptom | Root Cause to Check |
|---|---|
| Instance reachable on SG but connection fails | NACL missing ephemeral-port return rule (stateless) |
| Public IP assigned, no internet access | Route table missing `0.0.0.0/0 → IGW`, or IGW not attached |
| Private subnet has no outbound internet | NAT Gateway deployed in a **private** subnet (must be public) or route table missing `0.0.0.0/0 → NAT GW` |
| Peering/TGW attachment fails | Overlapping CIDR blocks between VPCs |
| ENI allocation failures (Lambda-in-VPC, ECS tasks) | Subnet CIDR too small — each ENI consumes an IP, high-density workloads exhaust the range fast |
| Traffic silently dropped, no explicit deny anywhere | Missing route entirely — routing is fail-closed by omission, not by explicit deny |

**Tool for this skill:** **VPC Reachability Analyzer** — static, config-based path analysis between two resources (source/destination + protocol/port) without sending live traffic; tells you exactly which component (SG, NACL, route table) is blocking.

## Skill 5.3.2: Networking Logs

| Log Type | Level | Destination(s) | Key Notes |
|---|---|---|---|
| **VPC Flow Logs** | ENI / Subnet / VPC | S3, CloudWatch Logs, Kinesis Data Firehose | Metadata only, not packet content; default fields ~15 (version, account-id, interface-id, srcaddr, dstaddr, ports, protocol, packets, bytes, start, end, **action** ACCEPT/REJECT, log-status); custom format adds vpc-id, tcp-flags, pkt-srcaddr, az-id, etc. **Does NOT capture**: traffic to 169.254.169.254 (metadata), DNS to the Amazon-provided resolver, DHCP, Windows license activation |
| **ELB Access Logs** | ALB/NLB/CLB | S3 only | Disabled by default; delivered roughly every 5 min; requires bucket policy granting the ELB service account PutObject |
| **WAF Logs** | Web ACL | Kinesis Data Firehose, S3, CloudWatch Logs | Full request logs with field redaction option |
| **CloudFront Logs** | Distribution | Standard logs (S3, delayed) vs. **Real-time logs** (Kinesis Data Streams, near-real-time, configurable sampling %, choose specific fields) | Standard is fine for audits; real-time is for live dashboards/alerting |
| **Container Logs** | ECS/EKS | CloudWatch Logs via `awslogs` driver (ECS) or Fluent Bit/Container Insights (EKS) | Container Insights adds performance metrics on top of raw logs |

## Skill 5.3.3: CloudFront Caching Issues

- **Cache hit ratio** improved by: longer TTLs, minimizing what's in the **cache key** (headers/cookies/query strings forwarded), and **Origin Shield** for multi-edge-location scenarios.
- **Cache Policy** controls what's in the cache key + TTL settings; **Origin Request Policy** controls what's forwarded to origin *without* affecting the cache key — a common exam distinction.
- **Invalidations** cost money at scale (first 1,000 paths/month free, then per-path charge) — best practice is **versioned filenames** (cache-busting via new object keys) instead of repeated invalidation for frequently-updated content.
- Debug via response header: `X-Cache: Hit from cloudfront` vs. `Miss from cloudfront`.

## Skill 5.3.4: Hybrid & Private Connectivity Troubleshooting

| Connection Type | Failure Mode | Where to Look |
|---|---|---|
| Direct Connect | BGP session down | VLAN tag mismatch, ASN mismatch, BGP auth key, physical cross-connect (LOA-CFA) issue |
| Direct Connect | Degraded but not down | CloudWatch metrics: `ConnectionState`, `ConnectionBpsIngress/Egress`, `ConnectionErrorCount`; use **DX Resiliency Toolkit** to validate redundancy model |
| Site-to-Site VPN | One tunnel down | Not a full outage (2 tunnels/connection) but redundancy is degraded — check IKE Phase 1/2 parameter mismatch (PSK, encryption, DH group) between both sides |
| Site-to-Site VPN | Static vs. BGP misconfig | Confirm both sides use the same routing mode — mixing static routes with a BGP-configured tunnel breaks route propagation |
| PrivateLink (Interface Endpoint) | Can't reach the service | Endpoint SG doesn't allow the service port; Private DNS not enabled (app resolves the public AWS name instead of the endpoint); endpoint not present in the caller's AZ |
| DX as primary + VPN as backup | Traffic doesn't fail over correctly | BGP path preference not configured (AS-path prepending or local-pref) so DX isn't preferred when healthy |

## Skill 5.3.5: CloudWatch Network Monitoring

| Tool | What It Does |
|---|---|
| VPC Flow Logs + Metric Filters | Derive network metrics from log data (no native per-VPC bandwidth metric exists) |
| NAT Gateway CloudWatch metrics | `BytesOutToDestination`, `PacketsDropCount`, `ErrorPortAllocation` (watch this one — signals port exhaustion under high connection churn) |
| Transit Gateway CloudWatch metrics | `BytesIn/Out`, `PacketDropCount` |
| VPN CloudWatch metrics | `TunnelState` (0/1), `TunnelDataIn/Out` |
| **VPC Reachability Analyzer** | Point-in-time, config-based path check (not live traffic) |
| **VPC Traffic Mirroring** | Copies **actual packets** to an analysis target (Gateway Load Balancer, ENI) — distinct from Flow Logs, which is metadata-only |
| **CloudWatch Network Monitor** | Agentless health probes over hybrid connectivity (DX/VPN paths), alerts on performance degradation without needing an agent on either end |

**Distractor:** "Need to inspect actual packet contents for security analysis" → **Traffic Mirroring**, not Flow Logs (Flow Logs only tells you *that* traffic happened and whether it was accepted/rejected, never *what* was in it).

---

That's full Domain 5 depth across all 12 skills. Want to move into cross-domain scenario practice questions now, or knock out the remaining Domain 4 skills (everything past IAM) first to close that gap before we do integration practice?