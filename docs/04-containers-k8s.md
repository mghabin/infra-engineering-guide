# Chapter 04 — Containers & Kubernetes

Opinionated, cloud-agnostic defaults for running container workloads in 2026.
Most of this chapter is about *not* over-engineering: Kubernetes is the right
answer less often than the industry believes, and even when it is, the smallest
viable footprint wins.

> Conventions: **Do** = required default. **Don't** = reject in review unless a
> written exception exists. **Prefer** = strong default; deviate only with
> measurement or a documented constraint.

---

## 1. Do you need Kubernetes at all?

**Default to "no" until you can name three workloads that need it.** Kelsey
Hightower has been saying for years that "Kubernetes is a platform for building
platforms; it is a better place to start, not the endgame" — most teams running
fewer than ~30 services on one team should not operate it themselves
([Hightower, 2017](https://twitter.com/kelseyhightower/status/935252923721793536);
[ThoughtWorks Tech Radar — Kubernetes](https://www.thoughtworks.com/radar)).

A small team's decision tree:

1. One stateless HTTP service + managed DB → **Cloud Run / Azure Container Apps / AWS App Runner**. Done.
2. ≤10 services, event-driven, scale-to-zero matters → **ACA / Cloud Run / Fargate, or Knative on a managed cluster**.
3. Heavy stateful workloads, multi-tenant, custom networking, GPU pools, or you're a platform team → **managed Kubernetes (AKS/EKS/GKE)**.
4. Air-gapped / regulated / per-pod hardware control → self-hosted K8s. The only honest reason to run kubeadm in 2026.

> **K8s-fit smell test:** if your `kubectl get pods -A` fits on one screen
> *and* you have no platform engineer, you are paying the K8s tax for nothing.

---

## 2. Container image hygiene

**Use a minimal, distroless or Wolfi-based runtime image.** No shell, no
package manager, no `curl` in production. Distroless and Chainguard/Wolfi
images ship only what the app needs, eliminating most CVE noise and the
"attacker shells in" class of incident
([Google distroless](https://github.com/GoogleContainerTools/distroless);
[Chainguard Images](https://www.chainguard.dev/chainguard-images);
Liz Rice, *Container Security*, O'Reilly 2020, ch. 6).

```dockerfile
# Multi-stage; build with a full SDK, run on distroless.
FROM golang:1.23 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

**Do** use multi-stage builds; the runtime stage never sees build tools.
**Do** run as a non-root UID (`USER 65532` or `nonroot`); Pod Security
`restricted` requires `runAsNonRoot: true`
([K8s Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).
**Don't** ship `:latest`; pin by digest (`@sha256:...`). **Don't** install a
shell "for debugging" — use ephemeral debug containers (`kubectl debug`)
([K8s ephemeral containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)).

**Build with BuildKit and produce SBOMs at build time.** `docker buildx build
--sbom=true --provenance=mode=max --platform=linux/amd64,linux/arm64` gives
you reproducible multi-arch images plus SLSA provenance and an SPDX SBOM in
one shot ([BuildKit attestations](https://docs.docker.com/build/attestations/)).
This ties directly into ch03's signing/provenance story: cosign-sign the
*digest*, verify on admission with policy-controller / Kyverno / Connaisseur.

---

## 3. Registries

**Do** scan on push and on a daily schedule, *not only at build time* — new CVEs
land against old images
([Trivy](https://aquasecurity.github.io/trivy/); [Grype](https://github.com/anchore/grype)).
**Do** run a pull-through cache (Harbor, ACR/ECR/Artifact Registry remote
repos) in every cluster's region: it cuts cold-start latency, defeats Docker
Hub rate-limits, and gives you a single chokepoint for vulnerability gates.
**Do** set a retention policy: keep last N tags + anything signed in the last
90 days; everything else is garbage-collected. Untagged layers are the #1
registry cost driver.
**Don't** allow direct `docker pull docker.io/...` from cluster nodes — proxy
it. (See ch07 for image policy enforcement and ch03 for cosign.)

---

## 4. The 12-factor app, revisited

[12factor.net](https://12factor.net) still holds; the container-era reading:

- **Config in env vars or projected files**, never baked into the image. Same
  image promotes from dev → prod; only env differs.
- **Stateless processes.** Anything that survives a pod restart lives in a DB,
  object store, or cache — not on local disk.
- **Logs to stdout/stderr.** The platform ships them. No log files, no
  in-process log shippers (see ch06).
- **Disposability.** Boot in <10s, shut down on SIGTERM cleanly. K8s gives you
  `terminationGracePeriodSeconds` (default 30s) — *use it*, don't fight it.
- **Dev/prod parity.** Same base image, same runtime, same config keys. If
  prod uses Postgres, dev does not use SQLite.

---

## 5. Resource requests, limits, and QoS

This is the section most teams get wrong.

**Do** set CPU and memory **requests > 0** on every container. Without
requests, the scheduler treats the pod as zero-cost, packs it onto any node,
and you discover the truth under load
([K8s — Resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/);
*Production Kubernetes*, Dotson et al., O'Reilly 2021, ch. 7).

**Do** set **memory limits**. The kernel OOM-kills cleanly; an unbounded
process takes the node with it.

**Prefer no CPU limits** for latency-sensitive services. Tim Hockin (K8s
co-founder/maintainer) and the CFS-throttling literature show that CPU limits
cause throttling on bursty workloads even when the node has idle CPU, with
p99 latency spikes that look like network problems
([Hockin, 2023 — "Stop using CPU limits"](https://twitter.com/thockin/status/1134193838841401345);
[Henning Jacobs, "CPU limits and aggressive throttling"](https://erickhun.com/posts/kubernetes-faster-services-no-cpu-limits/)).
Set CPU **requests** correctly (that's what drives scheduling and proportional
share) and skip the limit unless you have a specific noisy-neighbor case.
For batch / untrusted multi-tenant workloads, CPU limits remain appropriate.

**QoS classes** flow from this:

- `Guaranteed` (requests == limits on every container): last to be evicted.
  Use for stateful / latency-critical pods.
- `Burstable` (requests set, limits higher or unset): the right default for
  most services.
- `BestEffort` (nothing set): never in production.

Right-size from real data, not guesses: VPA in **recommendation-only mode**
(`updateMode: Off`) is the standard tool; turning on auto-update with HPA on
the same metric breaks both
([VPA design notes](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler);
*Kubernetes Patterns*, Ibryam & Huss, 2nd ed., O'Reilly 2023, ch. 2).

---

## 6. Probes and lifecycle

**Three probes, three jobs** — and the most common bug is using one for all
three ([K8s — Configure probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)):

- **`startupProbe`** — "is the app done booting?" Generous
  `failureThreshold`. Disables the other probes until it passes. Use whenever
  cold start > a few seconds (JVM, .NET, large model load).
- **`readinessProbe`** — "should I be in the Service endpoints right now?"
  Cheap, frequent, *no external dependencies*. A readiness probe that checks
  the database will cascade-fail your whole fleet when the DB blips.
- **`livenessProbe`** — "am I wedged and need a restart?" Used sparingly. If
  in doubt, omit it; a bad liveness probe is worse than none
  (*Production Kubernetes*, ch. 9).

**SIGTERM handling is mandatory.** On pod deletion K8s removes the pod from
endpoints *and* sends SIGTERM concurrently — there is a race where in-flight
requests arrive at a terminating pod. The fix is a `preStop` sleep (5–10s)
before SIGTERM, and an app that drains in-flight work on SIGTERM
([K8s — Pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination);
*Kubernetes Patterns*, ch. 5 "Managed Lifecycle").

```yaml
lifecycle:
  preStop:
    exec: { command: ["sleep", "10"] }
terminationGracePeriodSeconds: 60
```

**PDBs and spread.** Every Deployment with replicas > 1 ships a
`PodDisruptionBudget` (`minAvailable: N-1` or `maxUnavailable: 1`) and a
`topologySpreadConstraints` across zones. Without a PDB, a node drain or
cluster upgrade can take the whole service down in one move
([K8s — PDB](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/);
[topologySpreadConstraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)).
Prefer `topologySpreadConstraints` over hand-rolled `podAntiAffinity` —
anti-affinity hard rules will starve the scheduler at scale.

---

## 7. Pod and cluster security

**Enforce Pod Security `restricted` at the namespace level.** PodSecurityPolicy
is gone (removed in 1.25); the replacement is the built-in PodSecurity
admission controller
([K8s — Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/);
[Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).
Workload namespaces:

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
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

**Don't** use `privileged: true`, `hostNetwork`, `hostPID`, or hostPath mounts
in app workloads — these are infra-only escape hatches. If your app needs
them, it isn't an app, it's a node agent, and it goes in `kube-system` with a
written threat model.

For policy enforcement beyond PSA defaults, use Kyverno or Gatekeeper. For
runtime detection, Falco / Tetragon (see ch07).
Liz Rice, *Container Security* (O'Reilly, 2020) is the canonical reference;
the runtime-isolation chapter is required reading before anyone proposes
gVisor or Kata.

---

## 8. Networking and NetworkPolicy

**Do** apply a `default-deny` NetworkPolicy in every namespace and
allow-list explicitly. Most production breaches that escalate inside a
cluster do so because pod-to-pod traffic was unrestricted by default
([K8s — NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/);
*Production Kubernetes*, ch. 11).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: app }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

**CNI: prefer Cilium** for new clusters. eBPF-based dataplane, kube-proxy
replacement, L7-aware policies, Hubble for flow visibility, and the de-facto
direction of travel for managed providers
([Cilium docs](https://docs.cilium.io/); [ThoughtWorks Tech Radar — eBPF, Adopt](https://www.thoughtworks.com/radar/techniques/ebpf)).
Calico is a fine alternative if you only need L3/L4 policy and prefer iptables
familiarity. The cloud-default CNIs (Azure CNI, AWS VPC CNI, GKE Dataplane v2
which *is* Cilium) are acceptable for managed clusters; only swap if you have
a specific reason.

---

## 9. Service mesh — usually no

**Default: do not install a service mesh.** Most teams that adopt one ship a
pile of complexity to solve problems they don't have. A mesh earns its keep
when you need *all three*: (a) mTLS between services with rotating certs you
won't manage yourself, (b) L7 traffic-shaping for canary/AB at scale, and
(c) per-service authz policy beyond NetworkPolicy
([William Morgan — "Do you need a service mesh?"](https://buoyant.io/service-mesh-manifesto);
[Layer5 Service Mesh Landscape](https://layer5.io/service-mesh-landscape)).

If the answer is genuinely yes:

- **Prefer Linkerd** for most cases. Smaller, faster, Rust dataplane, lower cognitive load, CNCF-graduated. Public benchmarks show materially lower p99 overhead than Istio for typical workloads ([Linkerd 2.14 benchmarks, Buoyant](https://buoyant.io/linkerd-vs-istio-benchmarks-2024); [CNCF Linkerd graduation](https://www.cncf.io/announcements/2021/07/28/cloud-native-computing-foundation-announces-linkerd-graduation/)).
- **Choose Istio** only when you specifically need the breadth of its API (multi-cluster mesh, advanced WASM filters, deep VirtualService routing). Ambient mode is a real improvement on the sidecar tax but it is still Istio.
- **Avoid** running a mesh "for observability" — OpenTelemetry, Cilium Hubble, and eBPF tooling cover that without a sidecar on every pod.

---

## 10. Ingress and Gateway API

**Migrate north-south traffic to Gateway API.** The Ingress v1 API is in
maintenance mode; Gateway API is GA, role-oriented (Infra/Cluster/App split),
expression-richer, and supported by every major implementation
([Gateway API project](https://gateway-api.sigs.k8s.io/);
[Migrating from Ingress](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/);
[ThoughtWorks Tech Radar — Gateway API, Trial](https://www.thoughtworks.com/radar)).

**Do** new traffic on Gateway API (ingress-nginx Gateway, Envoy Gateway, Istio
Gateway, Cilium Gateway, GKE/AKS/EKS managed Gateway).
**Don't** invest further in `Ingress` annotations — every controller's
annotation dialect is incompatible with every other's. That problem is exactly
what Gateway API exists to fix.

---

## 11. Autoscaling

- **HPA on real signals.** CPU is the lowest-quality default; prefer custom
  or external metrics (RPS, queue depth, p95 latency)
  ([K8s — HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)).
- **KEDA for event-driven scale and scale-to-zero.** Queues (Kafka, SQS,
  Service Bus, Pub/Sub), cron, Prometheus metrics — KEDA wraps them as HPA
  external metrics with a clean operator model
  ([KEDA](https://keda.sh/); CNCF graduated 2023).
- **VPA in recommendation mode** for right-sizing; never auto-mode together
  with HPA on the same metric.
- **Cluster scale: prefer Karpenter** on AWS (and increasingly portable —
  donated to CNCF, with Azure and GCP providers in active development).
  Faster, cheaper, instance-type-aware vs the legacy Cluster Autoscaler
  ([Karpenter](https://karpenter.sh/); CNCF project page).
  On AKS use the AKS Karpenter provider or the cluster autoscaler; on GKE,
  use node auto-provisioning.

---

## 12. Runtime: containerd, not Docker

**Do** standardize on `containerd` as the node runtime. The Docker-shim was
removed in K8s 1.24; `containerd` (and CRI-O) is what every managed K8s
provider ships
([K8s — Don't Panic: Kubernetes and Docker, 2020](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/);
[K8s — dockershim removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/)).
Docker (the daemon) remains a fine *developer* tool — `docker buildx` for
building images, `docker compose` for local — but it has no place on a
production K8s node.

---

## 13. Managed vs self-hosted

**Do** use managed K8s (AKS / EKS / GKE) unless you have a regulatory or
hardware reason not to. The control-plane SLA, automated upgrades, integrated
IAM, CNI defaults, and node-image lifecycle are work you do not want to own.
Cost difference is rounding error vs one SRE-week chasing an etcd corruption.

**Self-hosted (kubeadm, Cluster API, Talos, k3s) is justified when:**

- Air-gapped or on-prem hardware.
- You are a hyperscaler / hosting provider.
- You need a stripped-down distro for edge (k3s, Talos).

Otherwise: managed. Kelsey Hightower's "managed Kubernetes is the only
Kubernetes most companies should run" stance has aged into received wisdom
([Hightower, KubeCon keynotes 2018–2022](https://www.youtube.com/results?search_query=kelsey+hightower+kubecon+keynote)).

---

## 14. Serverless containers — when they win outright

**Prefer** ACA / Cloud Run / App Runner / Fargate when:

- Single workload, HTTP or queue-driven, scale-to-zero matters, no platform team.
- Team operational budget is "≤1 day/month on infra."
- Per-request billing beats a 24/7 node pool at your current traffic.

These are *production* runtimes built on K8s primitives (Cloud Run on
Knative; ACA on AKS+Dapr+KEDA; Fargate on Firecracker). They hide the
cluster, which is the whole point.

**They stop winning when** you need GPU pods, DaemonSets, tight pod-to-pod
networking, persistent local volumes, or >50 services on shared infra. That is
where managed K8s reasserts itself.

---

## 15. Operators and CRDs — high-leverage or high-rope

CRDs + controllers fit when the resource has a real *lifecycle* the platform
should manage continuously: certificates (cert-manager), databases (CNPG,
Zalando postgres-operator), message brokers (Strimzi). Brendan Burns' framing
— "operators encode operational knowledge as code" — is correct, and operators
wrapping mature open-source data systems are some of the highest-leverage
infrastructure in the ecosystem (Burns et al., *Kubernetes Up & Running*, 3rd
ed., O'Reilly 2022, ch. 17; *Kubernetes Patterns*, 2nd ed., ch. 28).

They become a foot-gun when the CRD models your *application's* domain (a
bespoke control plane only your team understands — Conway's Law in YAML),
when one person wrote it and left, or when it exists to avoid writing a Helm
chart.

**Rule:** consume mature operators; write them only as a platform team selling
the abstraction to >3 internal customers and willing to own it for years.

---

## 16. Checklist (PR / repo gate)

- [ ] Image: distroless / Wolfi / Chainguard, multi-stage, non-root, pinned by digest
- [ ] BuildKit SBOM + provenance generated; cosign-signed (ch03)
- [ ] CPU & memory **requests** set; memory **limit** set; CPU limit only with justification
- [ ] `startupProbe` (cold start >5s); dependency-free `readinessProbe`; `livenessProbe` only if defensible
- [ ] `preStop` sleep + SIGTERM-aware app; `terminationGracePeriodSeconds` ≥ drain time
- [ ] PDB + `topologySpreadConstraints` for every multi-replica Deployment
- [ ] Namespace labelled `pod-security.kubernetes.io/enforce: restricted`
- [ ] `securityContext`: non-root, read-only FS, drop ALL caps, seccomp RuntimeDefault
- [ ] `default-deny` NetworkPolicy + explicit allow-list
- [ ] North-south on Gateway API (or scheduled migration off Ingress)
- [ ] No service mesh unless mTLS + L7 shaping + per-service authz are *all* required
- [ ] HPA on a real signal; KEDA for event-driven; Karpenter / managed autoscaler for nodes
- [ ] Cluster: managed K8s (AKS/EKS/GKE) or documented reason for self-hosting
