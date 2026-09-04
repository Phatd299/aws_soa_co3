# VPC Networking and Private Connectivity for SOA-C03

Amazon VPC is the software-defined network every AWS workload you operate lives inside: subnets, route tables, gateways, and firewall layers that decide exactly which packets flow where. Task 5.1 of the SOA-C03 exam tests whether you can configure those pieces correctly — public and private subnets, NAT and egress-only internet gateways, security groups versus network ACLs — and then connect networks privately with VPC endpoints, PrivateLink, peering, and Transit Gateway. It also expects you to audit the network protection services running in a single account (AWS WAF, Shield, Network Firewall, Route 53 Resolver DNS Firewall) and to cut network spend, most famously by replacing NAT gateway data-processing charges with free gateway endpoints. This lesson works through each building block from an operations perspective: what to configure, which settings matter, and the trade-offs the exam probes. Troubleshooting broken connectivity belongs to Task 5.3 — here you learn to build it right the first time.

What you’ll learn
Configure subnets, route tables, internet gateways, and NAT so public and private tiers route correctly
Choose between security groups and network ACLs using statefulness, rule ordering, and ephemeral ports
Select the right private connectivity: gateway endpoints, interface endpoints, PrivateLink, peering, or Transit Gateway
Configure Site-to-Site VPN and Client VPN, including tunnels, gateways, and routing choices
Audit AWS WAF, Shield, Network Firewall, and Route 53 Resolver DNS Firewall coverage in a single account
Cut network spend with gateway endpoints, same-AZ placement, and deliberate NAT topology
Subnets, CIDR planning, and the five reserved IPs
A VPC is a private, isolated network you define inside an AWS Region, carved into subnets that each live in exactly one Availability Zone. The VPC's IPv4 CIDR block can range from /16 down to /28, and every subnet takes a slice of it — subnets in the same VPC can never overlap. Plan CIDRs deliberately: a subnet cannot be resized after creation, and if the VPC will ever peer with another network, its range must not overlap that network's range either. At scale, Amazon VPC IP Address Manager (IPAM) exists to plan, track, and allocate CIDR blocks across accounts and Regions — know it by name and purpose.

In every subnet, AWS reserves five IP addresses you cannot assign. In 10.0.0.0/24:

10.0.0.0 — the network address
10.0.0.1 — the VPC router
10.0.0.2 — the Amazon DNS resolver
10.0.0.3 — reserved for future use
10.0.0.255 — the broadcast address (broadcast is not supported, but the address is still held back)
So a /28 subnet yields 11 usable addresses, not 16 — a favorite trap when a question asks how many instances or interfaces fit in a small subnet, and a real operational pain when a load balancer or endpoint quietly consumes several of them. Finally, understand that “public subnet” is not a property you set on the subnet itself. A subnet is public purely because of where its route table points, which is the next section's job.

Route tables, internet gateways, and NAT
A route table decides where each subnet's traffic goes. Every VPC has one main route table; you create custom tables and associate them with specific subnets, and each subnet uses exactly one table (subnets with no explicit association fall back to the main table). Every table carries an unremovable local route covering the VPC CIDR, so intra-VPC traffic always works. When routes overlap, the most specific prefix wins — a /32 beats a /16, which beats 0.0.0.0/0.

An internet gateway (IGW) is a horizontally scaled gateway attached to the VPC. A subnet is public when two things are true: its route table sends 0.0.0.0/0 to the IGW, and instances in it have public IPv4 addresses (auto-assign or Elastic IP). Miss either one and nothing is reachable from the internet.

Private subnets that need outbound internet access — patching, package downloads, external APIs — route 0.0.0.0/0 to a NAT gateway instead. A NAT gateway is a managed, AZ-resident appliance that lives in a public subnet with an Elastic IP. It handles only outbound-initiated traffic; the internet cannot open connections inward through it. Because it is zonal, production designs place one NAT gateway per AZ and give each private subnet a route to the NAT in its own AZ, so an AZ failure cannot strand the surviving AZs. The self-managed alternative, a NAT instance on EC2, still appears as a distractor: workable for tiny workloads, but you patch it, size it, and build its failover story yourself.

IPv6 changes the picture entirely. VPC IPv6 addresses are globally unique and publicly routable, so there is no IPv6 NAT. When instances need outbound-only IPv6, route ::/0 to an egress-only internet gateway — stateful, allows outbound connections and their replies, blocks everything inbound-initiated. “IPv6, outbound only” maps to the egress-only IGW every time.

Security groups vs network ACLs: the classic table
Security groups and network ACLs are the two VPC firewall layers, and the exam relentlessly tests their differences. A security group attaches to an elastic network interface — effectively to an instance, load balancer node, or endpoint — and is stateful: allow a request in, and the response is automatically allowed out regardless of outbound rules. It supports allow rules only; anything not allowed is implicitly denied, and all rules are evaluated as one set. A network ACL attaches to a subnet and is stateless: return traffic must be explicitly allowed. It supports both allow and deny rules, evaluated in rule-number order from the lowest, and the first match wins.

Security group	Network ACL
Attaches to	Network interface (instance level)	Subnet (all traffic in and out)
State	Stateful — replies auto-allowed	Stateless — replies need rules
Rule types	Allow only	Allow and deny
Evaluation	All rules as a set	By rule number, first match wins
Defaults	Deny all in, allow all out	Default NACL allows all; custom NACL denies all until edited
Statelessness creates the classic ephemeral-port question. A client connects to your web server on port 443; the response returns to a high-numbered ephemeral port chosen by the client — commonly within 1024–65535, varying by operating system. A NACL that allows inbound 443 but has no outbound allow for the ephemeral range silently kills every response. Security groups never hit this; statefulness handles it. Operational default: use security groups for workload rules, and reach for NACLs when you need a subnet-wide deny, such as blocking a hostile IP range.

VPC endpoints and PrivateLink
VPC endpoints let resources in private subnets reach AWS services without an internet gateway, NAT device, or public IPs — the traffic never leaves the AWS network. There are two kinds, and choosing between them is a guaranteed exam moment.

Gateway endpoints support exactly two services: Amazon S3 and DynamoDB. You create the endpoint and select which route tables receive it; AWS inserts a managed prefix-list route that steers S3 or DynamoDB traffic through the endpoint. They cost nothing — no hourly fee, no data-processing fee — which makes “private S3 access at no additional cost” an automatic gateway-endpoint answer. They work only from within the VPC that owns them.

Interface endpoints cover most other AWS services. An interface endpoint is a set of elastic network interfaces with private IPs in your subnets, powered by AWS PrivateLink. Enable private DNS and the service's normal public hostname resolves to those private IPs, so applications need no configuration change. Interface endpoints bill per hour per AZ plus per GB — they solve privacy, not cost. Both endpoint types accept an endpoint policy, an IAM-style resource policy that scopes what the endpoint may be used for, such as restricting an S3 gateway endpoint to specific buckets.

PrivateLink also works in the provider direction: exposing your own service privately. You place the service behind a Network Load Balancer and create an endpoint service; consumers create interface endpoints to it from their own VPCs. Because traffic flows through ENIs rather than routes between networks, overlapping CIDR ranges are irrelevant — the standout advantage over peering when consumers' address ranges are outside your control. Connectivity is one-directional: consumers initiate to the provider, never the reverse.

Connecting VPCs: peering vs Transit Gateway
VPC peering creates a private, one-to-one connection between two VPCs — same account or different accounts, same Region or across Regions. Three facts do most of the exam work. First, the CIDR ranges must not overlap; overlapping VPCs simply cannot peer. Second, routing is manual on both sides: each VPC adds a route for the other's CIDR targeting the peering connection, and security groups still apply on top. Third — the classic — peering is never transitive. If VPC A peers with B and B peers with C, A cannot reach C through B; it needs its own direct connection to C. A full mesh of many VPCs grows quadratically, which is why peering fits a handful of VPCs, not a fleet.

AWS Transit Gateway is the hub-and-spoke answer to that mesh problem. Each VPC (or Site-to-Site VPN) becomes an attachment to the hub, and routing between attachments is transitive by default: any spoke reaches any other through the hub with a single attachment each. Operationally, you create the attachment — choosing a subnet per AZ where the Transit Gateway places its interface — then add routes in each VPC's subnet route tables pointing remote CIDRs at the Transit Gateway. The hub has route tables of its own, too: by associating attachments with different Transit Gateway route tables and controlling propagation, you segment traffic — for example, every spoke reaches a shared-services VPC, but spokes cannot reach each other.

Selection cue: two or three VPCs at lowest cost points to peering; “transitive routing,” “hub and spoke,” or dozens of VPCs plus a VPN under one roof points to Transit Gateway.

Hybrid connectivity: Site-to-Site VPN and Client VPN
AWS Site-to-Site VPN extends your on-premises network to AWS through IPsec tunnels over the internet. You configure three resources. A customer gateway represents the on-prem side: your firewall or router and its public IP address. On the AWS side, tunnels terminate on either a virtual private gateway attached to a single VPC, or a Transit Gateway VPN attachment when several VPCs need the same on-premises connectivity. The VPN connection itself always provisions two tunnels, terminating on distinct AWS endpoints for high availability — configure your customer device for both, because a connection built on one tunnel fails every resilience question. Routing is either static, where you enter the on-premises CIDRs yourself, or dynamic over BGP; with a virtual private gateway you can additionally enable route propagation so learned routes appear in your subnet route tables without manual edits.

AWS Client VPN answers a different scenario: individual users rather than a whole site. It is a managed, OpenVPN-based endpoint. You create it with a client CIDR pool (the addresses connecting users receive), associate it with subnets in a VPC to give it network paths, and write authorization rules defining which destination CIDRs users may reach. Authentication is certificate-based (mutual TLS) or user-based through an identity provider. From the associated VPC, connected clients can be routed onward — to peered VPCs or across your Site-to-Site VPN — by adding entries to the Client VPN endpoint's route table.

Cue mapping: “connect the data center or office network to AWS” is Site-to-Site VPN; “remote employees need secure access from their laptops” is Client VPN.

Auditing network protection services in one account
Skill 5.1.3 asks you to audit the network protection services in a single account — know what each protects, at which layer, and where to confirm it is actually attached.

AWS WAF is a layer 7 web application firewall. Web ACLs hold rules — rate limits, IP sets, managed rule groups for SQL injection and cross-site scripting — and attach to specific resource types: CloudFront distributions, Application Load Balancers, and API Gateway APIs. Auditing WAF means listing web ACLs and, crucially, their associations: a web ACL protecting nothing is a common finding. WAF does not attach to instances or Network Load Balancers.

AWS Shield Standard is automatic and free for every AWS customer, absorbing common layer 3 and 4 DDoS attacks such as SYN floods and UDP reflection at the edge — nothing to configure. Shield Advanced is the paid tier: enhanced detection, cost protection against DDoS-driven scaling charges, and access to the Shield Response Team. At audit time, you check which resources are subscribed.

AWS Network Firewall is a managed, stateful network firewall for VPC traffic. You deploy firewall endpoints into dedicated firewall subnets and steer traffic through them with route table entries — for example, routing traffic between the internet gateway and workload subnets through the firewall. It inspects with stateless and stateful rule groups, including domain-based and Suricata-compatible rules. An audit verifies both the firewall policy and the route steering; a firewall no route table points at inspects nothing.

Route 53 Resolver DNS Firewall filters outbound DNS queries from your VPCs against domain lists — block known-bad domains or allow only an approved set. Audit it by checking rule-group associations per VPC; it complements, rather than replaces, the packet-level services above.

Optimizing network costs
Network cost questions on this exam almost always turn on one of three levers: NAT data processing, cross-AZ transfer, and choosing the cheapest private-connectivity construct that meets the requirement.

Walk the classic scenario. An analytics fleet in private subnets uploads hundreds of gigabytes of objects to S3 daily. Its route table sends 0.0.0.0/0 to a NAT gateway, so every byte pays the NAT data-processing rate on top of the gateway's hourly charge. The fix costs nothing: create a gateway endpoint for S3, associate it with the private subnets' route tables, and AWS inserts a managed prefix-list route for S3. That route is more specific than the default route, so S3 traffic immediately diverts through the endpoint — free, and never touching the internet. Do the same for DynamoDB. This is the highest-yield cost answer in Domain 5: “S3 traffic must not traverse the internet, at no additional cost” has exactly one answer.

Second lever: Availability Zones. Traffic between AZs over private IPs is charged in both directions, while same-AZ private-IP traffic is free. Keep chatty tiers — an app fleet and its cache, say — in the same AZ where the availability design allows. And note the NAT topology trade-off: consolidating to a single NAT gateway saves hourly fees but adds cross-AZ data charges from the other AZs plus a single-AZ failure domain. Per-AZ NAT is the resilient default; one shared NAT is the frugal exception for light traffic.

Third, know each option's cost posture: peering has no hourly attachment charge (you pay only data transfer), Transit Gateway adds an hourly fee per attachment plus per-GB processing as the price of transitive convenience, and interface endpoints bill hourly per AZ plus per GB — typically well below NAT processing rates for heavy AWS API traffic.

💡
Tip. This task is tested through configure-it-right scenarios and trigger phrases: “S3 traffic must not traverse the internet, at no cost” means a gateway endpoint; “transitive routing across many VPCs” means Transit Gateway (peering is never transitive); “IPv6 outbound only” means an egress-only internet gateway; “expose a service to consumer VPCs with overlapping CIDRs” means PrivateLink. Security group versus NACL questions probe statefulness and ephemeral ports, and cost questions almost always resolve to eliminating NAT data-processing charges or cross-AZ transfer. Expect troubleshooting variants of the same facts under Task 5.3 — here the exam asks for the correct initial configuration.

Key takeaways
A subnet is public only if its route table sends 0.0.0.0/0 to an internet gateway and its instances have public IPs.
Every subnet loses five IPs: network address, VPC router, DNS resolver, future-use, and broadcast.
IPv6 outbound-only means an egress-only internet gateway — there is no IPv6 NAT.
Security groups are stateful and allow-only; NACLs are stateless, ordered by rule number, can deny — and need ephemeral-port rules.
Private S3 or DynamoDB access at no cost is a gateway endpoint; most other services need an interface endpoint (PrivateLink).
VPC peering is never transitive and requires non-overlapping CIDRs; many-VPC transitive routing is Transit Gateway.
Every Site-to-Site VPN connection provides two tunnels — configure both for high availability.
NAT data-processing charges on S3 traffic are the exam's favorite cost cut: add a gateway endpoint.