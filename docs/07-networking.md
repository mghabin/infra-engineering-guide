# Chapter 07 — Networking

Opinionated, cloud-agnostic networking defaults for 2026. The thesis: the network is no longer your security perimeter, but it is still your blast-radius boundary, your cost boundary, and the thing that wakes you at 3 a.m. when DNS breaks. Treat IP space like a public API — once allocated, never reshape — and treat trust as a transport-layer concern: every flow authenticated and encrypted, no implicit "we're inside the VPC so it's fine."

> Conventions: **Do** = required default. **Don't** = reject in review unless a written exception exists. **Prefer** = strong default; deviate only with measurement or a documented constraint. Vendor parallels are stated explicitly because the primitives converge but the names do not.

---

## 1. Cloud network primitives

The mental model is the same across clouds; only the names change.

- Isolated L3 network: VPC (AWS) / VNet (Azure) / VPC (GCP, *global*).
- Subnet scope: AZ-scoped (AWS) / region-scoped (Azure, GCP).
- Source NAT for private hosts: NAT Gateway / Azure NAT Gateway / Cloud NAT.
- Cross-network L3 stitch: VPC Peering or TGW / VNet Peering or vWAN / VPC Peering or NCC.
- Private SaaS access: PrivateLink / Private Endpoint / Private Service Connect.
- Hybrid private circuit: Direct Connect / ExpressRoute / Cloud Interconnect.

Two non-obvious facts:

- **AWS subnets are AZ-scoped; Azure and GCP are region-scoped.** AWS forces one subnet per AZ per tier (a 3-AZ "private" tier is *three* subnets); capacity math below uses the AWS shape because it is the strictest ([AWS VPC subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html); [Azure VNet](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview); [GCP VPC](https://cloud.google.com/vpc/docs/vpc)).
- **GCP VPCs are global.** A single VPC spans every region you turn on; a misconfigured firewall rule has a planetary blast radius — review them like IAM.

---

## 2. IP address planning

**Allocate a /16 per VPC per environment per region, with /24 subnets as the starter brick.** A /24 nominally has 256 addresses; in cloud subnets the *usable* count is lower because **major cloud providers reserve 5 IPv4 addresses per subnet** (4 on GCP). VPC subnets are not classic Ethernet broadcast domains, so the "broadcast" slot is a bookkeeping reservation, not a real broadcast address. Provider specifics:

- AWS: 5 — network, VPC router, base+2 for DNS, "future use" ([AWS — VPC and subnet sizing](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html)).
- Azure: 5 — network, default gateway, two for Azure DNS mapping, subnet broadcast slot ([Azure — VNet FAQ](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq#are-there-any-restrictions-on-using-ip-addresses-within-these-subnets)).
- GCP: 4 — network, gateway, second-to-last, broadcast ([GCP — Subnet ranges](https://cloud.google.com/vpc/docs/subnets#reserved_ip_addresses_in_every_subnet)).

Design as if 5 are gone everywhere and you'll never be wrong.

**Use RFC 1918 with a written global plan before the first VPC.** Reserve a /10 (`10.0.0.0/10` ≈ 4M addresses) per cloud account / GCP project / Azure subscription, then carve /16s per VPC. Document every range in IPAM (AWS VPC IPAM, Azure Virtual Network Manager, NetBox) ([RFC 1918](https://datatracker.ietf.org/doc/html/rfc1918); [AWS VPC IPAM](https://docs.aws.amazon.com/vpc/latest/ipam/what-it-is-ipam.html)).

**Never overlap CIDRs across anything you might one day connect.** Peering, VPN, Transit Gateway, and ExpressRoute all refuse overlapping ranges; the "fix" is renumbering production. AWS Well-Architected calls this the most common multi-account network failure ([AWS WA — REL02-BP02](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_network_topology_ip_address_management.html)).

**RFC 6598 (`100.64.0.0/10`) is a constrained exception, not a default.** The RFC defines that range as *Shared Address Space* for service-provider CGN. Using it inside an enterprise has real collision risk: many ISPs hand it out to residential CPE (so remote-worker VPNs land overlapping), SD-WAN vendors use it internally, and M&A integrations frequently surface another party already squatting it. Use it only when (a) RFC 1918 is genuinely exhausted in the connectivity domain, (b) IPAM marks it non-routable externally, and (c) you've verified non-overlap with every VPN/SD-WAN/peer ([RFC 6598](https://datatracker.ietf.org/doc/html/rfc6598); [AWS — EKS custom networking](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html)).

**IPv4 exhaustion remediation, in order:**

- Add a **secondary RFC 1918 CIDR** to the VPC (all three clouds support it).
- Move pod/service ranges to a **non-routable secondary** (the legitimate use of RFC 6598 in EKS, or a dedicated /16 from the plan).
- **Adopt IPv6 east-west** (pods, services, internal LBs); keep IPv4 only at ingress/egress translation points (§12).
- **Renumber** as a last resort: dual-CIDR cutover, IPAM-driven allocation, DNS TTLs lowered weeks ahead, every hard-coded IP found and ticketed first.

**Don't** use the cloud provider's default VPC for anything real — defaults overlap across regions and ship with wide-open SGs ([AWS default VPC](https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html)).

---

## 3. Topology: stage-gate, don't over-build

The right topology is a function of how many independently managed networks you actually have and whether you need centralized inspection. Pick the *smallest* topology that fits today and the next 12 months.

- **Stage 1 — Single VPC/VNet.** 1–2 teams, one or a few accounts, no hybrid. Subnet-per-tier, security-group segmentation. Correct for most startups and most internal platforms in their first 18 months. Don't apologize for it.
- **Stage 2 — Peering or shared network.** Small multi-team orgs (≈3–10 workload accounts/projects, no central inspection). VPC peering in a hub-less star, **AWS RAM-shared subnets**, **VNet peering**, or **GCP Shared VPC** keep ops simple while preserving account isolation. Shared VPC avoids peering fees entirely ([GCP Shared VPC](https://cloud.google.com/vpc/docs/shared-vpc); [AWS VPC sharing via RAM](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-sharing.html)).
- **Stage 3 — Hub-and-spoke transit** (AWS TGW, Azure vWAN, GCP NCC). Justified when you have **(a)** hybrid connectivity to terminate, **(b)** a central inspection / egress-firewall requirement, or **(c)** more than ~10 independently managed networks where mesh peering becomes N×(N−1)/2 toxic. Each transit hub is a paid, region-scoped resource with per-attachment-hour and per-GB charges ([AWS TGW pricing](https://aws.amazon.com/transit-gateway/pricing/); [Azure vWAN pricing](https://azure.microsoft.com/en-us/pricing/details/virtual-wan/); [GCP NCC](https://cloud.google.com/network-connectivity/docs/network-connectivity-center/concepts/overview)).

**Adopt a Landing Zone model when you reach Stage 2/3.** Microsoft CAF "Azure landing zones" and AWS Control Tower / Multi-Account Strategy prescribe the same shape: a `connectivity` subscription/account hosting the hub, `identity` and `management` accounts at the org root, per-workload spokes connecting outward through the hub ([Microsoft CAF](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/); [AWS multi-account](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html)). Retro-fitting a landing zone is the most expensive infra migration most orgs do; design the *seams* on day one even if you only build Stage 1.

---

## 4. Private connectivity: PrivateLink vs Private Endpoint vs PSC

The three primitives converge in *intent* — expose a service via a private IP inside the consumer's network, no peering, no CIDR coupling — but differ in producer/consumer model, DNS automation, transitivity, cross-region reach, and pricing. Don't treat them as drop-in equivalents.

- **AWS PrivateLink.** Producer publishes an NLB behind a *VPC Endpoint Service*; consumer creates an *Interface Endpoint* (ENI in their subnet). DNS via Private Hosted Zones (manual, or auto for AWS-service endpoints). **Not transitive over TGW by default**; cross-region needs a peer endpoint ([AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html)).
- **Azure Private Endpoint / Private Link Service.** Producer publishes a *Private Link Service* fronted by a Standard LB; consumer attaches a *Private Endpoint* (NIC in their subnet). DNS auto-provisioned into Private DNS Zones when opted in. **Transitive across peered VNets** and via vWAN ([Azure Private Link](https://learn.microsoft.com/en-us/azure/private-link/private-link-overview); [Private Endpoint DNS](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns)).
- **GCP Private Service Connect.** Producer publishes a *service attachment*; consumer creates a *PSC endpoint* or *PSC backend*. DNS via Cloud DNS, often auto-created for Google APIs. **Reachable from anywhere the consumer VPC routes**, including across regions in the same global VPC ([GCP PSC](https://cloud.google.com/vpc/docs/private-service-connect)).

Decision matrix:

- **Consuming a managed PaaS** (object storage, databases, secrets, registries) → always private endpoint. The default public endpoint is an exfiltration surface; private endpoints also remove NAT egress costs.
- **One service exposed across orgs/accounts to many consumers** → PrivateLink/PE/PSC. Unidirectional, no CIDR coupling.
- **Same org, bidirectional reach, multiple services** → peering or shared VPC. Endpoints become unwieldy past a handful of services.
- **Many networks, hybrid, or central inspection** → transit hub (§3). Endpoints attach to spokes; the hub need not see per-service flows.

Pricing watch: every endpoint has an hourly charge per AZ/zone *plus* per-GB processing on AWS/Azure; PSC charges per forwarding rule and per GB. A few endpoints are free; a few hundred are a line item.

---

## 5. DNS

DNS is the one thing that, when broken, breaks everything else with no useful error message. Treat it like a tier-0 dependency. Two concerns get conflated and shouldn't be:

- **Authoritative private zone ownership** — *who owns the records* for `db.prod.internal`. Workload-owned, GitOps-managed, scoped to the VPC/VNet/project that runs the workload.
- **Recursive resolver placement** — *which IP a client asks*. Each cloud provides a per-VPC resolver at a well-known address (AWS `.2`, Azure `168.63.129.16`, GCP metadata). Use it by default.

**Run split-horizon DNS.** Public zones for what the internet sees; private zones (Route 53 PHZ, Azure Private DNS, Cloud DNS private zones) for everything inside the network ([AWS Route 53 PHZ](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html); [Azure Private DNS](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview); [Cloud DNS private zones](https://cloud.google.com/dns/docs/zones)). Private endpoints depend on this — the public name resolves to the public service unless a private zone overrides it.

**Prefer native zone association over hub resolvers.** All three clouds let you associate one private zone with many VPCs/VNets directly. That is the right default for a single org in one cloud. **Centralize resolution in a hub only when** you need (a) on-prem ⇄ cloud forwarding, (b) split answers across sovereignty boundaries, (c) cross-cloud resolution, or (d) a single audit/logging point for compliance. Hub resolvers are extra hops, extra cost, and an extra failure domain — earn them ([Route 53 Resolver](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html); [Azure DNS Private Resolver](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview)).

**Hybrid:** conditional forwarding both ways. On-prem forwards `*.cloud.example.com` to cloud resolver IPs; cloud forwards `*.corp.example.com` on-prem. Don't try to be clever with delegation across trust boundaries.

**DNSSEC.** Sign your **public** authoritative zones; validators are widely deployed at recursive resolvers and operational cost is now low at all major DNS providers ([RFC 9364 — DNSSEC BCP](https://datatracker.ietf.org/doc/html/rfc9364)). Skip signing internal private zones — the threat model is better answered by mTLS (§7).

---

## 6. Egress control

The default cloud posture — every instance can reach the entire internet on :443 — is the easy way to ship malware C2 traffic. Fix it, but cost it honestly first.

**Default posture: no public IPs on workloads.** Only edge LBs and NAT gateways have public IPs ([AWS WA — SEC05](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/sec_network_protection_create_layers.html); [Azure CAF — network security](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/network-security)).

### 6.1 Cost math

NAT and centralized firewalls are *not* cheap; the bill scales with traffic and with how many AZs you span. Approximate US-region list prices:

- **AWS NAT Gateway:** ~$0.045/hr *per AZ* + ~$0.045/GB processed. Three-AZ deployment ≈ $100/month idle, before a byte moves ([AWS NAT pricing](https://aws.amazon.com/vpc/pricing/)).
- **AWS Network Firewall:** ~$0.395/hr per endpoint + ~$0.065/GB ([AWS NFW pricing](https://aws.amazon.com/network-firewall/pricing/)).
- **Azure NAT Gateway:** ~$0.045/hr + ~$0.045/GB. **Azure Firewall Standard:** ~$1.25/hr + ~$0.016/GB ([Azure Firewall pricing](https://azure.microsoft.com/en-us/pricing/details/azure-firewall/)).
- **GCP Cloud NAT:** ~$0.044/hr per VM + ~$0.045/GB ([Cloud NAT pricing](https://cloud.google.com/nat/pricing)).
- **Cross-AZ data transfer** is the silent killer: ~$0.01/GB *each direction* on AWS between AZs in a region. Centralizing NAT in one AZ saves NAT hourly fees but doubles cross-AZ charges; per-AZ NAT is the opposite trade.

Three honest egress designs, in order of escalating control and cost:

- **NAT only.** Allows all outbound. Acceptable for dev, and for prod when paired with private endpoints for the high-value SaaS targets so the dangerous flows don't traverse it. Per-AZ for HA.
- **Centralized firewall in the hub** (AWS Network Firewall, Azure Firewall, GCP Cloud NGFW, Palo Alto/Fortinet VM-Series): FQDN allowlists, TLS inspection where lawful, full logging. Right default for prod *once* you have a hub (§3 Stage 3); premature for Stage 1.
- **Explicit forward proxy** (Squid, Envoy, Cloudflare Gateway, Zscaler): identity-aware allowlists, CONNECT logging, per-user policy. Required in regulated envs and the only design that survives the failure modes below.

### 6.2 Failure modes of FQDN-based filtering

FQDN allowlists at L3/4 firewalls work by snooping SNI/DNS and are weakened or bypassed by:

- **Shared CDN and SaaS frontends.** `*.cloudfront.net`, `*.azureedge.net`, Cloudflare hosts, etc. host thousands of tenants — allowing one allows all.
- **QUIC / HTTP/3.** SNI moves into the encrypted Initial; many firewalls fall back to allow-or-block per IP. Either decrypt or block UDP/443 ([RFC 9000 — QUIC](https://datatracker.ietf.org/doc/html/rfc9000)).
- **DoH / DoT.** Workloads (and malware) can resolve outside your resolver, defeating DNS-based policy. Block known DoH endpoints; force resolution through your resolver.
- **Encrypted Client Hello (ECH).** When negotiated, even SNI is hidden; FQDN policy degrades to IP/ASN policy ([draft-ietf-tls-esni](https://datatracker.ietf.org/doc/draft-ietf-tls-esni/)).

**Practical:** for SaaS that actually matters (object storage, secrets, source control, registries) use **private endpoints (§4)** — bypass the FQDN-filter problem entirely. Use the FQDN firewall for the long tail (package mirrors, telemetry, vendor APIs) and accept it is best-effort, not a security boundary.

**Apply BCP 38 / RFC 2827** at the edge: drop traffic whose source isn't yours. Cloud-native NAT does this implicitly ([BCP 38](https://datatracker.ietf.org/doc/html/bcp38)).

---

## 7. Zero-trust networking

**The network is not your perimeter; identity is.** BeyondCorp made this canonical and NIST codified it: every request authenticated and authorized, no implicit trust based on source IP ([BeyondCorp 2014](https://research.google/pubs/pub43231/); [NIST SP 800-207](https://csrc.nist.gov/publications/detail/sp/800-207/final)). The principles are right; the *ordering* matters.

**Adoption triggers — do these as they apply:**

- **Day one (everyone):** no public IPs on workloads; IAM-scoped service-to-service auth using cloud-native identities (IAM roles, Managed Identities, Workload Identity); TLS on every external listener.
- **First multi-team / first regulated workload:** identity-aware proxy for human access (Cloudflare Access, Tailscale, Pomerium, Google IAP) replacing shared-secret VPNs ([Cloudflare ZT](https://developers.cloudflare.com/cloudflare-one/); [Tailscale](https://tailscale.com/blog/how-tailscale-works)).
- **First Kubernetes platform / dozens of services:** workload identity via SPIFFE/SPIRE or cloud equivalent, mTLS at the mesh layer ([SPIFFE](https://spiffe.io/docs/latest/spiffe-about/overview/)).
- **Maturity target:** mTLS *everywhere*, including dev. mTLS in
  production is already a `must` per ch06 §12; this ramp is about extending
  it to non-prod and pre-platform clusters as the org matures. Don't make
  full-coverage a prerequisite for shipping; make it the destination.

For ≤5 services, TLS to the LB + IAM-authenticated APIs is a defensible posture; mesh-mTLS is overhead you don't need yet. **Don't** treat "private subnet" as a security boundary regardless — a misconfigured IAM role still walks right through.

---

## 8. Load balancing

**Pick the layer deliberately.** L4 (NLB / Standard LB / TCP-UDP LB) for throughput, preserved client IP, non-HTTP protocols. L7 (ALB / App Gateway / HTTPS LB) for HTTP routing, header dispatch, WAF, gRPC fan-out ([AWS ELB types](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html); [Azure LB choice](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/load-balancing-overview); [GCP LBs](https://cloud.google.com/load-balancing/docs/choosing-load-balancer)).

- **Health checks:** cheap, deterministic, unauthenticated. `/healthz` returns 200 fast, no DB calls. Heavier checks belong in `/readyz` ([Google SRE — LB in the Datacenter](https://sre.google/sre-book/load-balancing-datacenter/)).
- **Connection draining (deregistration delay)** on every target group. 30–60s for HTTP, longer for long-poll/websocket ([AWS deregistration delay](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#deregistration-delay)).
- **Algorithm:** prefer **least outstanding requests** over round-robin for L7 with variable latency. P2C ("power of two choices") is what most modern proxies do under the hood.

---

## 9. TLS termination

**Terminate at the edge; re-encrypt to the workload** when the workload sits behind an untrusted hop or the threat model includes lateral movement. For small single-team systems, edge-termination + IAM auth is defensible; end-to-end TLS becomes mandatory once you cross trust boundaries (multi-tenant clusters, regulated data, shared meshes).

**Public certs:** ACME via Let's Encrypt or ZeroSSL. In Kubernetes, **cert-manager** is the standard: `Issuer`/`ClusterIssuer` + `Certificate`, automatic renewal, DNS-01 for wildcards ([cert-manager](https://cert-manager.io/docs/); [RFC 8555 — ACME](https://datatracker.ietf.org/doc/html/rfc8555)). Renewal must run unattended; a cert rotated by a human is a future outage.

**Internal certs:** a private PKI rooted in your KMS (AWS Private CA, Azure Key Vault Issuer, GCP CAS), short-lived (≤24h) workload certs issued by the mesh / SPIRE. Don't reuse the public PKI internally — you will leak names into Certificate Transparency.

---

## 10. Hybrid connectivity

**Site-to-site is fine; user VPN is largely obsolete for new builds.** ExpressRoute / Direct Connect / Cloud Interconnect for the office↔cloud trunk. For users, identity-aware proxies (§7) replace the shared-tunnel pattern; legacy IPsec/SSL VPNs remain valid where regulators or deeply integrated endpoints require them ([AWS DX](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html); [ExpressRoute](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-introduction); [GCP Interconnect](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/overview)).

**Always pair the private circuit with an IPsec VPN as failover.** DX/ER outages have happened and the postmortems are public. The VPN is cheap insurance and BGP can prefer one over the other ([AWS DX resiliency](https://aws.amazon.com/directconnect/resiliency-recommendation/)).

**Use BGP for multi-path hybrid.** Static routes in a single cloud route table do work, and on some platforms support health-check-based failover (GCP custom routes with priorities, AWS Gateway Load Balancer health checks), but failover semantics are platform-specific and weaker than dynamic routing. For hybrid trunks BGP is the right answer.

---

## 11. BGP — scope and minimum literacy

BGP is the routing protocol you actually run in three places: **hybrid circuits** (Direct Connect / ExpressRoute / Interconnect), **IPsec/SD-WAN overlays**, and **third-party router appliances** in the cloud (Palo Alto, Cisco, Aviatrix, Megaport). It is *not* something you configure on a typical TGW/vWAN spoke attachment, and it is not how cloud anycast load balancers expose themselves to you.

Minimum literacy when you do touch it:

- **AS numbers:** private 16-bit `64512–65534` and 32-bit `4200000000–4294967294`. Pick from the 32-bit private range to avoid collisions. Cloud providers own well-known public ASNs (AWS 7224, Azure 12076, Google 15169 / 16550) ([RFC 6996](https://datatracker.ietf.org/doc/html/rfc6996)).
- **eBGP** between your edge and cloud/ISP; **iBGP** only if you run a routed fabric internally.
- **Path-selection knobs:** `local-preference`, AS-path prepending, MED, communities ("don't export," "prefer this circuit").
- **Failure modes:** route flapping, AS-path loops, leaking more-specifics, accepting full tables when you wanted a default. Always filter inbound and outbound; never `redistribute connected` blindly ([RFC 4271 — BGP-4](https://datatracker.ietf.org/doc/html/rfc4271)).

**Do** enable **BFD where the platform supports it** for sub-second failover on hybrid links — support varies by product and attachment type, so consult the matrix: [AWS DX BFD](https://docs.aws.amazon.com/directconnect/latest/UserGuide/bfd-enabled.html); [Azure ExpressRoute BFD](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-bfd); [GCP Cloud Router BFD](https://cloud.google.com/network-connectivity/docs/router/concepts/bfd).

---

## 12. IPv6 in 2026

**Status:** ~45% of Google traffic is IPv6, AWS has charged for public IPv4 since Feb 2024, IPv6-only EKS/AKS/GKE node pools are GA ([Google IPv6 stats](https://www.google.com/intl/en/ipv6/statistics.html); [AWS public IPv4 charge](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/); [RFC 8200](https://datatracker.ietf.org/doc/html/rfc8200)).

**Do** dual-stack new VPCs. The cloud allocator hands you a /56 (AWS) or /64s (Azure/GCP) of provider-assigned **global** IPv6; plan subnets at /64 — the IPv6 standard, SLAAC-friendly, *not* a place to be clever and split smaller ([RFC 6177](https://datatracker.ietf.org/doc/html/rfc6177); [RFC 4291](https://datatracker.ietf.org/doc/html/rfc4291)).

**ULA vs provider global IPv6.** RFC 4193 defines **Unique Local Addresses** under `fc00::/7`; in practice all locally-assigned ULAs are in `fd00::/8` (the `fc00::/8` half is reserved for a registry that was never chartered). ULAs are the IPv6 analogue of RFC 1918 ([RFC 4193](https://datatracker.ietf.org/doc/html/rfc4193)). **In cloud environments, prefer provider-assigned global IPv6** for VPC subnets: it avoids NPTv6/NAT66 entirely, cloud SGs and routing assume globally unique source addresses, and workloads can still be private — globally unique ≠ globally reachable. Reserve ULAs for genuinely on-prem-only segments or sovereignty cases.

**Dual-stack rollout caveats:**

- **Duplicated policy.** Every SG, route, NACL, and DNS entry needs an IPv6 sibling. Audit with policy-as-code (§16) so the two stacks can't drift.
- **NAT64 / DNS64** when an IPv6-only client must reach an IPv4-only service ([RFC 6146](https://datatracker.ietf.org/doc/html/rfc6146); [RFC 6147](https://datatracker.ietf.org/doc/html/rfc6147)).
- **ICMPv6 and PMTUD.** IPv6 has no in-flight fragmentation; PMTUD relies on ICMPv6 "Packet Too Big" reaching the source. **Do not** drop ICMPv6 with the same blunt rule you used for v4 ICMP — black-hole connections that hang at random sizes ([RFC 4890](https://datatracker.ietf.org/doc/html/rfc4890)).
- **NAT66 is rarely the answer.** Use edge firewalls and address-selection policy instead.

---

## 13. MTU and PMTUD

Default Ethernet MTU is 1500; cloud overlays shave bytes off. AWS supports 9001 jumbo frames intra-VPC and intra-region peering, but most internet paths and cross-cloud / VPN paths cap at 1500 or lower (IPsec ~1400). Two failure modes:

- **Black-hole MTU.** A router drops oversized packets *and* the ICMP "Fragmentation Needed" / "Packet Too Big" reply is filtered. TCP handshakes succeed; bulk transfers hang. Allow ICMP type 3 code 4 (v4) and ICMPv6 type 2 (v6) end-to-end ([RFC 8201](https://datatracker.ietf.org/doc/html/rfc8201)).
- **Encapsulation overhead.** VXLAN, GRE, IPsec, WireGuard each subtract 40–80 bytes. On overlays, set workload MTU explicitly (1450 is a common safe value) or enable **TCP MSS clamping** at the edge.

Enable jumbo frames only inside a homogeneous fabric; never across hops you don't control.

---

## 14. CDN, anycast, QUIC/HTTP3, cross-region cost

- **Put a CDN in front of any internet-facing surface.** Cloudflare, CloudFront, Front Door, Cloud CDN, Fastly. Edge TLS termination, DDoS absorption, WAF, lower egress vs cross-region direct serving.
- **Anycast** is how CDNs and global LBs route users to the nearest PoP. AWS Global Accelerator, GCP Global External LB, Azure Front Door expose anycast IPs as managed services — you don't run BGP yourself ([AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html); [GCP network tiers](https://cloud.google.com/network-tiers/docs/overview)).
- **QUIC / HTTP/3** (RFC 9000, RFC 9114) is widely deployed at major CDNs and improves tail latency on lossy networks. Enable on your CDN; for origin protocols the gain is smaller and the ops cost (UDP/443 through middleboxes, observability gaps) is real ([RFC 9114](https://datatracker.ietf.org/doc/html/rfc9114)).
- **Cross-region peering vs transit.** VPC peering between regions charges inter-region data transfer (~$0.02/GB on AWS, similar elsewhere); the same traffic via a transit hub adds the hub's per-GB on top. Few region pairs: peering is cheaper. Many regions plus central inspection: transit wins despite the per-GB premium. Measure before defaulting.

---

## 15. Service mesh (cross-ref ch04 §9)

**Mesh adoption is decided in ch04 §9** — that section owns the default
("usually no") and the concrete triggers. This chapter only covers what a
mesh gives you at the L7 networking layer once you have one. Cilium (eBPF,
fastest data path, doubles as your CNI), Linkerd (simplest, Rust micro-proxy),
and Istio ambient (most features) are the live options ([Cilium](https://docs.cilium.io/); [Linkerd](https://linkerd.io/2/reference/architecture/); [Istio ambient](https://istio.io/latest/docs/ambient/)).

**Mesh-managed mTLS beats infra-mTLS** when (a) you have ≥dozens of services, (b) churn is high, or (c) you want SPIFFE identity for policy. For ≤5 services, two well-tuned LB listeners and cert-manager are simpler. The must/ramp posture for mTLS itself is in ch06 §12 and ch07 §7.

**Gateway API** is the upstream-blessed successor to Ingress for new platform work; Ingress remains supported but feature-frozen ([Gateway API](https://gateway-api.sigs.k8s.io/)).

---

## 16. Firewall-as-code

Network policy is configuration; treat it like code or it will rot.

- **Source of truth in git.** Terraform / OpenTofu / Pulumi / Bicep for cloud primitives (SGs, NSGs, firewall rules, route tables, endpoints). Kubernetes NetworkPolicy / CiliumNetworkPolicy as YAML alongside workloads.
- **Policy-as-code at admission.** OPA/Gatekeeper or Kyverno to forbid `0.0.0.0/0` ingress, mandate private endpoints for tagged services, enforce paired v4/v6 rules.
- **Continuous diff.** Terraform plan in CI on a schedule; AWS Config / Azure Policy / GCP Org Policy catches console-clicks before they become incidents.
- **Pre-merge tests.** `tflint`, `checkov`, `tfsec`, plus a reachability prover (AWS VPC Reachability Analyzer, Azure Network Watcher) on critical paths.

---

## 17. Anti-patterns

- **VPN-as-perimeter for users.** One stolen laptop = full reach. → Identity-aware proxy.
- **"WAF later".** The vuln you ship today is what the WAF would have caught. → WAF day one at the edge LB or CDN.
- **Private subnet ⇒ secure.** No IAM ⇒ any compromised pod owns it. → Defence in depth: SG + IAM + mTLS + private endpoints.
- **Overlapping CIDRs.** Future peering/TGW/VPN merges fail; renumbering prod is a career event. → Central IPAM, one plan.
- **Floating `0.0.0.0/0` egress in prod.** Free C2 channel for any RCE. → Private endpoints for high-value SaaS + FQDN/proxy for the long tail.
- **Mesh peering at scale.** N² connections, untraceable routing. → Move to hub-and-spoke when networks > ~10 or hybrid arrives.
- **Plaintext inside, across trust boundaries.** → End-to-end TLS, mesh-managed.
- **Health check that hits the DB.** First DB blip ⇒ fleet unhealthy. → `/healthz` cheap & local, `/readyz` for deps.
- **Default VPC in production.** Overlapping CIDRs, wide-open SGs. → Delete it; explicit VPCs only.
- **Static routes on hybrid links.** Failover semantics are weak or platform-specific. → BGP + BFD where supported.
- **Public IP on every VM.** → No public IPs; egress via NAT/firewall, ingress via LB only.
- **Dropping all ICMP/ICMPv6.** Breaks PMTUD; hangs random transfers. → Allow PTB / "frag needed" end to end.

---

## 18. Network observability (cross-ref ch08)

**You cannot debug what you cannot see.** Minimum:

- **VPC/NSG flow logs** sampled to a queryable store ([AWS](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html); [Azure](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-nsg-flow-logging-overview); [GCP](https://cloud.google.com/vpc/docs/flow-logs)).
- **DNS query logs** at the resolver — the highest-signal log in any network incident.
- **L7 access logs** at LB and mesh with trace IDs propagated.
- **eBPF tooling** (Cilium Hubble, Pixie) for in-cluster visibility — kernel-level, no sidecars ([Cilium Hubble](https://docs.cilium.io/en/stable/observability/hubble/)).

---

## TL;DR

- Plan IP space in IPAM. /16 per VPC, /24 per subnet (5 reserved by major clouds, 4 on GCP), never overlap, never reshape. RFC 6598 only as a documented exception.
- Stage-gate topology: single VPC → peering/shared VPC → transit hub. Don't buy a hub before you have hybrid, central inspection, or many networks.
- No public IPs on workloads. Cost the egress design (hourly + per-GB + cross-AZ) before picking it. Private endpoints for the SaaS that matters; FQDN policy is best-effort for the rest.
- PrivateLink / Private Endpoint / PSC differ in DNS, transitivity, and pricing — pick from §4 matrix.
- Split-horizon DNS with native zone association by default; hub resolvers only when hybrid, sovereignty, cross-cloud, or central audit demands them. DNSSEC on public zones.
- Zero-trust by adoption trigger: no public IPs and IAM auth on day one; mesh-mTLS as a maturity target.
- TLS at the edge, end-to-end across trust boundaries. cert-manager + ACME, no human in renewal.
- BGP scope: hybrid circuits, IPsec/SD-WAN, router appliances. Enable BFD where the platform supports it.
- Dual-stack IPv6 on new VPCs in 2026; provider global IPv6 by default, ULA (`fd00::/8` under RFC 4193 `fc00::/7`) only for on-prem-only segments. Don't drop ICMPv6.
- Mind MTU and PMTUD; allow PTB. CDN in front of internet surfaces.
- Flow logs + DNS logs + L7 access logs, or you are flying blind.
