## VPC Fundamentals

A VPC is a logically isolated network in AWS. When you create a VPC, you define a CIDR block — the range of IP addresses available within it. The allowed range is /16 (65,536 addresses) to /28 (16 addresses). You can add secondary CIDR blocks to a VPC after creation, which is useful when you run out of address space without having to recreate the VPC.

Subnets divide the VPC CIDR into smaller ranges. Each subnet lives in exactly one Availability Zone. AWS reserves five IP addresses in every subnet: the network address (first), the VPC router (second), DNS (third), future use (fourth), and the broadcast address (last). A /24 subnet gives you 256 addresses minus 5 reserved equals 251 usable. This reservation detail shows up in exam questions about "why can't I launch 256 instances in a /24 subnet."

A subnet is public if and only if its route table has a route to an Internet Gateway. There is nothing inherently public or private about a subnet — it is entirely determined by the route table. An Internet Gateway is a horizontally scaled, redundant, highly available VPC component. There is one IGW per VPC (you cannot attach multiple). The IGW also performs NAT for instances with public IP addresses — it translates between the public IP and the private IP when traffic enters and leaves the VPC.

A subnet is private when its route table has no route to an IGW. Private subnets can still reach the internet through a NAT Gateway. The NAT Gateway must be in a public subnet (it needs the IGW route to forward traffic to the internet), and the private subnet's route table must have a route sending internet-bound traffic (0.0.0.0/0) to the NAT Gateway.

NAT Gateway is AZ-scoped. A single NAT Gateway in us-east-1a handles outbound internet traffic for private subnets that route to it. If us-east-1a fails, all private subnets routing through that NAT Gateway lose internet access. For high availability, deploy one NAT Gateway per AZ and configure each AZ's private subnets to route to the NAT Gateway in the same AZ. This is more expensive but eliminates the cross-AZ dependency and cross-AZ data transfer charges.

NAT Gateway charges: hourly rate plus per-GB data processing. Data that passes through the NAT Gateway is charged even if it's going to an AWS service. This is why VPC Endpoints matter — traffic to S3 and DynamoDB through a Gateway Endpoint bypasses the NAT Gateway entirely, saving both cost and latency.

NAT Instance is the legacy alternative — an EC2 instance you manage yourself, configured to perform NAT. You must disable source/destination check on the NAT instance (EC2 by default drops packets where it's not the source or destination — NAT requires forwarding packets for other instances). NAT Instances require manual scaling, patching, and HA configuration. The exam occasionally presents NAT Instance as a cost-saving option for low-throughput scenarios (cheaper than NAT Gateway for very low traffic), but NAT Gateway is always the answer when the question asks for managed, highly available, or least operational overhead.

Elastic IP addresses are static public IPv4 addresses allocated to your account. You associate them with EC2 instances or NAT Gateways. An EIP remains yours until you release it — if you stop and start an instance, the EIP stays the same, unlike the auto-assigned public IP which changes on stop/start. You are charged for EIPs that are allocated but not associated with a running instance — AWS charges for wasted address space.

IPv6 in VPCs: IPv6 CIDR blocks are /56 for the VPC and /64 for subnets. IPv6 addresses are globally unique and public by design — there is no concept of private IPv6. For instances in private subnets that need outbound-only IPv6 internet access (without being reachable inbound), you use an Egress-Only Internet Gateway. EIGW is the IPv6 equivalent of NAT Gateway — it allows outbound IPv6 traffic but blocks unsolicited inbound connections.

VPC DNS: each VPC has a DNS resolver at the base VPC CIDR plus two (e.g., for 10.0.0.0/16, the resolver is at 10.0.0.2). The enableDnsHostnames attribute controls whether EC2 instances get public DNS hostnames. The enableDnsSupport attribute controls whether DNS resolution is enabled at all. Both must be true for instances to resolve public DNS names and get public hostnames. These settings also matter for VPC Endpoints — interface endpoints require enableDnsSupport to be true and Private DNS to be enabled on the endpoint for the service's default DNS name to resolve to the private endpoint IP.

Route tables: every subnet must be associated with a route table. Routes have a destination CIDR and a target. The most specific route wins — longest prefix match. Local route (the VPC CIDR to local) is always present and cannot be deleted or overridden. Routes are evaluated most-specific first: /32 before /24 before /16 before /0. The 0.0.0.0/0 route is the default route — traffic that doesn't match any more-specific route goes here. In a public subnet, 0.0.0.0/0 points to the IGW. In a private subnet, 0.0.0.0/0 points to the NAT Gateway (or is absent, meaning no internet access).

## VPC Endpoints

VPC Endpoints provide private connectivity from your VPC to AWS services without traffic leaving the AWS network and without requiring an IGW, NAT Gateway, or VPN.

Gateway Endpoints support S3 and DynamoDB only. They work by adding a route to your route table with the target being the gateway endpoint. Traffic to S3 or DynamoDB matching that route goes through the endpoint rather than the internet. Gateway Endpoints are free — no hourly charge, no data processing charge. They do not use ENIs and do not appear as IP addresses in your subnet. They are regional — a Gateway Endpoint for S3 in us-east-1 only covers S3 endpoints in us-east-1. The endpoint has a resource policy that controls which S3 buckets can be accessed through it — you can restrict access to specific buckets for data exfiltration prevention.

Interface Endpoints (AWS PrivateLink) support most other AWS services — SSM, Secrets Manager, KMS, CloudWatch, EC2 API, STS, SNS, SQS, API Gateway, and many more. They work by creating an ENI in your subnet with a private IP address. Traffic to the service goes to that ENI. Interface Endpoints have hourly charges ($0.01/hour per AZ) plus data processing charges ($0.01/GB). For high-volume traffic to services like SSM or Secrets Manager, the cost adds up but is usually justified by the security benefit of keeping traffic private.

Interface Endpoints support Private DNS. When Private DNS is enabled on the endpoint, the service's default DNS name (e.g., ssm.us-east-1.amazonaws.com) resolves to the endpoint's private IP within the VPC. Applications don't need to be updated to use a different endpoint URL — they continue using the standard service URL and automatically get routed through the private endpoint. Enabling Private DNS requires the VPC to have enableDnsHostnames and enableDnsSupport set to true.

Gateway Load Balancer Endpoints are a third type used with GWLB for traffic inspection. They intercept traffic flowing between subnets and route it through third-party appliances.

VPC Endpoint policies are resource policies on the endpoint that restrict what can be done through the endpoint. For an S3 Gateway Endpoint, you might write a policy that only allows access to buckets in your account, preventing data exfiltration to external S3 buckets. Combined with an S3 bucket policy that denies access unless the request comes through the endpoint, you can enforce that all S3 access from the VPC goes through the endpoint and only to approved buckets.

## VPC Peering

VPC Peering creates a private network connection between two VPCs — in the same account, across accounts, or across regions. Traffic stays on the AWS backbone.

The key constraint is that VPC peering is non-transitive. If VPC A peers with VPC B, and VPC B peers with VPC C, VPC A cannot reach VPC C through VPC B. The packets won't be forwarded. You would need a direct peering connection between A and C. This is the fundamental scalability problem with peering — in a mesh of N VPCs, you need N*(N-1)/2 peering connections. For 10 VPCs, that's 45 connections. For 50 VPCs, that's 1,225 connections. Each connection requires route table entries on both sides and security group rules. It becomes unmanageable quickly.

CIDR overlap is forbidden in peering. If VPC A uses 10.0.0.0/16 and VPC B also uses 10.0.0.0/16, you cannot peer them — the router wouldn't know which VPC to send traffic to. This is why CIDR planning matters in multi-VPC architectures.

After creating a peering connection, you must add route table entries on both sides. The connection itself doesn't create any routes — you manually add a route in VPC A's route table pointing to VPC B's CIDR via the peering connection, and vice versa. Security Groups can reference peering connections — in VPC A, you can write a Security Group rule with the source being a Security Group in VPC B (for same-region peering within the same account).

Inter-region peering is supported. Traffic is encrypted in transit automatically when it traverses the AWS backbone between regions. You cannot use an IGW, VPN, Direct Connect, or NAT device across a VPC peering connection.

## Transit Gateway

Transit Gateway is a regional network hub that simplifies connectivity at scale. Instead of creating a full mesh of peering connections, each VPC connects to the TGW with a single attachment, and the TGW routes between them.

TGW supports transitive routing — VPC A attached to TGW can reach VPC B attached to TGW without a direct peering between A and B. The TGW routes the traffic. This is the fundamental advantage over VPC peering.

TGW attachments: VPC attachments, VPN attachments (Site-to-Site VPN), Direct Connect Gateway attachments, peering attachments (to another TGW in the same or different region), and Connect attachments (for SD-WAN integration using GRE tunnel + BGP).

TGW route tables control which attachments can communicate. By default, all attachments are in the same route table and can communicate with each other. For network segmentation, you create multiple route tables. For example: a production route table that includes production VPCs and the Direct Connect attachment, and a development route table that includes dev VPCs and the internet-only VPN — production and development never route to each other even though they're on the same TGW.

Route propagation: VPC attachment routes can be automatically propagated to the TGW route table. When you create a VPC attachment, the VPC's CIDR is propagated to the associated route table. VPN and Direct Connect attachments propagate BGP routes. Static routes can be added manually.

TGW peering: connect TGWs in different regions using peering attachments. Traffic between them traverses the AWS global backbone. TGW peering does not support route propagation — you must add static routes manually on both sides. This is the main operational difference from within-region TGW routing.

Multicast support: TGW supports multicast routing, which standard VPC peering does not. If your application uses multicast (financial market data distribution, video streaming to multiple receivers), TGW with a multicast domain is the required architecture.

Resource Access Manager sharing: share a TGW with other accounts in your Organization using RAM. Member accounts create attachments to the shared TGW. The TGW owner manages the route tables. This is the architecture for a centralized network team managing connectivity on behalf of multiple application teams.

TGW bandwidth: each VPC attachment supports up to 50 Gbps burst throughput. VPN attachments are limited by VPN tunnel bandwidth (1.25 Gbps per tunnel, ECMP allows multiple tunnels). Direct Connect Gateway attachments inherit the Direct Connect bandwidth.

Network inspection with TGW: a common architecture routes all traffic through a centralized inspection VPC containing a firewall appliance (AWS Network Firewall or a third-party NGFW via GWLB). Traffic from spoke VPCs goes to TGW, routes to the inspection VPC, gets inspected, then routes back through TGW to the destination. This is implemented with separate TGW route tables — one for spoke-to-inspection routing and one for post-inspection routing to destination.

## Direct Connect

Direct Connect is a dedicated network connection from your on-premises environment to AWS. The physical connection goes from your data center to a Direct Connect location (a colocation facility where AWS has equipment), and from there to the AWS region.

Connection types: Dedicated connections are 1, 10, or 100 Gbps ports on AWS Direct Connect routers. You or your colocation provider physically connects your equipment to the AWS router. Hosted connections are sub-1G or multi-Gbps connections provisioned by an APN Partner who partitions their Dedicated connection and sells you a slice. Hosted connections range from 50 Mbps to 10 Gbps.

Virtual Interfaces (VIFs) are the logical layer on top of a physical connection. One physical connection supports multiple VIFs. Private VIF connects to a VPC via a Virtual Private Gateway (VGW) or Direct Connect Gateway. Public VIF connects to AWS public endpoints (S3, DynamoDB, other public services) without going through the internet. Transit VIF connects to a TGW via a Direct Connect Gateway for routing to multiple VPCs.

Direct Connect Gateway is a global resource (not regional) that connects a single Direct Connect connection to multiple VPCs in multiple regions. Without a DCG, one Direct Connect connection can only attach to one VGW in one region. With a DCG, you attach the Direct Connect connection once and then associate the DCG with VGWs or TGWs in multiple regions.

Encryption: Direct Connect is a private connection but not encrypted by default. The traffic is not going over the internet, but someone with access to the physical layer could theoretically intercept it. For data that requires encryption in transit (compliance requirements, sensitive data), your options are MACsec (IEEE 802.1AE Layer 2 encryption — encrypts the Ethernet frames; requires dedicated connections and specific Direct Connect equipment) or running a VPN tunnel over the Direct Connect connection (IPsec VPN over the private VIF — adds encryption but introduces VPN overhead and throughput limits).

BGP is required for Direct Connect. You run BGP between your router and the AWS Direct Connect router. BGP advertises your on-premises prefixes to AWS and learns AWS prefixes from AWS. For private VIF, AWS advertises your VPC CIDR(s). For public VIF, AWS advertises all AWS public IP ranges.

BGP communities are used to control route propagation. For public VIF, you can tag your routes with communities to control which AWS regions receive them (scope to a single region, to a continent, or globally). AWS uses well-known communities to control which routes you receive.

Redundancy: a single Direct Connect connection is a single point of failure — the physical cable, the port, the Direct Connect location. AWS recommends two connections from different Direct Connect locations for redundancy. For maximum resilience, two connections from two different locations, connected to routers in two different physical buildings on-premises.

The exam resilience pattern: primary connection is Direct Connect for low latency and consistent bandwidth; backup connection is Site-to-Site VPN over the internet. If the Direct Connect circuit fails, BGP reconverges and traffic flows over the VPN. This is the standard hybrid network architecture for most enterprises.

Direct Connect SiteLink enables Direct Connect customers to use the AWS backbone to route traffic between on-premises locations through the Direct Connect network — instead of routing between sites over the internet, traffic goes on-premises → DC location → AWS backbone → DC location → on-premises. This makes Direct Connect a private WAN backbone.

Link Aggregation Groups (LAG): bond multiple Direct Connect connections into a single managed connection for higher bandwidth and redundancy. All connections in a LAG must be the same speed and terminate at the same Direct Connect location. If one connection fails, traffic redistributes to the remaining connections.

## Site-to-Site VPN

Site-to-Site VPN creates an IPsec tunnel between your on-premises network and a Virtual Private Gateway (VGW) or Transit Gateway attached to your VPC. The tunnel runs over the public internet.

Each VPN connection consists of two tunnels for redundancy — two tunnels to two different AWS endpoints. Both tunnels should be configured and active; AWS may bring one tunnel down for maintenance at any time. If only one tunnel is configured, maintenance windows cause outages. Both tunnels use BGP (dynamic routing) or static routing.

Throughput: each tunnel supports up to 1.25 Gbps. If you need more throughput, you can use ECMP (Equal Cost Multi-Path routing) over a Transit Gateway — multiple VPN connections to the same TGW with routes aggregated, allowing the TGW to spread traffic across tunnels. Static routing does not support ECMP; BGP is required.

Customer Gateway is the AWS resource representing your on-premises VPN device. It records the public IP address of your device and the BGP ASN. The actual VPN device on your side must support IPsec IKEv1 or IKEv2.

Accelerated VPN uses AWS Global Accelerator to route VPN traffic from your on-premises location to the nearest AWS edge location, then over the AWS backbone to the VGW. This reduces latency and improves consistency compared to routing over the public internet for the entire path. Only available with Transit Gateway, not VGW.

VPN CloudHub: if you have multiple on-premises sites, each with its own VPN connection to the same VGW, those sites can communicate with each other through the VGW using VPN CloudHub. The VGW acts as a hub and the on-premises sites are spokes. This is a simple WAN solution using existing VPN infrastructure.

## CloudFront

CloudFront is a CDN with over 450 edge locations globally. Content is cached at edge locations close to users, reducing latency and offloading traffic from origin servers.

Origins: S3 buckets, ALBs, EC2 instances, API Gateway endpoints, and custom HTTP origins. A CloudFront distribution can have multiple origins with different cache behaviors routing to each based on URL path pattern.

Origin Access Control (OAC) is the mechanism for making S3 origins private. Without OAC, an S3 bucket must be public for CloudFront to serve from it — anyone who discovers the bucket URL can bypass CloudFront (and WAF, and signed URL restrictions). With OAC, the bucket policy allows only CloudFront to read the bucket. Requests from CloudFront include a signed header that S3 validates. OAC replaced Origin Access Identity (OAI) — OAI is the legacy mechanism and OAC is preferred for new distributions. OAC supports SSE-KMS encrypted S3 buckets; OAI does not.

Cache behaviors define how CloudFront handles requests matching specific path patterns. A distribution has a default cache behavior (matches everything) and can have additional behaviors with specific patterns evaluated in order. Each behavior specifies: the origin, cache policy (TTL settings, which request attributes to include in the cache key), origin request policy (which headers/cookies/query strings to forward to origin), response headers policy (what headers to add to responses), viewer protocol policy (HTTP only, HTTPS only, redirect HTTP to HTTPS), allowed HTTP methods, and whether to compress objects.

Cache key: by default, CloudFront caches based on the URL path only. If you need CloudFront to cache different versions for different query strings, cookies, or headers, you must include them in the cache key. Including more attributes in the cache key improves cache hit accuracy but reduces the cache hit ratio (more unique cache keys means fewer hits). The balance is to include only what's strictly necessary for content differentiation.

TTL: objects are cached based on Cache-Control headers from the origin (max-age, s-maxage). You can override with minimum TTL, maximum TTL, and default TTL in the cache policy. Setting a very short TTL defeats the purpose of caching; setting a very long TTL means stale content is served until you invalidate.

Cache invalidation: when you update content at the origin, cached copies at edge locations serve the old content until TTL expires. You can force invalidation by submitting invalidation requests specifying paths (/images/logo.png or /* for all). Invalidations are not instantaneous — they propagate to all edge locations within a few minutes. First 1,000 invalidation paths per month are free; $0.005 per path after that. For frequent updates, the better pattern is versioned file names (logo-v2.png instead of logo.png) so old and new content coexist without invalidation.

Origin Shield is an optional additional caching layer between the edge locations and the origin. When multiple edge locations need to fetch the same content from the origin (cache miss), without Origin Shield each edge location makes a separate request to the origin. With Origin Shield, all edge location requests go through Origin Shield first — if Origin Shield has the content cached, it serves the edge locations without hitting the origin at all. This dramatically reduces origin load for popular content. Origin Shield is in a specific AWS region (you choose the one closest to your origin) and adds a small amount of latency.

Lambda@Edge allows you to run code at CloudFront edge locations in response to CloudFront events. Four event types: viewer request (when CloudFront receives a request from the viewer, before checking the cache), origin request (when CloudFront forwards a request to origin, only on cache miss), origin response (when CloudFront receives a response from origin, before caching), and viewer response (when CloudFront returns a response to the viewer, after cache check). Lambda@Edge functions run in Node.js or Python, have a 5-second timeout for viewer events and 30 seconds for origin events, and are deployed from us-east-1 and replicated to edge locations automatically. Use cases: authentication and authorization (check a JWT token in the viewer request), A/B testing (vary the origin based on a cookie or header), URL rewriting and redirecting, personalizing content based on viewer location or device type.

CloudFront Functions is a lighter-weight alternative for simpler use cases. JavaScript only, sub-millisecond execution, only viewer request and viewer response events (no origin events). Much cheaper than Lambda@Edge — $0.10 per million invocations versus $0.60 per million for Lambda@Edge. Use CloudFront Functions for header manipulation, URL rewrites, simple redirects, and request normalization. Use Lambda@Edge when you need network calls, more compute time, or origin events.

Signed URLs and Signed Cookies restrict access to content. Signed URL: grants access to a single specific object. The URL includes a signature, an expiry time, and optionally an IP restriction. Use signed URLs when you want to control access per-object — video streaming where each user gets a signed URL for their specific video file. Signed Cookies: grant access to multiple objects matching a pattern. The cookies are set once and apply to all subsequent requests in the session. Use signed cookies for authenticated sections of a website where users should access multiple files without generating individual signed URLs for each.

Two signers for signed URLs/cookies: CloudFront key pairs (legacy, uses root account) and trusted key groups (current recommended approach, uses IAM-managed key pairs). Trusted key groups are preferred because they don't require root account access to rotate keys.

Geo-restriction: block or allow access from specific countries. Two methods: CloudFront's built-in geo-restriction (uses a country list — all or nothing for a country) or Lambda@Edge with a third-party geolocation database (more granular, can restrict by city or region). The built-in geo-restriction is simpler but coarser.

CloudFront with WAF: attach a WAF Web ACL to the CloudFront distribution. WAF rules are evaluated at the edge — requests are blocked before they reach your origin. This is the most efficient DDoS and attack protection architecture because malicious traffic is dropped at the edge rather than at the load balancer or EC2 instances, saving bandwidth and compute cost at origin.

Real-time logs: stream CloudFront access logs to Kinesis Data Streams in real time (within seconds). From there, logs go to Kinesis Data Firehose for delivery to S3 or OpenSearch, or to Lambda for processing. Standard access logs are delivered to S3 in batches with up to 24-hour delay. Real-time logs are for operational monitoring and anomaly detection that can't wait for batch delivery.

Field-level encryption: an additional layer of protection for sensitive fields in HTTP POST requests. CloudFront encrypts specific form fields (like credit card numbers or SSNs) with a public key at the edge. Only your backend application with the corresponding private key can decrypt them. Even if someone intercepts traffic between CloudFront and the origin (which shouldn't happen on the AWS backbone but adds defense in depth), they can't read the encrypted fields.

## Global Accelerator

Global Accelerator provides two static Anycast IP addresses that route traffic to the nearest AWS edge location. From the edge, traffic travels over the AWS private backbone to your endpoints. It supports any TCP and UDP application.

Anycast: both static IPs are announced from all AWS edge locations simultaneously. When a user's DNS resolver resolves your application's domain to those IPs, their traffic is automatically routed to the nearest edge location by internet routing protocols (BGP Anycast). The user doesn't choose the edge location — the internet routing infrastructure does, based on which path is shortest.

From the edge, Global Accelerator routes traffic over the AWS private network to your endpoints. The AWS backbone has higher reliability and lower latency than the public internet, especially for trans-oceanic connections. A user in Asia accessing a US-east origin sees improved performance because their traffic rides the public internet only to the nearest Asian edge location, then takes the fast AWS backbone across the Pacific.

Endpoints: ALBs, NLBs, EC2 instances, and Elastic IPs in one or more AWS regions. You can configure endpoints in multiple regions for geographic redundancy. Each endpoint has a weight (controls traffic distribution) and a health check.

Health checks: Global Accelerator continuously health-checks your endpoints. When an endpoint fails (health check fails), Global Accelerator automatically reroutes traffic to the next-best healthy endpoint within about 30 seconds. This is faster than DNS-based failover (which is bounded by DNS TTL) and more deterministic because Global Accelerator controls the routing rather than DNS caches.

Traffic dials: at the endpoint group level, you can set a traffic dial (0-100%) that overrides the weight-based distribution. Setting a traffic dial to 0 drains all traffic from a region without removing the endpoint. Used for maintenance, canary testing, or gradual region failover.

Static IPs: the two static IPs are fixed for the lifetime of the accelerator. You whitelist these IPs in firewall rules, partner whitelists, and client applications once. If you need to change regions or endpoints behind Global Accelerator, the IPs stay the same. This is a significant operational benefit over DNS-based approaches where the IP changes when you change your endpoint.

Global Accelerator versus CloudFront: the exam tests this repeatedly. Use Global Accelerator for non-HTTP protocols (UDP gaming, custom TCP protocols, VoIP), when clients need to whitelist static IPs (corporate firewalls, partner integrations), when you need deterministic sub-30-second failover, or when you need the same acceleration for both static and dynamic content without caching. Use CloudFront when you need HTTP/HTTPS caching, WAF integration, Lambda@Edge compute at the edge, signed URLs for content protection, or field-level encryption.

Both use the AWS edge network. The difference is what they do at the edge: CloudFront caches content and runs compute; Global Accelerator just routes traffic without caching or compute.

Custom routing accelerator: a specialized type that lets you deterministically route each user to a specific EC2 instance based on the source IP and port. Used for stateful applications where the same user must always reach the same backend — gaming servers, collaborative applications, real-time communication where session state is on a specific instance.

## Load Balancers

All AWS load balancers are fully managed, scale automatically, and exist across multiple AZs. You don't manage the underlying instances. The three current types — ALB, NLB, and GWLB — serve fundamentally different use cases.

Application Load Balancer operates at Layer 7. It understands HTTP and HTTPS. Routing decisions can be based on: host header (api.example.com versus www.example.com), URL path (/api/* versus /images/*), query string parameters (?version=2), HTTP headers (including custom headers), HTTP method (GET versus POST), and source IP.

ALB listener rules are evaluated in order of priority (lowest number first). Each rule has conditions (path pattern, host header, etc.) and actions (forward to target group, redirect, return fixed response, authenticate with Cognito or OIDC, forward with header insertion). The default rule catches everything that doesn't match other rules.

Target groups are the backend pools behind an ALB. Targets can be EC2 instances, IP addresses (for on-premises or peered VPC targets), Lambda functions, or another ALB. Each target group has its own health check configuration. A listener rule forwards to a target group (or multiple target groups with weights for canary). You can have multiple target groups behind one ALB for different paths — /api/* goes to the API target group, /* goes to the frontend target group.

Weighted target group routing lets you split traffic between two target groups at a configurable ratio. This is how you implement canary deployments at the load balancer level — send 5% of traffic to the new version target group and 95% to the old version target group, then gradually adjust.

ALB supports connection draining (Deregistration Delay): when a target is deregistered or marked unhealthy, the ALB waits a configurable period (default 300 seconds) before stopping sending new requests to it, allowing in-flight requests to complete. Set this lower for fast-scaling environments where instances are short-lived.

Sticky sessions (session affinity): ALB can route all requests from a specific client to the same target using a cookie. ALB-generated cookies (AWSALB) are created by the load balancer. Application-based cookies are created by your application. Sticky sessions break the load balancer's ability to distribute load evenly — use them only when the application genuinely requires session affinity, and prefer stateless architectures (session state in ElastiCache or DynamoDB) instead.

ALB access logs: every request is logged to S3 with details including client IP, request time, request path, backend target, response code, and latency. These logs are batch-delivered — not real-time. For real-time request monitoring, use CloudWatch metrics (request count, active connections, response codes, latency). For anomaly detection or debugging, use access logs with Athena queries.

X-Forwarded-For header: ALB adds this header to every request to the backend, containing the original client IP address. Your backend application reads this header to get the true client IP rather than the load balancer's IP. If there are multiple proxies in the chain, the header contains a comma-separated list — the leftmost IP is the original client.

ALB and HTTPS: ALB terminates SSL/TLS. The certificate is stored in ACM (AWS Certificate Manager). ALB supports multiple SSL certificates (for multiple domains) using SNI (Server Name Indication) — the client sends the hostname it's connecting to in the TLS handshake, and ALB selects the correct certificate. For re-encryption (end-to-end TLS), the ALB establishes a new TLS connection to the backend targets — the backend targets must also have TLS certificates.

Security Groups on ALB: unlike NLB, ALB has Security Groups. You control inbound traffic to the ALB with its Security Group, and you can reference the ALB's Security Group as the source in the EC2 instance's Security Group — instances only accept traffic from the ALB, not directly from clients.

Network Load Balancer operates at Layer 4. It routes based on IP protocol, source/destination IP, source/destination port, and TCP sequence number. It does not inspect HTTP content. It routes based purely on network-level information.

NLB has static IP addresses per AZ — one static IP per AZ where it's active. This is a critical distinction from ALB, which has no static IPs. When clients need to whitelist the load balancer's IP addresses in their firewall rules, NLB is required. You can also assign Elastic IPs to the NLB's per-AZ IPs, giving you static IPs you fully control.

NLB preserves the source IP address of the client all the way to the target. ALB replaces the source IP with the load balancer's IP (the original client IP is in X-Forwarded-For). NLB targets see the actual client IP natively without reading any header. This matters for applications that need the client IP for logging, rate limiting, or access control at the application level.

NLB does not have Security Groups. Traffic from any source can reach the NLB. Security is enforced at the target (EC2 instance Security Groups must allow traffic from 0.0.0.0/0 on the target port if clients are sending from arbitrary IPs, because NLB preserves source IPs and the EC2 instance sees traffic from the internet directly). This is a common misconfiguration: people add an NLB in front of EC2 instances and wonder why traffic isn't reaching the instances — the EC2 Security Group is only allowing traffic from the NLB's IP, but NLB passes through the original client IP.

NLB supports TCP, UDP, TLS, and TCP_UDP listeners. TLS termination at NLB (like ALB) offloads TLS processing from backends. NLB also supports TLS pass-through — it routes encrypted traffic to targets without decrypting, so the target handles TLS. This is required when end-to-end encryption must be maintained and you can't inspect traffic at the load balancer.

NLB performance: capable of handling millions of requests per second with consistent low latency (measured in microseconds for TCP, not milliseconds). Designed for extreme throughput — gaming, financial market data, real-time communications.

NLB cross-zone load balancing: when disabled (the default for NLB, unlike ALB where it's enabled by default), each NLB node only distributes traffic to targets in its own AZ. If AZ-A has 2 targets and AZ-B has 8 targets, and traffic is split evenly between AZ-A and AZ-B NLB nodes, AZ-A targets each get 25% of total traffic and AZ-B targets each get 6.25% — highly uneven. Enabling cross-zone load balancing distributes across all targets in all AZs evenly. When cross-zone is enabled on NLB, data transfer between AZs incurs charges.

Gateway Load Balancer operates at Layer 3/4 and is purpose-built for deploying, scaling, and managing third-party virtual network appliances — firewalls, intrusion detection and prevention systems, deep packet inspection systems, and traffic analytics tools.

GWLB uses the GENEVE protocol (encapsulation over UDP port 6081) to forward traffic to appliances. The appliance receives the original packet, inspects or modifies it, and returns it to the GWLB. GWLB then forwards the packet to its original destination. From the perspective of the traffic source and destination, GWLB is transparent — they see each other's IPs, not the appliance's IP.

The architecture: traffic flows into a VPC and is intercepted by GWLB via GWLB Endpoints in the route tables. The route table in the spoke VPC sends traffic to a GWLB Endpoint. The GWLB Endpoint routes it to the GWLB in the inspection VPC. GWLB fans out to a pool of appliance instances (for scale and HA), each appliance inspects the packet, and GWLB returns it to the endpoint, which routes it to the original destination.

Appliance scaling: the GWLB distributes traffic to appliance instances in its target group using a flow-based hash — all packets in the same flow go to the same appliance, ensuring consistent inspection state. The GWLB scales the number of appliance instances automatically (or you configure Auto Scaling). If an appliance fails its health check, GWLB stops sending traffic to it.

The GWLB exam pattern: any question involving third-party firewalls, IDS/IPS, or network appliances at scale where you need transparent traffic inspection → Gateway Load Balancer with GWLB Endpoints in the route tables.

## DNS and Route 53 Advanced Concepts

Route 53 is the AWS DNS service. It is global — there are no regions to select, it's available everywhere. Route 53 handles domain registration, DNS resolution, and health checking.

Hosted zones: a public hosted zone contains DNS records for a domain accessible from the internet. A private hosted zone contains records only accessible from associated VPCs. You associate a private hosted zone with one or more VPCs. If you have multiple VPCs that need to resolve the same private domain, associate all of them with the hosted zone. You can associate VPCs from different accounts using CLI/SDK (not the console).

Resolver: within a VPC, the Route 53 Resolver (at VPC base + 2) handles DNS queries. It resolves Route 53 private hosted zones, public Route 53 records, and forwards other queries to the internet. Route 53 Resolver Endpoints extend DNS resolution in hybrid architectures.

Inbound Resolver Endpoints: create ENIs in your VPC that on-premises DNS servers can send queries to. On-premises systems can resolve Route 53 private hosted zones and VPC-internal DNS names by forwarding queries to the inbound endpoint's IP addresses.

Outbound Resolver Endpoints: create ENIs in your VPC that the Resolver uses to forward DNS queries to on-premises DNS servers. You create Resolver Rules (forward rules) that specify which domain names should be forwarded to which on-premises DNS server IPs. For example, forward all *.corp.internal queries to the on-premises Active Directory DNS server.

This combination — inbound + outbound resolver endpoints — is the standard hybrid DNS architecture. On-premises can resolve AWS-internal names, and EC2 instances can resolve on-premises names, without needing a custom DNS server in AWS.

Resolver DNS Firewall: allows you to block DNS queries to known malicious domains or restrict which domains your VPC resources can query. You create rule groups with domain lists (block lists or allow lists) and associate them with VPCs. This prevents DNS exfiltration (malware using DNS queries to exfiltrate data to attacker-controlled domains) and blocks connections to known command-and-control domains.

DNSSEC: Route 53 supports DNSSEC signing for public hosted zones. DNSSEC adds cryptographic signatures to DNS records, allowing resolvers to verify that records haven't been tampered with (DNS spoofing protection). You enable DNSSEC on a hosted zone, Route 53 generates a key signing key (KSK) stored in KMS, and signs your zone.

## Network Firewall

AWS Network Firewall is a managed stateful firewall and intrusion prevention service for VPCs. It sits in a dedicated subnet (firewall subnet) and traffic is routed through it via route table manipulation.

Network Firewall provides: stateful packet inspection, stateless packet filtering, intrusion prevention (Suricata-compatible rule sets), domain filtering (block/allow by domain name), and protocol detection.

Rule groups: stateless rule groups evaluate each packet independently (like NACLs — source/destination IP, port, protocol). Stateful rule groups maintain connection state and can inspect application-layer content. Suricata-compatible rule strings let you use the extensive Suricata community rule sets or write your own.

Firewall policy: combines stateless and stateful rule groups with a default action (allow or drop for stateless; alert or drop for stateful). One policy per firewall.

The architecture: traffic enters the VPC, route tables direct it to the firewall endpoint (a GWLB endpoint managed by Network Firewall), the firewall inspects it, and allowed traffic exits the firewall endpoint toward its destination. The route table manipulation is the key — you add routes so that traffic between subnets or to/from the internet passes through the firewall subnet.

Centralized Network Firewall with TGW: all spoke VPC traffic routes through TGW to a central inspection VPC running Network Firewall. The Network Firewall inspects all inter-VPC traffic and internet-bound traffic. This is the scalable enterprise pattern — one firewall policy enforced for all traffic in the organization.