# Chapter 04 — Containers & Kubernetes

Opinionated, cloud-agnostic defaults for running container workloads in
2026. Most of this chapter is about *not* over-engineering: K8s is the
right answer less often than the industry believes.

> Conventions: **Do** = required default. **Don't** = reject in review
> unless a written exception exists. **Prefer** = strong default; deviate
> only with measurement or a documented constraint.

---

## 1. Do you need Kubernetes at all?

**Default to "no" until you can name three workloads that need it.**
Kubernetes is a platform for building platforms; for a single team running
fewer than ~30 services, the operational tax (control-plane upgrades,
CNI/CSI churn, RBAC, capacity planning) usually exceeds the benefit
([CNCF — Kubernetes overview](https://kubernetes.io/docs/concepts/overview/);
*Production Kubernetes*, Dotson et al., O'Reilly 2021, ch. 1).

Decision tree:

- One stateless HTTP service + managed DB → **Cloud Run / Azure Container Apps / AWS App Runner**.
- ≤10 services, event-driven, scale-to-zero matters → **ACA / Cloud Run / Fargate, or Knative on a managed cluster**.
- Heavy stateful, multi-tenant, custom networking, GPU pools, or a platform team → **managed Kubernetes (AKS/EKS/GKE)**.
- Air-gapped / regulated / per-pod hardware control → self-hosted K8s. The only honest reason to run kubeadm in 2026.

> **Smell test:** if `kubectl get pods -A` fits on one screen *and* you
> have no platform engineer, you are paying the K8s tax for nothing.

---

## 2. Container image hygiene

**Use a minimal, distroless or Wolfi-based runtime image.** No shell, no
package manager, no `curl` in production
([Google distroless](https://github.com/GoogleContainerTools/distroless);
[Chainguard Images docs](https://edu.chainguard.dev/chainguard/chainguard-images/);
Liz Rice, *Container Security*, O'Reilly 2020, ch. 6).

```dockerfile
FROM golang:1.23 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

- **Do** use multi-stage builds; the runtime stage never sees build tools.
- **Do** run as a non-root UID; PSA `restricted` requires `runAsNonRoot: true` ([K8s — Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).
- **Don't** ship `:latest`; pin by digest (`@sha256:...`).
- **Don't** install a shell "for debugging" — use `kubectl debug` ephemeral containers ([K8s — ephemeral containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)).

**Build with BuildKit and produce SBOMs at build time.** `docker buildx build --sbom=true --provenance=mode=max --platform=linux/amd64,linux/arm64` produces multi-arch images plus SLSA provenance and an SPDX SBOM ([BuildKit attestations](https://docs.docker.com/build/attestations/)). Cosign-sign the *digest*; verification on admission is owned by **ch06** (signing / SLSA / verifyImages / Connaisseur policy live there).

**Ship large auxiliary payloads as separate OCI artifacts, not baked into the runtime image.** ML models, datasets, language packs, and WASM plugins should mount via the `image` volume source (alpha v1.31, **beta v1.33**, KEP-4639) ([K8s — image volume](https://kubernetes.io/docs/concepts/storage/volumes/#image)). Keeps the runtime image small and distroless, lets you cosign-sign and version the artifact independently, and lets you swap a model without rebuilding the app image.

---

## 3. Registries

- **Do** scan on push *and* on a daily schedule — new CVEs land against old images ([Trivy](https://trivy.dev/); [Grype](https://github.com/anchore/grype)).
- **Do** run a pull-through cache (Harbor / ACR / ECR / Artifact Registry remote repos) per region: cuts cold-start latency, defeats Docker Hub rate-limits, gives you one chokepoint for vulnerability gates.
- **Do** set retention: keep last N tags + anything signed in the last 90 days; everything else is GC'd. Untagged layers are the #1 registry cost driver.
- **Don't** allow direct `docker pull docker.io/...` from cluster nodes — proxy it. (Image policy / signature verification owned by ch06.)

---

## 4. The 12-factor app, revisited

[12factor.net](https://12factor.net) still holds:

- **Config in env vars or projected files**, never baked into the image.
- **Stateless processes.** Anything surviving a pod restart lives in a DB, object store, or cache.
- **Logs to stdout/stderr.** Platform ships them. No log files, no in-process shippers (ch06).
- **Disposability.** Boot in <10s, shut down on SIGTERM cleanly. Use `terminationGracePeriodSeconds`.
- **Dev/prod parity.** Same base image, runtime, config keys.

---

## 5. Resource requests, limits, and QoS

This is the section most teams get wrong. **Precondition:** assume cgroup v2 nodes (default on Bottlerocket, Talos, Flatcar, AL2023, Ubuntu 22.04+); MemoryQoS, modern eviction, and in-place resize semantics described here behave subtly differently — or not at all — on cgroup v1, which is end-of-life on every supported node OS.

**Do** set CPU and memory **requests > 0** on every container. Without
requests the scheduler treats the pod as zero-cost and packs it onto any
node ([K8s — Resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/);
*Production Kubernetes*, ch. 7).

**Do** set a **memory limit** on every container. It caps blast radius —
if a container exceeds it, the kernel cgroup OOM-kills *that container*
only. Distinguish two failure modes that get confused:

- **Container OOMKill** — one container exceeds its memory limit; kernel kills it; siblings unaffected.
- **Kubelet node-pressure eviction** — node-level memory/disk/PID pressure makes the kubelet evict pods *before* the kernel OOMs the node. Pods using less than their **request** are protected; pods over request are evicted first, ordered by QoS (`BestEffort` → `Burstable` → `Guaranteed`) ([K8s — Node-pressure eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#pod-selection-for-kubelet-eviction); [Linux cgroup v2 memory controller](https://www.kernel.org/doc/Documentation/admin-guide/cgroup-v2.rst)).

So memory **limits** cap blast radius per container; memory **requests**
(and QoS) drive eviction priority. Neither makes failure clean — in-flight
requests still die. Size limits from observed RSS + headroom.

**Default to Burstable** for application workloads: CPU and memory
**requests**, memory **limit**, **omit the CPU limit**. CFS throttling
spikes p99 latency on bursty workloads even with idle node CPU; correctly
sized requests deliver proportional share without that pathology
([K8s — CPU management policies](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/);
[kernel.org — CFS bandwidth control](https://docs.kernel.org/scheduler/sched-bwc.html)).
Use CPU limits only for batch, untrusted multi-tenant, or measured
noisy-neighbour cases.

**Reserve `Guaranteed` QoS** (requests == limits on CPU *and* memory) for
workloads that intentionally accept CPU caps — typically pods needing
static CPU pinning or that must never be evicted under node pressure
([K8s — Pod QoS classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/);
[K8s — CPU manager static policy](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy)).
You trade burst headroom for predictability; most services don't want
that. QoS summary:

- `Guaranteed` — requests == limits on CPU and memory. Last-evicted; eligible for static CPU pinning.
- `Burstable` — requests set, memory limit set, CPU limit usually unset. The right default.
- `BestEffort` — nothing set. Never in production.

**Right-size from data, and prefer in-place resize over rolling restarts when you can.** Run VPA in **recommendation-only mode** (`updateMode: Off`); auto-mode together with HPA on the same metric breaks both ([VPA repo](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler); *Kubernetes Patterns*, Ibryam & Huss, 2nd ed., O'Reilly 2023, ch. 2). On v1.33+, **in-place pod resize is on by default** (KEP-1287, beta): patch CPU/memory requests and limits on a running pod via the new `resize` subresource (`kubectl patch pod <name> --subresource resize`) without a restart, which finally makes VPA recommendations safe to apply to long-running stateful or warmed-cache workloads ([K8s v1.33 — In-Place Pod Resize Beta](https://kubernetes.io/blog/2025/05/16/kubernetes-v1-33-in-place-pod-resize-beta/)). Watch the new `PodResizePending` (Deferred / Infeasible) and `PodResizeInProgress` conditions; native sidecars (§6.1) can also be resized in place.

---

## 6. Probes and lifecycle

**Three probes, three jobs** ([K8s — Configure probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)):

- **`startupProbe`** — "is the app done booting?" Generous `failureThreshold`. Disables the others until it passes. Use whenever cold start > a few seconds.
- **`readinessProbe`** — "should I be in Service endpoints right now?" Cheap, frequent, *no external dependencies*. A readiness probe that checks the DB will cascade-fail your fleet when the DB blips.
- **`livenessProbe`** — "am I wedged?" Used sparingly. If in doubt, omit it; a bad liveness probe is worse than none (*Production Kubernetes*, ch. 9).

### 6.1 Native sidecar containers (1.29 beta, 1.33 GA)

KEP-753 makes sidecars a first-class pod construct: declare them as `initContainers` with `restartPolicy: Always` and the kubelet guarantees they start *before* app containers, run for the whole pod lifetime, get aligned OOM-score so they aren't preempted before the app, support all three probes, and **terminate after** the main containers exit ([K8s v1.33 release — Sidecar containers stable](https://kubernetes.io/blog/2025/04/23/kubernetes-v1-33-release/); [KEP-753](https://kep.k8s.io/753)). Use this for meshes, log shippers, secret agents, OPA/Kyverno injectors, and DB proxies — it removes the entire class of "app exited but sidecar still holding the pod open" race.

```yaml
spec:
  initContainers:
    - name: envoy
      image: envoyproxy/envoy:v1.32.0
      restartPolicy: Always   # makes this a native sidecar
      readinessProbe: { httpGet: { path: /ready, port: 15021 } }
  containers:
    - name: app
      image: registry.example.com/app@sha256:...
```

With native sidecars the previous `preStop` Envoy-drain trick is no longer needed: when the app container exits, the kubelet sends SIGTERM to the sidecar, which can drain on its own. Reserve `preStop` for sidecars or apps you can't modify.

### 6.2 Graceful shutdown

**Graceful shutdown is an app responsibility, not a `sleep` workaround.** On pod deletion the kubelet sends SIGTERM while endpoint controllers begin removing the pod from EndpointSlices; propagation to every kube-proxy / LB / ingress controller is not instantaneous, so for a brief window new requests can still land on a terminating pod ([K8s — Pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination); [K8s — EndpointSlice conditions](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/#conditions)). Robust sequence:

- **Fail readiness first.** On SIGTERM, flip `/readyz` to fail so endpoint controllers stop sending new traffic; the kubelet keeps the pod `terminating` while in-flight work finishes.
- **Drain in-flight work in the app.** Stop new connections, finish outstanding requests, close pools, flush queues, exit 0. Set `terminationGracePeriodSeconds` ≥ real drain time + buffer.
- **Coordinate with external load balancers** that hit pod IPs directly (AWS NLB/ALB, Azure LB, GKE container-native LB). Use the controller's drain hook (e.g., AWS LB controller's `deregistration_delay.timeout_seconds`) instead of a blind sleep.
- **Use `preStop` only for specific drain coordination** the app can't do. A blanket `preStop: ["sleep", "10"]` is a smell: it delays SIGTERM without proving the pod drained, and hides shutdown bugs.

### 6.3 PDBs and spread

Every Deployment with replicas > 1 ships a `PodDisruptionBudget` (`maxUnavailable: 1`) and `topologySpreadConstraints` across zones. Without a PDB, a node drain can take the whole service down ([K8s — PDB](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/); [K8s — topologySpreadConstraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)). Prefer spread constraints over hand-rolled `podAntiAffinity` — hard anti-affinity starves the scheduler at scale.

---

## 7. Pod and cluster security

**Enforce Pod Security `restricted` at the namespace level.**
PodSecurityPolicy was removed in 1.25; the replacement is the built-in
PodSecurity admission controller
([K8s — Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/);
[K8s — Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).
Two rollout rules:

- **Pin `enforce-version` to your cluster's minor.** `latest` re-evaluates against the current cluster, so an upgrade can suddenly reject pods that were valid yesterday. Pin and bump deliberately during upgrade rehearsals ([K8s — PSA labels](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-admission-labels-for-namespaces)).
- **Run `warn` + `audit` first.** `warn` surfaces warnings on `kubectl apply`; `audit` writes annotations to the audit log. Run both for a release cycle, fix offenders, then flip `enforce` ([K8s — PSA rollout](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)).

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.33   # pin to cluster minor
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.33
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.33
```

Every prod container runs with:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }
  seccompProfile: { type: RuntimeDefault }
```

**Don't** use `privileged: true`, `hostNetwork`, `hostPID`, or hostPath
mounts in app workloads. If your app needs them, it's a node agent — lives
in `kube-system` with a written threat model.

**Forward-looking: enable user namespaces (`spec.hostUsers: false`) once on v1.30+ (beta) / v1.33+ (GA on by default).** KEP-127 maps in-container UID 0 to an unprivileged host UID, which would have neutered several of the runc CVEs in the Leaky Vessels class (§12). Pair with PSA `restricted`; do not treat it as a replacement ([K8s v1.33 upcoming changes — user namespaces](https://kubernetes.io/blog/2025/03/26/kubernetes-v1-33-upcoming-changes/); [KEP-127](https://kep.k8s.io/127)).

For policy beyond PSA defaults you have two layers, in this order:

- **Built-in `ValidatingAdmissionPolicy` (CEL, GA in v1.30)** for the simple things — required labels, replicas ≤ N, registry allow-list, no `hostNetwork`/`hostPID`. In-process, no webhook hop, no extra controller to operate; supports `Deny` / `Warn` / `Audit` actions for safe rollout ([K8s — Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)).
- **Kyverno or Gatekeeper** for what CEL can't express cleanly: mutation, image-signature verification, cross-resource policies, generation. Pick one and bind it to the same policy bundle that runs in CI — this admission layer is the runtime half of the policy-as-code rule owned by **ch06** (which also owns SLSA, cosign verification, verifyImages, and Connaisseur policy).

For runtime detection: Falco / Tetragon (ch07). Liz Rice, *Container Security* is the canonical reference; required reading before anyone proposes gVisor or Kata.

---

## 8. Networking and NetworkPolicy

**Do** apply a `default-deny` NetworkPolicy in every namespace and
allow-list explicitly. Most in-cluster lateral movement happens because
pod-to-pod traffic was unrestricted by default
([K8s — Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/);
*Production Kubernetes*, ch. 11).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: app }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

**CNI: prefer your managed provider's default** (Azure CNI / AWS VPC CNI /
GKE Dataplane v2) unless you need a capability it doesn't ship — it's what
the provider tests, patches, and supports under SLA
([AKS networking](https://learn.microsoft.com/azure/aks/concepts-network);
[Amazon VPC CNI plugin](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html);
[GKE Dataplane v2](https://cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2)).

**Switch to Cilium** (or another eBPF CNI) when you have a concrete need:
identity-aware L7 NetworkPolicy, kube-proxy replacement at scale, cluster
mesh / multi-cluster service routing, transparent WireGuard / IPsec
encryption, or Hubble flow visibility you can't get from provider tooling
([Cilium docs](https://docs.cilium.io/en/stable/overview/intro/);
[CNCF — Cilium graduation](https://www.cncf.io/announcements/2023/10/11/cloud-native-computing-foundation-announces-cilium-graduation/)).
Calico remains a solid L3/L4 choice. Note: GKE Dataplane v2 *is* Cilium.

**If you already run Cilium (or Dataplane v2), evaluate Cilium's built-in service mesh + Gateway API implementation before adding a second mesh control plane (§9).** mTLS via WireGuard/IPsec, identity-aware L7 policy, and Gateway API conformance are already there; "Cilium CNI + Cilium mesh" and "Cilium CNI + Istio Ambient" are distinct options with different operating costs.

---

## 9. Service mesh — usually no

**Default: do not install a full L7 service mesh.** Per-pod sidecar
lifecycles, certificate rotation tied to pod restarts, control-plane
upgrades, and an extra hop to debug are real continuous costs
([Istio — concepts](https://istio.io/latest/docs/concepts/what-is-istio/);
[Linkerd — overview](https://linkerd.io/2/overview/)). mTLS as a
*transport security* stance is owned by **ch07**; this section is about
when a mesh, as a runtime component, earns its keep.

Concrete triggers — **any one** can justify a mesh:

- **Workload identity & mTLS by default** across many services with rotating SPIFFE-style identities you don't want to issue per-app.
- **L7 traffic shaping** (header/percentage routing, mirroring, retries, per-route timeouts) needed across multiple teams.
- **Per-service authorization** beyond NetworkPolicy (HTTP method/path-level, JWT claims).
- **Multi-cluster east-west** under a single logical service namespace.
- **In-cluster encryption compliance** your CNI doesn't already meet (many CNIs offer transparent WireGuard / IPsec — check first).

**Counter-rule for the mTLS trigger: prefer an L4-only ambient data plane before a sidecar mesh.** Istio Ambient went GA in **Istio 1.24 (2024-11-07)**: the per-node `ztunnel` provides mTLS and L4 authz for every workload in a labelled namespace, with no sidecar injection, no application restarts on mesh upgrade, and reported overhead reductions exceeding 90% vs sidecar mode ([Istio — Ambient mode reaches GA](https://istio.io/latest/blog/2024/ambient-reaches-ga/); [Istio — Ambient overview](https://istio.io/latest/docs/ambient/overview/)). Add waypoint proxies *only* for namespaces that actually need L7 features. This materially weakens the "no mesh by default" rule when the only trigger that fired is mTLS.

If you adopt a mesh, pick by workload fit, not benchmarks:

- **Istio Ambient** — L4 mTLS / identity for the whole cluster with per-namespace L7 waypoints; the right starting point in 2026 when the trigger is identity ([CNCF — Istio graduation](https://www.cncf.io/announcements/2023/07/12/cncf-istio/)).
- **Linkerd** — lower-complexity, opinionated, Rust dataplane, narrower feature surface, CNCF-graduated, sidecar model; pick when you want operational simplicity over feature breadth ([CNCF — Linkerd graduation](https://www.cncf.io/announcements/2021/07/28/cloud-native-computing-foundation-announces-linkerd-graduation/)).
- **Istio sidecar (classic)** — only when you need WASM filters, deep VirtualService routing, or features ambient hasn't matched yet.
- **Avoid** running a mesh "for observability" alone — OpenTelemetry, Cilium Hubble, and eBPF cover that without a sidecar on every pod.

Public mesh benchmarks are mostly vendor-published; treat as indicative.
Pilot against your own workloads before committing.

---

## 10. Ingress and Gateway API

**Treat Gateway API as the strategic direction for north-south traffic.**
Ingress v1 is feature-frozen; Gateway API is the actively-developed
replacement with a role-oriented split (Infra / Cluster / App) and a
richer expression model
([Gateway API project](https://gateway-api.sigs.k8s.io/);
[Migrating from Ingress](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/)).
Maturity tiers as of v1.2 (2024-11-21):

- **Standard channel, GA core:** `GatewayClass`, `Gateway`, `HTTPRoute`, `ReferenceGrant`, **GRPCRoute** (Standard since v1.1, May 2024). Target this for new traffic.
- **Standard channel, Extended GA features (v1.2):** **HTTPRoute Timeouts** (`request` / `backendRequest`), **BackendProtocol** AppProtocol values (`kubernetes.io/h2c`, `kubernetes.io/ws`, `kubernetes.io/wss`), and **Infrastructure label/annotation propagation** from `Gateway` to provisioned LB resources ([Gateway API v1.2 release notes](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.2.0)).
- **Experimental channel** (TCPRoute, UDPRoute, TLSRoute, BackendTLSPolicy, advanced policy attachment) — varying maturity, **not uniformly implemented**. Verify against your controller's conformance report ([Gateway API — conformance](https://gateway-api.sigs.k8s.io/concepts/conformance/); [v1.2 implementations](https://gateway-api.sigs.k8s.io/implementations/v1.2/)).

Rules:

- **Do** build new traffic on the GA core (ingress-nginx Gateway, Envoy Gateway, Istio Gateway, Cilium Gateway, GKE/AKS/EKS managed Gateway).
- **Don't** invest further in `Ingress` annotations — every controller's dialect is mutually incompatible. That's exactly what Gateway API exists to fix.
- **Don't** assume a `*Route` type from a blog post is portable; check the conformance matrix.

---

## 11. Autoscaling

- **HPA on real signals.** CPU is the lowest-quality default; prefer custom or external metrics (RPS, queue depth, p95 latency) ([K8s — HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)).
- **KEDA for event-driven scale and scale-to-zero.** Wraps queues, cron, Prometheus metrics as HPA external metrics ([KEDA](https://keda.sh/); [CNCF — KEDA graduation](https://www.cncf.io/announcements/2023/08/22/cloud-native-computing-foundation-announces-keda-graduation/)).
- **VPA in recommendation mode** for right-sizing; never auto-mode together with HPA on the same metric. On v1.33+ apply VPA recommendations via the in-place `resize` subresource (§5) when restart cost is high.
- **Cluster scale: prefer Karpenter v1.** Karpenter shipped **v1.0 in August 2024** with stable `NodePool` / `EC2NodeClass` APIs; v1beta1 is removed in v1.1 ([AWS — Announcing Karpenter 1.0](https://aws.amazon.com/blogs/containers/announcing-karpenter-1-0/); [Karpenter v1.0.0 release](https://github.com/kubernetes-sigs/karpenter/releases/tag/v1.0.0)). On AKS use **Node Auto-Provisioning (NAP)**, the productised AKS Karpenter provider ([AKS — Node Auto-Provisioning](https://learn.microsoft.com/en-us/azure/aks/node-autoprovision)). On GKE use node auto-provisioning.
- **Use Karpenter disruption budgets by reason.** Separate budgets for `Drifted`, `Underutilized`, and `Empty` are the practical cost/availability lever most teams miss — keep `Drifted` aggressive for security patching, throttle `Underutilized` to protect long-running workloads.

---

## 12. Runtime: containerd, not Docker

**Do** standardize on `containerd`. The dockershim was removed in K8s
1.24; `containerd` (and CRI-O) is what every managed K8s provider ships
([K8s — Don't Panic, 2020](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/);
[K8s — dockershim removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/)).
Docker remains a fine *developer* tool — `docker buildx`, `docker compose`
— but has no place on a production K8s node.

**Track the runc / containerd / BuildKit CVE feed as a Tier-0 stream.** The 2024 **Leaky Vessels** disclosures (CVE-2024-21626 in `runc`, plus CVE-2024-23651/23652/23653 in BuildKit) demonstrated container-to-host escapes that PSA `restricted`, non-root, and read-only-root-FS do **not** contain — they are kernel-namespacing / file-descriptor / build-time-mount flaws in the runtime itself ([Snyk — Leaky Vessels](https://snyk.io/blog/leaky-vessels-docker-runc-container-breakout-vulnerabilities/); [NVD — CVE-2024-21626](https://nvd.nist.gov/vuln/detail/CVE-2024-21626)). Concrete asks:

- **Document a node-image patch SLA.** Critical runtime CVE → managed-K8s node-image rollout within 14 days; surface the version of `runc` and `containerd` in node labels so it's visible to dashboards.
- **Treat CI runners as production.** Pin BuildKit / Buildx to a patched release and rebuild self-hosted runner images on the same SLA — Leaky Vessels hit BuildKit too.
- **User namespaces and gVisor / Kata** (§7, §15) are the structural mitigations for the next one.

---

## 13. Multi-tenancy and namespace strategy

If you operate K8s for more than one team, you owe them a tenancy model
on day one. Retrofitting isolation onto a flat cluster is one of the most
expensive K8s migrations there is
([K8s — Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/);
[CNCF Multi-Tenancy WG](https://github.com/kubernetes-sigs/multi-tenancy)).

Pick a **namespace pattern** and document it:

- **Namespace per app/environment** (`payments-prod`, `payments-stg`) — default for single-tenant clusters.
- **Namespace per team**, app-as-label inside — when teams own many small services and the blast-radius unit is the team.
- **Namespace per tenant** (soft multi-tenancy) — for internal customers you trust not to break out. Combine with NetworkPolicy + quota + PSA `restricted`.
- **Cluster per tenant** (hard multi-tenancy) — the only honest answer for untrusted code, regulated isolation, or sharply different upgrade cadences. Use vCluster or fleet management if cluster sprawl is the concern.

**RBAC depth.** Default to namespaced `Role` + `RoleBinding`; reserve
`ClusterRole` for cluster-wide reads (nodes, CRDs, metrics). Bind humans
through groups (OIDC / IdP claims), never by user. Audit
`ClusterRoleBinding`s quarterly — they accumulate
([K8s — RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)).

**Quota & limits per namespace.** Every tenant namespace ships a
`ResourceQuota` (CPU / memory / pod / PVC count) and a `LimitRange` so a
forgotten request doesn't land a BestEffort pod in production
([K8s — Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/);
[K8s — Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)).

**Admission stack, in order:** PodSecurity (built-in baseline) → built-in `ValidatingAdmissionPolicy` for simple CEL constraints → Kyverno or Gatekeeper for mutation, image-signature verification, and cross-resource policy → image-policy / cosign verification (owned by **ch06**) ([Kyverno docs](https://kyverno.io/docs/); [Gatekeeper docs](https://open-policy-agent.github.io/gatekeeper/website/docs/)).

**Pick separate clusters when *any* applies:**

- Tenants execute untrusted or third-party code.
- Regulatory boundary requires physical isolation (PCI scope, FedRAMP high, HIPAA-segmented).
- Tenants need different K8s minor versions or upgrade windows.
- A tenant's outage must not touch others, or its resource shape (GPU, large-memory) would dominate scheduling.

---

## 14. Managed vs self-hosted

**Do** use managed K8s (AKS / EKS / GKE) unless regulation or hardware
forces otherwise. The control-plane SLA, automated upgrades, integrated
IAM, CNI defaults, and node-image lifecycle are work you don't want to own
([AKS SLA](https://learn.microsoft.com/azure/aks/free-standard-pricing-tiers);
[EKS SLA](https://aws.amazon.com/eks/sla/);
[GKE SLA](https://cloud.google.com/kubernetes-engine/sla)).

**Self-hosted (kubeadm, Cluster API, Talos, k3s) is justified when:**

- Air-gapped or on-prem hardware.
- You are a hyperscaler / hosting provider.
- Edge footprint requires a stripped-down distro (k3s, Talos).
- Regulation forbids a hyperscaler-managed control plane.

### 14.1 If you must self-host

Self-hosting K8s is operating a distributed database (etcd) plus a
distributed control plane plus a node fleet. Budget for it like any other
Tier-0 system.

- **etcd backup & restore drills.** Snapshot on schedule (`etcdctl snapshot save`), store off-cluster, **practice restore**. An untested backup is not a backup ([K8s — Operating etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/); [etcd — disaster recovery](https://etcd.io/docs/v3.5/op-guide/recovery/)).
- **Control-plane sizing.** Three (or five) etcd members on dedicated low-latency disks; separate etcd from API server under load; size API server to peak watch fan-out, not steady-state QPS ([K8s — Large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/); [etcd — hardware recommendations](https://etcd.io/docs/v3.5/op-guide/hardware/)).
- **Version-skew policy.** Stay within the supported window: kubelet up to 3 minors behind the API server; control-plane components within 1 minor; clients within 1 ([K8s — Version skew policy](https://kubernetes.io/releases/version-skew-policy/)).
- **Deprecation tracking.** Run `kube-no-trouble` / Pluto in CI against manifests and Helm charts before every minor upgrade ([K8s — Deprecated API migration guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)).
- **Upgrade order.** etcd → control plane → node pools, one minor at a time, canary node pool first. Never skip minors.
- **Node OS.** Prefer a minimal, image-based, atomic-upgrade OS: **Talos Linux** (API-driven, no SSH), **Bottlerocket** (read-only root), or **Flatcar Container Linux** (auto-update channels). General-purpose distros work but cost you patching and drift management forever ([Talos Linux](https://www.talos.dev/v1.7/introduction/what-is-talos/); [Bottlerocket](https://bottlerocket.dev/); [Flatcar](https://www.flatcar.org/docs/latest/)).
- **RuntimeClass + sandboxed runtimes.** For untrusted workloads, run them under **gVisor** (`runsc`) or **Kata Containers** (lightweight VMs) via a `RuntimeClass`. Don't expect PSA `restricted` to carry that load alone ([K8s — RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/); [gVisor docs](https://gvisor.dev/docs/); [Kata Containers docs](https://github.com/kata-containers/kata-containers/tree/main/docs)).

---

## 15. Serverless containers — when they win outright

**Prefer** ACA / Cloud Run / App Runner / Fargate when:

- Single workload, HTTP or queue-driven, scale-to-zero matters, no platform team.
- Operational budget is "≤1 day/month on infra."
- Per-request billing beats a 24/7 node pool at your current traffic.

These are *production* runtimes built on K8s primitives (Cloud Run on
Knative; ACA on AKS+Dapr+KEDA; Fargate on Firecracker)
([Cloud Run docs](https://cloud.google.com/run/docs/overview/what-is-cloud-run);
[Azure Container Apps docs](https://learn.microsoft.com/azure/container-apps/overview);
[AWS Fargate on EKS](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)).
**They stop winning** when you need GPU pods, DaemonSets, tight
pod-to-pod networking, persistent local volumes, or >50 services on
shared infra — where managed K8s reasserts itself.

---

## 16. Operators and CRDs — high-leverage or high-rope

CRDs + controllers fit when the resource has a real *lifecycle* the
platform should manage continuously: certificates (cert-manager),
databases (CNPG, Zalando postgres-operator), message brokers (Strimzi).
Operators wrapping mature OSS data systems are some of the
highest-leverage infra in the ecosystem (Burns et al., *Kubernetes Up &
Running*, 3rd ed., O'Reilly 2022, ch. 17; *Kubernetes Patterns*, 2nd ed.,
ch. 28).

They become a foot-gun when the CRD models your *application's* domain
(Conway's Law in YAML), when one person wrote it and left, or when it
exists to avoid writing a Helm chart. **Rule:** consume mature operators;
write them only as a platform team selling the abstraction to >3 internal
customers and willing to own it for years.

---

## 17. Batch, AI, and accelerator workloads

For any cluster that runs Jobs, ML training, batch inference, or HPC at scale, the default `Job` + HPA + Cluster Autoscaler stack is not enough — there is no queueing, no fair share between teams, no all-or-nothing (gang) admission, and GPU scheduling falls back to the device-plugin model.

- **Install Kueue for job queueing, fair sharing, and gang admission** ([Kueue — kubernetes-sigs/kueue](https://github.com/kubernetes-sigs/kueue)). Kueue is a sig-scheduling project with native integrations for `BatchJob`, JobSet, Kubeflow training jobs, RayJob/RayCluster, and plain Pod groups; supports cohorts, preemption, MultiKueue (multi-cluster dispatching), Topology-Aware Scheduling, and integration with Cluster Autoscaler `provisioningRequest`.
- **Use JobSet** for multi-pod parallel groups (e.g., distributed training) where the unit of scheduling is the whole set, not individual pods.
- **Track Dynamic Resource Allocation (DRA).** The structured-parameter core graduated to beta in v1.32 ([K8s v1.32 release](https://kubernetes.io/blog/2024/12/11/kubernetes-v1-32-release/)); it is the long-term replacement for the device-plugin-only model for GPUs / FPGAs / NICs and will eventually carry workload-aware GPU sharing.

This is the single biggest gap in most "we put ML on Kubernetes" platforms — paved-road treatment for these workloads is owned by **ch11**.

---

## 18. Checklist (PR / repo gate)

- [ ] Image: distroless / Wolfi / Chainguard, multi-stage, non-root, pinned by digest
- [ ] BuildKit SBOM + provenance generated; cosign-signed (verification owned by ch06)
- [ ] Large auxiliary payloads (models, datasets, plugins) shipped as OCI artifacts via `image` volume, not baked into the runtime image
- [ ] CPU & memory **requests** set; memory **limit** set; CPU limit only with justification (Burstable by default)
- [ ] `Guaranteed` QoS only where CPU caps / pinned cores are intentional; cgroup v2 nodes assumed
- [ ] VPA in recommendation mode; resize via `resize` subresource (v1.33+) for stateful/long-running pods instead of rolling restart
- [ ] `startupProbe` (cold start >5s); dependency-free `readinessProbe`; `livenessProbe` only if defensible
- [ ] Sidecars (mesh / log shipper / proxies) declared as native sidecars (`initContainers` + `restartPolicy: Always`) on v1.29+ clusters
- [ ] App handles SIGTERM: fail readiness → drain → exit; `terminationGracePeriodSeconds` ≥ drain time; `preStop` only for specific drain coordination
- [ ] PDB + `topologySpreadConstraints` for every multi-replica Deployment
- [ ] Namespace labelled `pod-security.kubernetes.io/enforce: restricted` with `enforce-version` pinned to cluster minor; `warn` + `audit` enabled
- [ ] `securityContext`: non-root, read-only FS, drop ALL caps, seccomp RuntimeDefault; `hostUsers: false` planned for v1.33+
- [ ] Admission stack: PSA → `ValidatingAdmissionPolicy` (CEL) for simple constraints → Kyverno/Gatekeeper for mutation & cross-resource → cosign verification (ch06)
- [ ] `default-deny` NetworkPolicy + explicit allow-list
- [ ] North-south on Gateway API GA core (`Gateway` / `HTTPRoute` / `GRPCRoute`, HTTPRoute Timeouts, BackendProtocol); experimental routes verified against conformance matrix
- [ ] No service mesh unless a concrete trigger fires; if the trigger is mTLS-only, prefer Istio Ambient (ztunnel / L4) before sidecar-mode Istio or Linkerd
- [ ] HPA on a real signal; KEDA for event-driven; Karpenter v1 / NAP / managed autoscaler for nodes with disruption budgets by reason
- [ ] Multi-tenant clusters: `ResourceQuota` + `LimitRange` + namespaced RBAC + admission policy stack above
- [ ] Batch / AI clusters: Kueue installed for queueing & fair share; JobSet for multi-pod groups; DRA tracked
- [ ] Node-runtime CVE SLA documented (Leaky Vessels-class): runc / containerd / BuildKit patched within 14 days; CI runner images on the same SLA
- [ ] Cluster: managed K8s (AKS/EKS/GKE) or documented self-hosting plan (etcd backup drill, version-skew policy, node OS choice, RuntimeClass for untrusted workloads)
