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

**Build with BuildKit and produce SBOMs at build time.** `docker buildx build --sbom=true --provenance=mode=max --platform=linux/amd64,linux/arm64` produces multi-arch images plus SLSA provenance and an SPDX SBOM ([BuildKit attestations](https://docs.docker.com/build/attestations/)). Cosign-sign the *digest*; verify on admission (ch03).

---

## 3. Registries

- **Do** scan on push *and* on a daily schedule — new CVEs land against old images ([Trivy](https://trivy.dev/); [Grype](https://github.com/anchore/grype)).
- **Do** run a pull-through cache (Harbor / ACR / ECR / Artifact Registry remote repos) per region: cuts cold-start latency, defeats Docker Hub rate-limits, gives you one chokepoint for vulnerability gates.
- **Do** set retention: keep last N tags + anything signed in the last 90 days; everything else is GC'd. Untagged layers are the #1 registry cost driver.
- **Don't** allow direct `docker pull docker.io/...` from cluster nodes — proxy it. (See ch07 for image policy and ch03 for cosign.)

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

This is the section most teams get wrong.

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

Right-size from data: VPA in **recommendation-only mode**
(`updateMode: Off`); auto-mode together with HPA on the same metric breaks
both ([VPA repo](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler);
*Kubernetes Patterns*, Ibryam & Huss, 2nd ed., O'Reilly 2023, ch. 2).

---

## 6. Probes and lifecycle

**Three probes, three jobs** ([K8s — Configure probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)):

- **`startupProbe`** — "is the app done booting?" Generous `failureThreshold`. Disables the others until it passes. Use whenever cold start > a few seconds.
- **`readinessProbe`** — "should I be in Service endpoints right now?" Cheap, frequent, *no external dependencies*. A readiness probe that checks the DB will cascade-fail your fleet when the DB blips.
- **`livenessProbe`** — "am I wedged?" Used sparingly. If in doubt, omit it; a bad liveness probe is worse than none (*Production Kubernetes*, ch. 9).

**Graceful shutdown is an app responsibility, not a `sleep` workaround.**
On pod deletion the kubelet sends SIGTERM at the same time as endpoint
controllers begin removing the pod from EndpointSlices. Endpoint
propagation to every kube-proxy / LB / ingress controller is not
instantaneous, so for a brief window new requests can still land on a
terminating pod
([K8s — Pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination);
[K8s — EndpointSlice conditions](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/#conditions)).
Robust sequence:

- **Fail readiness first.** On SIGTERM, flip `/readyz` to fail so endpoint controllers stop sending new traffic; the kubelet keeps the pod `terminating` while in-flight work finishes.
- **Drain in-flight work in the app.** Stop new connections, finish outstanding requests, close pools, flush queues, exit 0. Set `terminationGracePeriodSeconds` ≥ real drain time + buffer.
- **Coordinate with external load balancers** that hit pod IPs directly (AWS NLB/ALB, Azure LB, GKE container-native LB). Use the controller's drain hook (e.g., AWS LB controller's `deregistration_delay.timeout_seconds`) instead of a blind sleep.
- **Use `preStop` only for specific drain coordination** the app can't do — Envoy's `/quitquitquit`, signalling a sidecar, handing off a stateful peer. A blanket `preStop: ["sleep", "10"]` is a smell: it delays SIGTERM without proving the pod drained, and hides shutdown bugs.

```yaml
# preStop only drains the Envoy sidecar; the app handles its own SIGTERM.
lifecycle:
  preStop:
    exec: { command: ["/bin/sh", "-c",
      "curl -fsS -X POST http://127.0.0.1:15000/drain_listeners?inboundonly && sleep 5"] }
terminationGracePeriodSeconds: 60
```

**PDBs and spread.** Every Deployment with replicas > 1 ships a
`PodDisruptionBudget` (`maxUnavailable: 1`) and `topologySpreadConstraints`
across zones. Without a PDB, a node drain can take the whole service down
([K8s — PDB](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/);
[K8s — topologySpreadConstraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)).
Prefer spread constraints over hand-rolled `podAntiAffinity` — hard
anti-affinity starves the scheduler at scale.

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
    pod-security.kubernetes.io/enforce-version: v1.31   # pin to cluster minor
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.31
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.31
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

For policy beyond PSA defaults: Kyverno or Gatekeeper. For runtime
detection: Falco / Tetragon (ch07). Liz Rice, *Container Security* is the
canonical reference; required reading before anyone proposes gVisor or Kata.

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
mesh / multi-cluster service routing, or Hubble flow visibility you can't
get from provider tooling
([Cilium docs](https://docs.cilium.io/en/stable/overview/intro/);
[CNCF — Cilium graduation](https://www.cncf.io/announcements/2023/10/11/cloud-native-computing-foundation-announces-cilium-graduation/)).
Calico remains a solid L3/L4 choice. Note: GKE Dataplane v2 *is* Cilium.

---

## 9. Service mesh — usually no

**Default: do not install a service mesh.** A mesh runs alongside every
workload; sidecar lifecycles, certificate rotation, control-plane
upgrades, and an extra hop to debug are real continuous costs
([Istio — concepts](https://istio.io/latest/docs/concepts/what-is-istio/);
[Linkerd — overview](https://linkerd.io/2/overview/)).

Concrete triggers — **any one** can justify a mesh:

- **Workload identity & mTLS by default** across many services with rotating SPIFFE-style identities you don't want to issue per-app.
- **L7 traffic shaping** (header/percentage routing, mirroring, retries, per-route timeouts) needed across multiple teams.
- **Per-service authorization** beyond NetworkPolicy (HTTP method/path-level, JWT claims).
- **Multi-cluster east-west** under a single logical service namespace.
- **In-cluster encryption compliance** your CNI doesn't already meet (many CNIs offer transparent WireGuard / IPsec — check first).

If you adopt one, pick by workload fit, not benchmarks:

- **Linkerd** — lower-complexity, opinionated, Rust dataplane, narrower feature surface, CNCF-graduated ([CNCF — Linkerd graduation](https://www.cncf.io/announcements/2021/07/28/cloud-native-computing-foundation-announces-linkerd-graduation/); [Linkerd docs](https://linkerd.io/2/overview/)).
- **Istio** — when you need its breadth: ambient mode (no per-pod sidecar), WASM filters, deep VirtualService routing, multi-cluster ([Istio — ambient](https://istio.io/latest/docs/ambient/overview/); [CNCF — Istio graduation](https://www.cncf.io/announcements/2023/07/12/cncf-istio/)).
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
Portability today is uneven:

- **Stable / GA core:** `GatewayClass`, `Gateway`, `HTTPRoute`. GA in Gateway API v1.0+ and supported by every major implementation ([Gateway API v1.0 release](https://gateway-api.sigs.k8s.io/blog/2023/v1.0-release/)). Target this for new traffic.
- **Experimental / standard-channel features** (TCPRoute, UDPRoute, GRPCRoute, TLSRoute, BackendTLSPolicy, advanced policy attachment) are at varying maturity and **not uniformly implemented**. Verify against your controller's conformance report before depending on them ([Gateway API — conformance](https://gateway-api.sigs.k8s.io/concepts/conformance/); [implementations matrix](https://gateway-api.sigs.k8s.io/implementations/)).

- **Do** new traffic on the GA core (ingress-nginx Gateway, Envoy Gateway, Istio Gateway, Cilium Gateway, GKE/AKS/EKS managed Gateway).
- **Don't** invest further in `Ingress` annotations — every controller's dialect is mutually incompatible. That's exactly what Gateway API exists to fix.
- **Don't** assume a `*Route` type from a blog post is portable; check the conformance matrix.

---

## 11. Autoscaling

- **HPA on real signals.** CPU is the lowest-quality default; prefer custom or external metrics (RPS, queue depth, p95 latency) ([K8s — HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)).
- **KEDA for event-driven scale and scale-to-zero.** Wraps queues, cron, Prometheus metrics as HPA external metrics ([KEDA](https://keda.sh/); [CNCF — KEDA graduation](https://www.cncf.io/announcements/2023/08/22/cloud-native-computing-foundation-announces-keda-graduation/)).
- **VPA in recommendation mode** for right-sizing; never auto-mode together with HPA on the same metric.
- **Cluster scale: prefer Karpenter** on AWS (donated to CNCF, Azure/GCP providers in active development) — instance-type-aware vs the legacy Cluster Autoscaler ([Karpenter docs](https://karpenter.sh/); [CNCF — Karpenter](https://www.cncf.io/projects/karpenter/)). On AKS use the AKS Karpenter provider; on GKE, node auto-provisioning.

---

## 12. Runtime: containerd, not Docker

**Do** standardize on `containerd`. The dockershim was removed in K8s
1.24; `containerd` (and CRI-O) is what every managed K8s provider ships
([K8s — Don't Panic, 2020](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/);
[K8s — dockershim removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/)).
Docker remains a fine *developer* tool — `docker buildx`, `docker compose`
— but has no place on a production K8s node.

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

**Admission stack, in order:** PodSecurity (built-in baseline) → Kyverno
or Gatekeeper for org policy (required labels, signed-image enforcement,
registry allow-lists, no `:latest`) → image-policy verification
(cosign / policy-controller / Connaisseur, ch03)
([Kyverno docs](https://kyverno.io/docs/);
[Gatekeeper docs](https://open-policy-agent.github.io/gatekeeper/website/docs/)).

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

## 17. Checklist (PR / repo gate)

- [ ] Image: distroless / Wolfi / Chainguard, multi-stage, non-root, pinned by digest
- [ ] BuildKit SBOM + provenance generated; cosign-signed (ch03)
- [ ] CPU & memory **requests** set; memory **limit** set; CPU limit only with justification (Burstable by default)
- [ ] `Guaranteed` QoS only where CPU caps / pinned cores are intentional
- [ ] `startupProbe` (cold start >5s); dependency-free `readinessProbe`; `livenessProbe` only if defensible
- [ ] App handles SIGTERM: fail readiness → drain → exit; `terminationGracePeriodSeconds` ≥ drain time; `preStop` only for specific drain coordination
- [ ] PDB + `topologySpreadConstraints` for every multi-replica Deployment
- [ ] Namespace labelled `pod-security.kubernetes.io/enforce: restricted` with `enforce-version` pinned to cluster minor; `warn` + `audit` enabled
- [ ] `securityContext`: non-root, read-only FS, drop ALL caps, seccomp RuntimeDefault
- [ ] `default-deny` NetworkPolicy + explicit allow-list
- [ ] North-south on Gateway API GA core (`Gateway` / `HTTPRoute`); experimental routes verified against conformance matrix
- [ ] No service mesh unless a concrete trigger (mTLS / L7 shaping / per-service authz / multi-cluster / encryption compliance) is documented
- [ ] HPA on a real signal; KEDA for event-driven; Karpenter / managed autoscaler for nodes
- [ ] Multi-tenant clusters: `ResourceQuota` + `LimitRange` + namespaced RBAC + admission policy (Kyverno/Gatekeeper)
- [ ] Cluster: managed K8s (AKS/EKS/GKE) or documented self-hosting plan (etcd backup drill, version-skew policy, node OS choice, RuntimeClass for untrusted workloads)
