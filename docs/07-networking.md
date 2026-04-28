# Chapter 07 — Networking

Opinionated, cloud-agnostic networking defaults for 2026. The thesis: the
network is no longer your security perimeter, but it is still your blast-radius
boundary, your cost boundary, and the thing that wakes you at 3 a.m. when DNS
breaks. Treat IP space like a public API — once allocated, never reshape — and
treat trust as a transport-layer concern: every flow authenticated and
encrypted, no implicit "we're inside the VPC so it's fine."

> Conventions: **Do** = required default. **Don't** = reject in review unless a
> written exception exists. **Prefer** = strong default; deviate only with
> measurement or a documented constraint. Vendor parallels are stated
> explicitly because the primitives converge but the names do not.

---

## 1. Cloud network primitives

The mental model is the same across clouds; only the names change. Learn the
mapping once and stop translating in your head.

| Concept | AWS | Azure | GCP |
| --- | --- | --- | --- |
| Isolated L3 network | VPC | VNet | VPC (global!) |
| Subnet (broadcast/AZ scope) | Subnet (AZ-scoped) | Subnet (region-scoped) | Subnet (region-scoped) |
| Default gateway out | Internet Gateway | (implicit, via UDR/NAT) | Default internet GW |
| Source NAT for private hosts | NAT Gateway | NAT Gateway / Azure NAT | Cloud NAT |
| Routing | Route Table | Route Table / UDR | VPC Routes |
| Cross-network L3 stitch | VPC Peering / TGW | VNet Peering / vWAN | VPC Peering / NCC |
| Private SaaS access | PrivateLink (Endpoint) | Private Endpoint | Private Service Connect |
| Hybrid private circuit | Direct Connect | ExpressRoute | Cloud Interconnect |

Two non-obvious facts that bite teams new to a given cloud:

1. **AWS subnets are AZ-scoped; Azure and GCP subnets are region-scoped.** AWS
   forces you to allocate one subnet per AZ per tier (so a 3-AZ "private" tier
   is *three* subnets); Azure/GCP give you one subnet that spans the region's
   zones ([AWS VPC subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html);
   [Azure VNet overview](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview);
   [GCP VPC overview](https://cloud.google.com/vpc/docs/vpc)). Capacity math
   below assumes the AWS shape because it is the strictest.
2. **GCP VPCs are global, not regional.** A single GCP VPC spans every region
   you turn on, with regional subnets inside it. This collapses a lot of
   peering work that AWS/Azure require but also means a misconfigured firewall
   rule has a planetary blast radius — review them like IAM
   ([GCP VPC overview](https://cloud.google.com/vpc/docs/vpc)).

---

## 2. IP address planning

**Allocate a /16 per environment per region per account, with /24 subnets.**
A /16 gives you 256 /24s; a /24 gives you 251 usable hosts per subnet (cloud
providers reserve 5: network, broadcast, gateway, DNS, future). This is the
"never paint yourself into a corner" default
([AWS VPC sizing](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html);
[Azure subnet sizing](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq#how-small-and-how-large-can-vnets-and-subnets-be);
RFC 1918 §3).

**Use RFC 1918 space with a global allocation plan, written down, before the
first VPC is created.** Workable scheme: reserve a /10 (`10.0.0.0/10` = 4M
addresses) per cloud account / GCP project / Azure subscription, then carve
/16s out of it per VPC. Reserve `100.64.0.0/10` (RFC 6598 CGNAT) for things
like EKS pod CIDRs and Tailscale that must not collide with VPC RFC 1918
space ([RFC 6598](https://datatracker.ietf.org/doc/html/rfc6598)). Document
in IPAM (AWS VPC IPAM, Azure VNet Manager, NetBox) — the first time you
merge two acquired companies you will be grateful.

**Never overlap CIDRs across anything you might one day connect.** Peering,
VPN, Transit Gateway, and ExpressRoute all refuse overlapping ranges, and
the "fix" is renumbering production. AWS Well-Architected Reliability calls
this the most common multi-account network failure
([AWS WA — REL02-BP02](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_network_topology_ip_address_management.html)).

**Per-AZ /24s, three AZs, two tiers (public/private), per VPC** is a sane
default starter. That's six /24s out of 256 — leaves 250 for growth, sidecars,
EKS pod CIDRs, future tiers. Resist packing a /22 into a /23 to "save space."
There is no scarcity inside RFC 1918.

**Don't** use the cloud provider's default VPC for anything real. Delete it
or quarantine it; it has overlapping CIDRs across regions and a wide-open SG
by design ([AWS default VPC](https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html)).

---

## 3. Topology: hub-and-spoke wins at scale

Three honest topologies, in order of how far they take you:

1. **Single VPC.** Up to ~5 services, one team. Simpler than anyone admits.
2. **Mesh peering.** N×(N−1)/2 peerings. Toxic past ~6 VPCs. Avoid as target.
3. **Hub-and-spoke** with shared transit (AWS Transit Gateway, Azure Virtual
   WAN, GCP Network Connectivity Center). One inspection point, one routing
   domain, additive growth. **Default for any org with >1 account/sub/project.**
   ([AWS TGW](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html);
   [Azure vWAN](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-about);
   [GCP NCC](https://cloud.google.com/network-connectivity/docs/network-connectivity-center/concepts/overview)).

**Adopt a Landing Zone model from day one.** Microsoft's Cloud Adoption
Framework "Azure landing zones" and AWS Control Tower / "Multi-Account
Strategy" prescribe the same shape: a `connectivity` subscription/account
hosting the hub, `identity` and `management` accounts at the org root, and
per-workload spoke accounts that *only* connect outward through the hub
([Microsoft CAF](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/);
[AWS Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html);
[AWS multi-account](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html)).
Retro-fitting this is the most expensive infra migration most orgs ever do.

---

## 4. Private connectivity & the convergence pattern

The 2026 reality: **PrivateLink / Private Endpoint / Private Service Connect
have converged into the same idea** — expose a service via a private IP inside
the consumer's VPC, no peering, no overlap risk, unidirectional. Use it for
everything that supports it.

**Do** terminate every cloud-managed PaaS over a private endpoint. Storage,
databases, queues, key vaults. The provider-default public endpoint is
convenient and a data-exfiltration vector
([AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html);
[Azure Private Endpoint](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview);
[GCP Private Service Connect](https://cloud.google.com/vpc/docs/private-service-connect)).

**Prefer** PrivateLink-style endpoints over VPC peering for service-to-service
across orgs/accounts. Peering connects *networks* (full L3 reach, both
directions, CIDR overlap forbidden); PrivateLink connects a *single service*
(unidirectional, no CIDR coupling). Use Transit Gateway / vWAN / NCC for
spoke-to-spoke; never mesh-peer spokes. Set the routing domain so prod can't
reach dev unless explicitly permitted. The hub is the right place for
centralized egress inspection (next section).

---

## 5. DNS

DNS is the one thing that, when broken, breaks everything else with no useful
error message. Treat it like a tier-0 dependency.

**Run split-horizon DNS by default.** Public zones for what the internet sees;
private zones (Route 53 Private Hosted Zones, Azure Private DNS, Cloud DNS
private zones) for everything inside the VPC. Same name, different answers
([AWS Route 53 private zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html);
[Azure Private DNS](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview);
[Cloud DNS private zones](https://cloud.google.com/dns/docs/zones)). PrivateLink
endpoints *require* this — public DNS for `*.s3.amazonaws.com` resolves to the
public service unless a private zone overrides it.

**Centralize private DNS in the hub VNet/VPC.** Spokes resolve via conditional
forwarding rules pointed at the hub's resolver (Route 53 Resolver endpoints,
Azure DNS Private Resolver, Cloud DNS forwarding zones). One source of truth,
one place to audit. **For hybrid:** conditional forwarding both ways. On-prem
forwards `*.cloud.example.com` to cloud resolver IPs; cloud forwards
`*.corp.example.com` on-prem. Don't try to be clever with delegation across
trust boundaries. **Don't** put service-discovery records in your registrar's
zone — use the private zone the workload owns.

---

## 6. Egress control

The default cloud posture — every instance can reach the entire internet on
:443 — is the easy way to ship malware C2 traffic. Fix it.

**Default posture: no public IPs on workloads.** Only edge load balancers and
NAT gateways have public IPs. Everything else is private + egresses through a
controlled chokepoint. AWS Well-Architected Security calls this out as a
foundational control
([AWS WA — SEC05-BP01](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/sec_network_protection_create_layers.html);
[Azure CAF — secure landing zone](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/network-security)).

Three honest egress designs, in order of escalating control:

1. **NAT gateway only.** Cheap, allows *all* outbound. Acceptable for dev.
2. **Centralized firewall in the hub** (AWS Network Firewall, Azure Firewall,
   GCP Cloud NGFW, or Palo Alto VM-Series): FQDN-allowlisted egress, TLS
   inspection where lawful, logged. **Default for prod.**
3. **Explicit forward proxy** (Squid, Envoy, Cloudflare Gateway) with
   identity-aware allowlists and CONNECT logging. Required in regulated envs.

**Apply BCP 38 / RFC 2827 egress filtering** at the edge: drop traffic whose
source address is not yours. Cloud-native NAT does this implicitly; on-prem
or self-managed edges must enforce it
([BCP 38](https://datatracker.ietf.org/doc/html/bcp38)).

**Don't** rely on security groups alone for egress. They are stateful L4 ACLs
without name resolution; an attacker with code execution can still reach any
allowed CIDR. Pair SGs with FQDN policy at the firewall.

---

## 7. Zero-trust networking

**The network is not your perimeter; identity is.** Google's BeyondCorp papers
made this canonical a decade ago and the rest of the industry has caught up:
every request authenticated, every request authorized, no implicit trust
based on source IP
([BeyondCorp 2014](https://research.google/pubs/pub43231/);
[BeyondCorp 2018](https://research.google/pubs/pub47356/);
[NIST SP 800-207](https://csrc.nist.gov/publications/detail/sp/800-207/final)).

**Do** mTLS every service-to-service call. SPIFFE/SPIRE for workload identity
is the cleanest standard; service meshes (Istio, Linkerd, Cilium) hand you
mTLS without app changes ([SPIFFE](https://spiffe.io/docs/latest/spiffe-about/overview/);
cross-ref ch04 §service mesh). **Do** put user-facing apps behind an
identity-aware proxy (Cloudflare Access, Tailscale, Pomerium, IAP) — the
BeyondCorp model in a box, replacing the corporate VPN-for-users pattern
([Cloudflare Zero Trust](https://www.cloudflare.com/learning/security/glossary/what-is-zero-trust/);
[Tailscale](https://tailscale.com/blog/how-tailscale-works)). **Don't** treat
"private subnet" as a security boundary. A misconfigured IAM role still walks
right through. Defence in depth, not defence by obscurity.

---

## 8. Load balancing

**Pick the layer deliberately.** L4 (NLB / Standard LB / TCP/UDP LB) for raw
throughput, preserve client IP, non-HTTP protocols. L7 (ALB / App Gateway /
HTTPS LB) for HTTP routing, header-based dispatch, WAF integration, gRPC
fan-out
([AWS ELB types](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html);
[Azure LB vs App Gateway](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/load-balancing-overview);
[GCP load balancers](https://cloud.google.com/load-balancing/docs/choosing-load-balancer)).

**Health checks must be cheap, deterministic, and unauthenticated.** Hit
`/healthz`, return 200 fast, no DB calls. Anything heavier belongs in
`/readyz`. Google SRE has a chapter's worth of war stories about health
checks that took down the very thing they monitored
([Google SRE — LB in the Datacenter](https://sre.google/sre-book/load-balancing-datacenter/);
[SRE Workbook — SLOs](https://sre.google/workbook/implementing-slos/)).

**Enable connection draining (deregistration delay) on every target group.**
Default 30–60s for HTTP, longer for long-poll/websocket. Without it, every
deploy returns 502s for healthy traffic mid-flight
([AWS deregistration delay](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#deregistration-delay)).

**Algorithm:** prefer **least outstanding requests** over round-robin for L7
services with variable latency. Round-robin is fine for homogeneous
short-request workloads. P2C ("power of two choices") is what most modern
proxies actually do under the hood.

---

## 9. TLS termination strategy

**Terminate at the edge, re-encrypt to the mesh.** "TLS to the LB and HTTP
inside" is the 2015 answer; in zero-trust the inside hop must also be TLS,
terminated again at the workload (or mesh sidecar). The double-handshake cost
is small ([Google SRE Workbook](https://sre.google/workbook/);
[Cloudflare — mTLS](https://www.cloudflare.com/learning/access-management/what-is-mutual-tls/)).

**Public certs:** ACME via Let's Encrypt or ZeroSSL. In Kubernetes,
**cert-manager** is the standard — `Issuer`/`ClusterIssuer` + `Certificate`
resources, automatic renewal, DNS-01 for wildcards
([cert-manager](https://cert-manager.io/docs/);
[Let's Encrypt 2020 root expiry postmortem](https://letsencrypt.org/2020/12/21/extending-android-compatibility.html)).
Renewal must run unattended; a cert rotated by a human is a future outage.

**Internal certs:** a private PKI rooted in your KMS (AWS Private CA, Azure
Key Vault Issuer, GCP CAS), short-lived (≤24h) workload certs issued by the
mesh / SPIRE. Don't reuse the public PKI internally — you will leak names
into Certificate Transparency. **Don't** terminate TLS in your application
unless you have a reason; the LB and mesh do it better.

---

## 10. Hybrid connectivity

**Site-to-site is fine; user VPN is dead.** ExpressRoute / Direct Connect /
Cloud Interconnect for the office↔cloud trunk; replace the user VPN with an
identity-aware proxy (§7)
([AWS Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html);
[Azure ExpressRoute](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-introduction);
[GCP Cloud Interconnect](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/overview)).

**Always pair the private circuit with an IPsec VPN as failover.** Direct
Connect / ExpressRoute outages have happened; the postmortems are public. The
VPN is cheap insurance and BGP can prefer one over the other
([AWS DX resiliency recommendations](https://aws.amazon.com/directconnect/resiliency-recommendation/)).

**Use BGP for everything multi-path.** Static routes don't fail over.

---

## 11. BGP basics every cloud network engineer needs

You don't need to be a CCIE, but in 2026 BGP is on every hybrid edge, every
TGW/vWAN attachment, every anycast load balancer, and every K8s-on-bare-metal
CNI worth using. Minimum literacy:

- **AS numbers:** private ASNs `64512–65534` (16-bit) and `4200000000–
  4294967294` (32-bit, RFC 6996). Pick from the 32-bit private range to avoid
  collisions. Cloud providers each own well-known public ASNs (AWS 7224,
  Azure 12076, Google 15169 / 16550).
- **eBGP vs iBGP:** eBGP between your edge and the cloud / ISP; iBGP if you
  run a routed fabric internally. Most cloud-only orgs only ever speak eBGP.
- **Path selection knobs you actually use:** `local-preference` (prefer one
  exit, in-AS), `AS-path prepending` (make a route look worse), `MED`
  (suggest preference to a neighbor — they may ignore it), communities
  ("don't export," "prefer this circuit").
- **Failure modes:** route flapping, AS-path loops, leaking more-specifics,
  accepting full tables when you wanted a default. Always filter inbound and
  outbound; never `redistribute connected` blindly into BGP
  ([RFC 4271 — BGP-4](https://datatracker.ietf.org/doc/html/rfc4271);
  Kurose & Ross, *Computer Networking*, ch. 5).

**Do** run BFD with BGP for sub-second failover on hybrid links. **Don't**
expose your edge router with default 90s hold timers and call it HA.

---

## 12. IPv6 in 2026

**Status check:** ~45% of Google traffic is IPv6, AWS charges for public IPv4
since Feb 2024, and IPv6-only EKS/AKS/GKE node pools are GA. The "we'll do
IPv6 later" excuse has expired
([Google IPv6 stats](https://www.google.com/intl/en/ipv6/statistics.html);
[AWS public IPv4 charges](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/);
[RFC 6177](https://datatracker.ietf.org/doc/html/rfc6177);
[RFC 8200](https://datatracker.ietf.org/doc/html/rfc8200)).

**Do** dual-stack new VPCs. The cloud allocator hands you a /56 (AWS) or /64s
(Azure/GCP); plan subnets at /64 — the IPv6 standard, SLAAC-friendly, *not* a
place to be clever and split smaller.

**Prefer ULA (RFC 4193) `fd00::/8` for "private" intent**, but in IPv6 the
modern position is "global addresses + edge firewall" — NAT is no longer the
privacy mechanism it was in v4. **Don't** translate IPv6 with NAT66 unless
you genuinely have no choice.

---

## 13. Service mesh networking (cross-ref ch04)

**Use a mesh when you need workload-identity mTLS, L7 traffic policy, or
egress identity** — not because it is fashionable. Cilium (eBPF, fastest
data path, doubles as your CNI), Linkerd (simplest, Rust micro-proxy), Istio
ambient (most features) are the live options
([Cilium](https://docs.cilium.io/);
[Linkerd](https://linkerd.io/2/reference/architecture/);
[Istio ambient](https://istio.io/latest/docs/ambient/);
[CNCF Landscape](https://landscape.cncf.io/)).

**Mesh-managed mTLS beats infra-mTLS** when (a) you have ≥dozens of services,
(b) churn is high, or (c) you want SPIFFE identity for policy. For ≤5
services, two well-tuned LB listeners and cert-manager are simpler.

**Gateway API > Ingress** for new platform work — Ingress is frozen, Gateway
API is the upstream-blessed successor and the basis of every modern mesh
north-south story
([Gateway API](https://gateway-api.sigs.k8s.io/);
[ThoughtWorks Tech Radar](https://www.thoughtworks.com/radar)).

---

## 14. Anti-patterns

| Anti-pattern | Why it's wrong | Do this instead |
| --- | --- | --- |
| VPN-as-perimeter for users | Single shared trust zone; one stolen laptop = full reach. | Identity-aware proxy (Tailscale, Cloudflare Access, IAP). |
| "We'll add a WAF later" | The vuln you ship today is what the WAF would have caught. | WAF on day one at the edge LB. |
| Private subnet ⇒ "secure" | No IAM ⇒ any compromised pod owns it. | Defence in depth: SG + NACL + IAM + mTLS. |
| Overlapping CIDRs across accounts | Future peering / TGW / VPN merge fails. Renumbering prod = career event. | Central IPAM, allocate from a single plan. |
| Floating `0.0.0.0/0` egress | Free C2 channel for any RCE. | FQDN-allowlist firewall in the hub. |
| Mesh peering at scale | N² connections, untraceable routing. | Hub-and-spoke via TGW / vWAN / NCC. |
| TLS to the LB, HTTP inside | Inside is the new outside. | End-to-end TLS, mesh-managed if available. |
| Health check that hits the DB | First DB blip ⇒ entire fleet marked unhealthy ⇒ outage. | `/healthz` cheap & local, `/readyz` for deps. |
| Default VPC in production | Pre-created with overlapping CIDRs and wide-open SGs. | Delete it; explicit VPCs only. |
| Unnumbered subnets | Every change becomes archaeology. | IPAM with owner, purpose, expiry per range. |
| Static routes on hybrid links | Silent black-hole on link flap. | BGP + BFD on every hybrid attachment. |
| Public IP on every VM | One open SG = breach. | No public IPs; egress via NAT/firewall, ingress via LB only. |

---

## 15. Network observability (cross-ref ch08)

**You cannot debug what you cannot see.** Minimum: VPC/NSG flow logs sampled
to a queryable store ([AWS VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)),
DNS query logs at the resolver (the highest-signal log in any network
incident), and L7 access logs at LB and mesh with trace IDs propagated.
eBPF tooling (Cilium Hubble, Pixie) is the right default for in-cluster
visibility — kernel-level, no sidecars
([Cilium Hubble](https://docs.cilium.io/en/stable/observability/hubble/)).

---

## TL;DR

- Plan IP space in IPAM. /16 per VPC, /24 per subnet, never overlap, never reshape.
- Hub-and-spoke from day one; mesh peering never.
- No public IPs on workloads. Egress through a hub firewall with FQDN policy.
- Private endpoints for every PaaS that supports them.
- Split-horizon DNS centralized in the hub; conditional forwarding for hybrid.
- mTLS everywhere; identity-aware proxy for users; site-to-site VPN/circuit for offices.
- TLS at edge *and* mesh; cert-manager + ACME, no human in the renewal loop.
- BGP + BFD on hybrid links; static routes don't fail over.
- Dual-stack IPv6 on new VPCs in 2026.
- Flow logs + DNS logs + L7 access logs, or you are flying blind.
