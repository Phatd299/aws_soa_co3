Route 53 DNS and CloudFront Content Delivery for SOA-C03
14 min read
SOA-C03 · Networking and Content Delivery
Updated
18 Jul 2026
By the SaveMyCert Editorial Team
Mapped to “Networking and Content Delivery” in the official SOA-C03 exam guide
Amazon Route 53 is AWS's DNS service — hosted zones, records, routing policies, and the Resolver that answers every query inside your VPCs — and Amazon CloudFront is the CDN that delivers your content from edge locations worldwide. Task 5.2 of the SOA-C03 exam tests whether you can configure both as an operator: choose alias records over CNAMEs at the zone apex, wire hybrid DNS with Resolver endpoints, pick the routing policy a scenario demands, enable the query log that answers the question being asked, and build CloudFront distributions with Origin Access Control, sane TTLs, and an ACM certificate in the right Region. It closes with the exam's favorite comparison: CloudFront versus AWS Global Accelerator. Configuration lives here; diagnosing cache misses and failed resolutions belongs to Task 5.3, and health-check mechanics sit in Domain 2 — this lesson references them only where routing policies depend on them.

On this page
8 sections
What you’ll learn
Configure public and private hosted zones and choose the right record type for each job
Use alias records at the zone apex and TTL reductions ahead of planned migrations
Set up Route 53 Resolver endpoints and forwarding rules for hybrid DNS in both directions
Select the routing policy — weighted, latency, failover, geolocation, geoproximity, or multivalue — that matches a scenario
Enable the right query logging surface: public DNS query logs versus Resolver query logs
Configure CloudFront origins with OAC, cache behaviors, TTLs, invalidations, and ACM certificates, and know when Global Accelerator wins
Hosted zones and record types in practice
Route 53 organizes DNS records into hosted zones, and the first configuration decision is visibility. A public hosted zone answers queries from anywhere on the internet for a domain you control. A private hosted zone answers only from the VPCs you explicitly associate with it — you can associate more VPCs at any time, but each VPC must have DNS resolution and DNS hostnames enabled or queries fall through. You can run both zones for the same name (split-horizon DNS): inside an associated VPC the private zone wins, so db.example.com can resolve to a private IP internally while the public internet sees nothing.

The record types you will actually operate:

A — name to IPv4 address; AAAA — name to IPv6 address.
CNAME — name to another name. A CNAME cannot coexist with other record types at the same name, and DNS forbids it at the zone apex (the bare domain).
TXT — arbitrary text: domain-ownership verification, SPF and DKIM email policy.
MX — mail servers with priorities; the lowest number is tried first.
Operationally, remember that hosting a zone and registering a domain are separate functions: you can host a zone for a domain registered elsewhere by pointing that registrar at your zone's four Route 53 name servers. Record changes propagate quickly from Route 53's side; what limits how fast the world sees them is resolver caching — which is where TTLs and alias records, the next section, come in.

Alias records and TTL operations
Alias records are a Route 53 extension to standard DNS that solve the CNAME problem. Where a CNAME can only point at another name and is illegal at the zone apex, an alias record points directly at an AWS resource — a CloudFront distribution, an Application or Network Load Balancer, an API Gateway API, a Global Accelerator, or another record in the same hosted zone — and it works at the apex. “Point example.com (no www) at an ALB” is an alias-record answer every time. Alias queries to AWS resources are also free, unlike standard queries, and Route 53 manages the TTL itself, tracking the target: when the load balancer's underlying IPs change, the alias follows automatically. An alias can also evaluate its target's health, which feeds failover configurations.

For everything else, TTL is your main operational lever. A record's TTL tells every caching resolver in the world how long it may keep the answer — which means the TTL that hurts you during a change is the one that was set before the change. The standard migration play: well ahead of a planned cutover, and at least one full old-TTL period in advance, lower the record's TTL from, say, 3600 seconds to 60. Wait for the old value to age out of caches, make the change, confirm the new target is healthy, then raise the TTL back so resolvers stop hammering your zone. Low TTLs buy fast changes at the cost of more queries; high TTLs buy a cheap, resilient steady state at the cost of slow changes. Lowering TTL before a migration is the single most-tested DNS operations move.

Route 53 Resolver and hybrid DNS
Every VPC has a built-in DNS server: the Route 53 Resolver, reachable at the VPC CIDR base plus two — in 10.0.0.0/16, that is 10.0.0.2 (the link-local address 169.254.169.253 also works). It resolves private hosted zone names, VPC-internal names, and public internet names for every instance in the VPC, and it is reachable only from inside the VPC. That last property is exactly what hybrid DNS has to work around, in two directions.

Inbound Resolver endpoint — on premises resolves AWS names. An inbound endpoint places ENIs with fixed private IPs into your subnets. You configure your on-premises DNS servers to conditionally forward AWS zones — say internal.example.com — to those IPs; the queries arrive over your Site-to-Site VPN and hand off to the Resolver. Result: data-center servers resolve private hosted zone records.

Outbound Resolver endpoint — VPCs resolve on-premises names. An outbound endpoint gives the Resolver a path out. You create forwarding rules — “for corp.example.com, forward to 10.10.0.53” — and associate each rule with the VPCs that need it. Instances keep querying their normal .2 address; matching queries are forwarded through the endpoint to your on-premises DNS servers.

Full hybrid DNS uses both endpoints at once, one per direction. On the exam, direction is everything: “on-premises servers must resolve VPC or private-zone names” is an inbound endpoint; “EC2 instances must resolve data-center domains” is an outbound endpoint plus forwarding rules. Neither replaces network connectivity — endpoints carry DNS queries over the private links you already have.

Routing policies compared — and a canary migration
Routing policies decide how Route 53 answers when multiple records share a name, and the exam tests them as scenario-to-policy matching.

Policy	How it answers	Reach for it when
Simple	One record set, no health checks	A single resource, nothing to steer
Weighted	Splits answers by assigned weights	Canary releases, gradual migrations, A/B splits
Latency-based	The Region measured fastest for that user	Multi-Region deployments chasing performance
Failover	Primary while its health check passes; otherwise secondary	Active-passive DR, a static fallback page
Geolocation	By the user's location (country, continent)	Compliance or localized content; always set a default record
Geoproximity	By distance to your resources, with a bias knob	Deliberately shifting the share of traffic a Region absorbs
Multivalue answer	Up to eight healthy records; the client picks	Cheap DNS-level load spreading with per-record health checks
Distinguish the geo pair: geolocation is about where the user is — an exact-match lookup, so Germany gets the Frankfurt answer, full stop — while geoproximity is about how far the user is from your resources, with a bias value that grows or shrinks each resource's catchment area. And multivalue answer is not a load balancer: it returns several IPs and lets the client choose, but it does health-check each record, which simple routing cannot.

The canary walkthrough — migrating an API from an old load balancer to a new one:

Days ahead, lower the record TTL to 60 seconds and let the old TTL age out of caches.
Create two weighted alias records for the same name: old target weight 90, new target weight 10.
Watch the new target's error rates and latency while it takes a tenth of production traffic.
Shift weights in stages — 50/50, then 0/100. A bad stage rolls back by setting the new record's weight to 0.
Once stable, delete the old record and raise the TTL back up.
Query logging: two logs, two questions
Route 53 offers two query logs, and exam questions hinge on knowing which log records which queries.

Public DNS query logging captures the queries that Route 53's authoritative name servers answer for a public hosted zone — who on the internet asked for your domain, for which record name and type, from which resolver location, and what response code they received. It delivers to CloudWatch Logs, and the log group must live in us-east-1. Turn it on when the question is about traffic to your zone: verifying a record is actually being queried after a cutover, spotting query floods, or confirming resolvers picked up a change.

Resolver query logging captures the opposite side: queries originating inside your VPCs — everything instances ask the .2 Resolver, whether the answer comes from a private zone, a forwarding rule, or the public internet. It logs to CloudWatch Logs, S3, or Amazon Data Firehose, and you enable it per VPC. Turn it on when the question is about your workloads' behavior: which instance queried a suspicious domain, why an application resolves the wrong name, or what Route 53 Resolver DNS Firewall just blocked — Resolver logs record the firewall's action, making the pair a natural audit combination.

The mapping to remember: “who is querying my public domain” is public query logging on the hosted zone; “what are my instances looking up” is Resolver query logging on the VPC. They are configured in different places, capture disjoint traffic, and one is never a substitute for the other.

CloudFront for operators: origins, OAC, and cache behaviors
A CloudFront distribution is configuration, not infrastructure you place: you define origins (where content lives) and cache behaviors (how requests are handled), and CloudFront serves users from its global edge locations.

Origins. For S3, the operational standard is a private bucket fronted by Origin Access Control (OAC): CloudFront signs its requests to S3, and the bucket policy grants access only to your distribution — no public bucket, no direct-to-bucket bypass. OAC is the successor to the legacy origin access identity (OAI), and new configurations should use it. Custom origins cover everything that speaks HTTP: an Application Load Balancer, an API Gateway API, or any web server with a publicly resolvable DNS name.

Cache behaviors. A distribution routes by path pattern: /api/* can go to the ALB origin with caching effectively disabled, /static/* to the S3 origin with long TTLs, and the default behavior catches whatever matches nothing else. Patterns are evaluated in order and the first match wins, so order them most-specific first. Each behavior sets its viewer protocol policy (for example, redirect HTTP to HTTPS), its allowed methods, and the two policies that trip people up: the cache policy defines the cache key — which headers, cookies, and query strings make cached responses distinct — plus the TTL bounds, while the origin request policy defines what is forwarded to the origin without joining the key. Everything you add to the cache key fragments the cache and drops the hit ratio, so forward what the origin needs through the origin request policy, and key only on what truly changes the response.

TTLs, invalidations, and HTTPS with ACM
Caching lifetime is a negotiation between your origin's headers and each behavior's three TTL settings. If the origin sends Cache-Control: max-age (or Expires), CloudFront honors it — clamped between the behavior's minimum TTL and maximum TTL. If the origin sends nothing, the default TTL applies. The operational consequences: an origin emitting Cache-Control: no-cache against a minimum TTL of 0 is effectively uncached, and a forgotten long default TTL is why a quick fix is not showing up for users.

When stale content must go now, an invalidation removes objects from edge caches ahead of expiry. Invalidations are billed per path beyond a monthly free allowance — the first 1,000 paths each month are free — and a wildcard such as /images/* counts as a single path. The better operational pattern avoids the problem entirely: versioned object names, app.a1b2c3.css instead of app.css, mean every deploy references new objects, old versions simply age out, and rollback is instant. Reserve invalidations for mistakes.

Two more controls to recognize: an origin group gives a behavior a primary and a secondary origin, failing over when the primary returns configured error codes — origin failover, at concept level — and signed URLs or signed cookies restrict delivery of private content to authorized viewers.

Finally, the certificate gotcha the exam loves: to serve a custom domain over HTTPS, the ACM certificate for a CloudFront distribution must exist in us-east-1, no matter where your origin or your users are. A certificate for an ALB origin, by contrast, lives in the ALB's own Region.

CloudFront vs Global Accelerator
CloudFront and AWS Global Accelerator both put your application behind AWS's global edge network, and the exam loves making you choose. They solve different problems: CloudFront is a content delivery network that caches HTTP responses at the edge; Global Accelerator is a network-layer front door that gives you two static anycast IP addresses and proxies TCP and UDP connections over the AWS backbone to the closest healthy regional endpoint — an ALB, NLB, EC2 instance, or Elastic IP. It caches nothing.

CloudFront	Global Accelerator
Protocols	HTTP and HTTPS	TCP and UDP, any port
Addresses	Distribution domain name (edge IPs vary)	Two static anycast IPs
Caching	Yes — edge caches, TTLs, invalidations	None
Failover	Origin failover within a distribution	Near-instant health-based shift between Regions — no DNS change, no TTL wait
Typical triggers	“Cache content at the edge,” static assets, video, APIs behind a CDN	“Static IPs for firewall allowlists,” non-HTTP protocols (gaming, VoIP, MQTT), instant multi-Region failover
The static-IP point deserves emphasis: because Global Accelerator's two anycast IPs never change, enterprise customers can allowlist them permanently — impossible with CloudFront or a bare load balancer. And because failover happens inside the accelerator rather than in DNS, there is no resolver cache to wait out. Read the scenario's nouns: “cache,” “content,” and “HTTP” point to CloudFront; “static IP,” “UDP,” and “instant regional failover” point to Global Accelerator. Some architectures legitimately use both — but never claim Global Accelerator accelerates cacheable content delivery; that is CloudFront's job.

💡
Tip. Expect scenario stems that name a constraint and want the one matching feature: “apex domain to an ALB” means an alias record; “on-prem servers must resolve VPC names” means an inbound Resolver endpoint (outbound plus forwarding rules for the reverse); “shift 10% of traffic to the new stack” means weighted routing; “static anycast IPs” or “UDP” means Global Accelerator, while “cache at the edge” means CloudFront. Certificate questions hinge on us-east-1 for CloudFront, and query-logging questions on which side of the Resolver the traffic originates. Configuration is tested here; diagnosing broken resolution and cache misses appears under Task 5.3.

Key takeaways
Alias records work at the zone apex and are free to query when targeting AWS resources; CNAMEs cannot sit at the apex.
On-prem must resolve VPC names: inbound Resolver endpoint. VPCs must resolve on-prem domains: outbound endpoint plus forwarding rules.
Failover routing is active-passive with health checks; multivalue answer returns up to eight healthy records but is not a load balancer.
Lower a record's TTL well before a migration so cached answers age out fast, then raise it after the cutover.
Public DNS query logs cover queries to your public zone; Resolver query logs cover queries from inside your VPCs.
CloudFront custom-domain certificates must come from ACM in us-east-1 — no other Region works.
Prefer versioned object names over invalidations; invalidations bill per path beyond the monthly free allowance, and a wildcard counts as one path.
Static anycast IPs, TCP/UDP, or instant regional failover means Global Accelerator; caching HTTP content means CloudFront.