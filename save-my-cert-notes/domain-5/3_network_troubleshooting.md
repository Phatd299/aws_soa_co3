AWS Network Troubleshooting: VPC Flow Logs and the Layered Diagnosis Method
15 min read
SOA-C03 · Networking and Content Delivery
Updated
18 Jul 2026
By the SaveMyCert Editorial Team
Mapped to “Networking and Content Delivery” in the official SOA-C03 exam guide
Troubleshooting AWS network connectivity means working a fixed, layered checklist — route tables, security groups, network ACLs, gateway and endpoint state, then DNS — until you find the control that is dropping or misdirecting packets. Domain 5 of the SOA-C03 exam tests exactly this discipline: given a symptom like "the instance can't reach the internet" or "the tunnel is up but nothing flows", you must name the misconfigured component and its fix. This lesson teaches the method, then applies it to the classic failures — NAT gateway placement, ephemeral-port NACL traps, one-sided peering routes, Transit Gateway route tables — and shows you how to read the evidence: VPC Flow Log signatures, ELB access log fields, WAF and CloudFront logs, and the CloudWatch network monitoring services added in the v1.1 guide. Every scenario here maps to a question pattern the exam actually uses, so learn the signatures, not just the service names.

On this page
8 sections
What you’ll learn
Apply the layered checklist — route table, security group, NACL, gateway state, DNS — to any VPC connectivity failure.
Diagnose stateless NACL failures from paired ACCEPT/REJECT records in VPC Flow Logs.
Trace missing paths across peering connections and Transit Gateway route tables, and know when to run Reachability Analyzer.
Isolate faults between load balancer, WAF, CloudFront, and application tiers using each service's logs.
Remediate low CloudFront cache hit ratios by tightening cache keys, aligning TTLs, and versioning object names.
Troubleshoot Site-to-Site VPN, VPC endpoint, and PrivateLink failures, and monitor hybrid paths with CloudWatch.
The layered checklist: how to diagnose any VPC connectivity failure
Every VPC connectivity failure lives in one of five control layers, and checking them in a fixed order beats jumping around the console. Work through: route tables (does the source subnet have a route to the destination — and does the destination have a route back?), security groups, network ACLs, gateway and endpoint state (is the NAT gateway, peering connection, transit gateway attachment, or VPC endpoint actually present, up, and in the right place?), and finally DNS (is the client even resolving the name to the IP address you think it is?).

The distinction that powers the whole method: security groups are stateful — if a request is allowed in, the reply is automatically allowed out, regardless of outbound rules. Network ACLs are stateless — the reply is a brand-new packet that must match an explicit allow rule in the opposite direction, typically on the ephemeral port range 1024-65535.

That leads to return-path thinking: a TCP connection only works if the request is permitted by every control on the way there and the reply is permitted by every stateless control on the way back. When you audit a path, trace it twice — forward against the destination port, and backward against the ephemeral source port. Asymmetric failures, where the SYN arrives but the reply dies, look like timeouts to the application but are unmistakable in flow logs.

Only after the five layers check out do you blame the operating system firewall or the application itself — which AWS network controls cannot see, and which VPC Reachability Analyzer will not evaluate either.

No internet access: internet gateway, public IP, and NAT gateway placement
When an instance cannot reach the internet, the cause is almost always one of four misconfigurations, split by subnet type.

Public subnet
No IGW route: the subnet's route table must contain 0.0.0.0/0 pointing at the internet gateway. A subnet is only "public" because of this route.
No public IP: the instance needs a public IPv4 address or an Elastic IP. A correct route with no public address still fails, because the internet gateway performs one-to-one NAT against that address.
Private subnet
Missing NAT route: the private subnet's route table needs 0.0.0.0/0 pointing at the NAT gateway.
NAT gateway in the wrong subnet: the NAT gateway itself must sit in a public subnet whose route table points at the internet gateway. A NAT gateway placed in a private subnet has no path out, so everything behind it silently fails.
Availability Zone placement matters operationally: a NAT gateway is zonal. If private subnets in AZ B route through a NAT gateway in AZ A, an AZ A failure removes egress for AZ B too, and you pay cross-AZ data charges in the meantime. The resilient pattern is one NAT gateway per AZ, each serving its own AZ's subnets.

Two CloudWatch NAT gateway metrics diagnose problems at scale: ErrorPortAllocation counts failed port allocations — port exhaustion from too many concurrent connections to the same destination, remediated by adding NAT gateways or splitting traffic across subnets — and PacketsDropCount flags packets the gateway dropped. For IPv6, the egress-only internet gateway fills the NAT role: outbound-initiated traffic flows, inbound-initiated connections are blocked.

Two instances can't talk: security group references and the ephemeral-port trap
When two instances in the same VPC cannot communicate, start with security groups — specifically with what the rules reference. A classic failure: the database security group allows MySQL from the "app" security group, but the instance actually connecting was launched with a different group, or the rule references the load balancer's group while the app tier connects to the database directly. Confirm the source's actual group membership, not what the architecture diagram says.

If security groups check out, walk the network ACL scenario, because it produces the exam's favorite trap. An app server at 10.0.1.25 opens a connection to a database at 10.0.2.40 on port 3306. The connection times out. Flow logs on the database's ENI show a pair of records:

2 123456789012 eni-0a1b2c3d 10.0.1.25 10.0.2.40 48512 3306 6 4 216 1719923400 1719923460 ACCEPT OK
2 123456789012 eni-0a1b2c3d 10.0.2.40 10.0.1.25 3306 48512 6 4 188 1719923400 1719923460 REJECT OK
Read the signature: the request was ACCEPTed and the reply was REJECTed on the same ENI. A security group can never produce this — it is stateful, so an accepted request implies an allowed reply. The only control that can accept a request and reject its reply is the stateless network ACL. Here, the database subnet's NACL has no outbound allow for ephemeral ports; the fix is an outbound rule allowing TCP 1024-65535 to the app subnet's CIDR.

Generalize it: between any two hosts, every NACL on the path needs allows in both directions covering both the destination port and the ephemeral return ports — four rule evaluations per NACL, not one.

Peering and Transit Gateway: when the path itself is missing
Peering failures are usually route failures. VPC peering is non-transitive: if VPC A peers with B and B peers with C, A cannot reach C through B, no matter what routes you write. Expecting transitivity is a design error whose fix is a transit gateway, not more routes.

Within a working peering connection, routes must exist on both sides: VPC A's route tables need the B CIDR pointing at the pcx- peering connection, and VPC B's route tables need the A CIDR pointing at the same connection. Miss one side and you get an asymmetric black hole — SYNs arrive but replies have nowhere to go, and the client sees timeouts. Overlapping CIDRs are a harder stop: VPCs with overlapping ranges cannot establish peering at all, which is why address-planning mistakes resurface as "connectivity" tickets months later.

Transit Gateway adds a second routing layer, and both must be right:

The VPC route table needs a route for the remote CIDR pointing at the transit gateway, and the attachment for that VPC must exist.
The transit gateway route table associated with the source attachment needs a route to the destination attachment — and the destination VPC needs the reverse pair.
CloudWatch narrows the failing layer: transit gateway metrics PacketsDropCountNoRoute (no matching route in the TGW route table) and PacketsDropCountBlackhole (a route exists but is a blackhole) prove the drop happened at the transit gateway, not inside a VPC.

For multi-hop paths like these, VPC Reachability Analyzer is the fastest tool: it performs static configuration analysis between a source and destination and names the exact blocking component — the specific security group, NACL, or route table — without sending a single packet. Reach for it when the configuration spans many hops; use flow logs when you need to see what live traffic actually did. It does not evaluate OS firewalls or whether the application is listening.

Reading VPC Flow Logs: the fields and the three diagnostic signatures
VPC Flow Logs record metadata about IP traffic at three capture levels — an entire VPC, a subnet, or a single ENI — and publish to CloudWatch Logs or Amazon S3. The default record format is a space-separated line: version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status. For troubleshooting you mostly read four of them: srcaddr and dstaddr, the ports, and action (ACCEPT or REJECT).

Three signatures answer most connectivity questions:

Signature	What it means	Where to fix
ACCEPT request, then REJECT reply on the same ENI	A stateless NACL is blocking return traffic — a stateful security group cannot reject the reply of a flow it accepted	NACL outbound ephemeral-port rules (1024-65535)
REJECT on the inbound request	The security group or NACL is blocking the request itself	Security group inbound rules first, then the subnet NACL
No flow record at all	Packets never reached the ENI — the problem sits upstream of security rules	Route tables, DNS resolution (is the client connecting to the IP you expect?), source-side controls
The "no record" case has one caveat: flow logs deliberately exclude some traffic, including queries to the Amazon DNS resolver, instance metadata requests to 169.254.169.254, and DHCP. Absent DNS traffic is normal; an absent application flow is a finding. Also glance at log-status: NODATA means no traffic was seen in the aggregation window, while SKIPDATA means records were dropped — so your absence of evidence may not be evidence of absence.

Which log answers which question: ELB, WAF, CloudFront, and container logs
Layered architectures need layered logs. Each log type isolates fault to one tier, and the skill the exam tests is picking the one that answers your actual question.

Question	Log	Key fields
Was traffic blocked before reaching the instance?	VPC Flow Logs	action, ports
Did the load balancer or the target fail?	ELB access logs	elb_status_code vs target_status_code, target_processing_time, the target that served the request
Which rule blocked this request?	AWS WAF web ACL logs	Terminating rule ID, rule action
Did the edge serve cache or hit the origin?	CloudFront standard logs	x-edge-result-type, sc-status
What did the application itself do?	Container logs (ECS/EKS to CloudWatch Logs)	Application output
ELB access logs are the load-balancer-versus-target separator. A 504 with a target_processing_time of -1 means the target never responded before the timeout — a backend problem. A 502 points at a broken or malformed target response or a failed connection to the target. When elb_status_code is a 5xx but target_status_code shows the target returned 200, the fault is in the load balancer tier, not the application. The log also records which target served each request, so you can spot one bad instance hiding behind an otherwise healthy fleet — then hand the health-check tuning to your reliability playbook.

AWS WAF web ACL logs are for false-positive hunting: when legitimate users report being blocked, filter for block actions and read the terminating rule ID to see exactly which rule fired, then tune that rule instead of guessing. Container logs matter as the last tier: once every network layer shows clean, the answer is in the application's own output.

CloudFront caching issues: diagnosing a low cache hit ratio
A falling CloudFront cache hit ratio has three usual causes: a cache key that is too broad, TTLs that are effectively zero, and unnecessary invalidations. Diagnose before you tune.

Walkthrough. Origin load doubles and the distribution's cache hit rate reads 35%. Per request, the x-cache response header shows "Miss from cloudfront" even on repeated fetches of the same page. In the standard logs, x-edge-result-type is dominated by Miss — not Error — so the origin is healthy and this is purely a caching problem. The cache policy turns out to forward all query strings, and the site's marketing links append unique tracking parameters, so every visitor generates a unique cache key and nothing is ever reused. The application only varies its response on lang. Restricting the cache key to that one query string collapses thousands of one-off cache entries into a handful, and the hit ratio recovers.

The general rules:

Cache key too broad: forwarding all headers, cookies, or query strings makes every variation a separate cached object. Include only the values the origin actually uses to vary the response.
TTL zero: if the origin sends Cache-Control: no-cache, private, or max-age=0 and the minimum TTL is 0, CloudFront revalidates or refetches everything. Align origin headers and cache policy TTLs deliberately.
Invalidations: invalidating on every deploy empties the cache repeatedly. Prefer versioned object names (app.v42.js) so new content gets a new URL and old entries simply expire.
Distinguish misses from errors: an x-edge-result-type of Error means the origin returned a 5xx or timed out — a reliability problem, mitigated by an origin group with origin failover, not by cache tuning.

Hybrid and private connectivity: VPN, endpoints, and CloudWatch network monitoring
Site-to-Site VPN problems split into two states. Tunnel down (the TunnelState metric reads 0): IKE negotiation is failing — a mismatched pre-shared key, a wrong customer gateway IP or device configuration, or an on-premises firewall blocking UDP 500/4500. Tunnel up but no traffic: the tunnel is fine and the routing is not. With static routing, the on-premises CIDRs must be entered on the VPN connection and routed in the VPC route tables; with BGP, route propagation must be enabled on the route table. And security groups still apply to VPN traffic — instances must explicitly allow the on-premises CIDR, which is easy to forget because the traffic feels internal. (Direct Connect raises the same route-and-return questions but sits outside this exam's scope — think VPN and Transit Gateway.)

Private connectivity failures follow the endpoint type:

Interface endpoint resolves to a public IP: private DNS is not enabled on the endpoint, so the service's public hostname bypasses it — and from a private subnet the connection then dies. Enable private DNS, with DNS support enabled on the VPC.
Connection to an interface endpoint times out: the endpoint's security group must allow inbound 443 from the consumers.
Explicit denies: the endpoint policy can reject specific actions or principals even when the network path is clean.
Gateway endpoints (S3, DynamoDB) work through the route table: a subnet whose route table is not associated with the endpoint sends traffic out the NAT path instead — check associations before anything else.
Cross-account PrivateLink: the endpoint sits in a pending-acceptance state until the service owner accepts it; nothing flows before that.
For continuous visibility, CloudWatch Network Monitor runs synthetic probes from your VPC toward on-premises IPs and emits packet-loss and latency metrics for hybrid paths, so degradation surfaces before users report it. Internet Monitor covers the internet side, reporting performance and availability between your workloads and end-user city-networks.

💡
Tip. Expect scenario questions that hand you a symptom and demand the one misconfigured component. Learn the trigger phrases: "ACCEPT inbound but REJECT on the return traffic" points at a stateless network ACL missing ephemeral-port rules, "identify which component is blocking the path" means VPC Reachability Analyzer, "cache hit ratio dropped" means the cache key is too broad or TTLs are zero, "resolves to a public IP despite an interface endpoint" means private DNS is disabled, and "VPN tunnel is up but no traffic flows" means routes (static entries or BGP propagation) or security groups. When no flow log record exists at all, the answer is routing or DNS — the packets never reached the ENI.

Key takeaways
Security groups are stateful; network ACLs are stateless — a rejected reply to an accepted request is always a NACL problem.
ACCEPT then REJECT for the same flow in VPC Flow Logs means a NACL is blocking the return on ephemeral ports 1024-65535.
No flow-log record at all means packets never reached the ENI — suspect routing or DNS, not security rules.
A NAT gateway must sit in a public subnet with an IGW route; the private subnet routes 0.0.0.0/0 to the NAT gateway.
VPC peering is non-transitive and needs routes on both sides; Transit Gateway needs routes in both the VPC and TGW route tables.
VPC Reachability Analyzer names the exact blocking component through static analysis, without sending a packet.
Low CloudFront hit ratio: shrink the cache key, raise TTLs, and version object names instead of invalidating.
VPN tunnel up but no traffic: check static routes or BGP propagation first, then security groups for the on-premises CIDR.