# Kubernetes Fundamentals & Java-Specific Operational Patterns — Principal Engineer Reference

> File 2 of 7 in the Cloud-Native & DevOps series. Covers Kubernetes through the specific lens of running JVM workloads well — resource sizing, probe configuration, and autoscaling all have JVM-specific gotchas that generic Kubernetes documentation doesn't surface.

**Series:** 1. Docker & Containerization · **2. Kubernetes for Java (this file)** · 3. CI/CD & GitOps · 4. Cloud Provider Deep Dive · 5. Infrastructure as Code · 6. Observability & SRE · 7. Cost Optimization & FinOps

---

## Table of Contents

1. [Core Concepts, Java-Lens](#1-core-concepts-java-lens)
2. [Resource Requests & Limits for JVM Workloads](#2-resource-requests--limits-for-jvm-workloads)
3. [Probes — Liveness, Readiness & Startup](#3-probes--liveness-readiness--startup)
4. [Horizontal Pod Autoscaling & the JVM Warm-Up Tension](#4-horizontal-pod-autoscaling--the-jvm-warm-up-tension)
5. [ConfigMaps, Secrets & Spring Configuration](#5-configmaps-secrets--spring-configuration)
6. [Deployment Strategies](#6-deployment-strategies)
7. [Graceful Termination & Pod Disruption Budgets](#7-graceful-termination--pod-disruption-budgets)
8. [StatefulSets — When Java Workloads Need Them](#8-statefulsets--when-java-workloads-need-them)
9. [Namespaces & Multi-Tenancy](#9-namespaces--multi-tenancy)
10. [Worked Manifest — A Production-Grade Spring Boot Deployment](#10-worked-manifest--a-production-grade-spring-boot-deployment)

---

## 1. Core Concepts, Java-Lens

### 1.1 The Object Hierarchy

```
Deployment (declares DESIRED state: which image, how many replicas, update strategy)
   └─ manages → ReplicaSet (ensures N pod replicas exist)
         └─ manages → Pod (one or more containers, sharing network/storage)
               └─ runs → Container (the Docker image from File 1 of this series)

Service (a STABLE network identity + load balancer in front of a set of pods,
          selected by LABEL — directly the Java System Design guide's §8.3 service-
          discovery pattern, here provided NATIVELY by the platform, exactly as that
          section anticipated: "in Kubernetes specifically, this is almost entirely
          abstracted away by the platform's own built-in Service/DNS mechanism")
```

### 1.2 Why Pods, Not Containers, Are the Atomic Unit

```
A Pod can hold MULTIPLE containers sharing network namespace and storage — the dominant
use case being a SIDECAR pattern: a main application container plus a SEPARATE, auxiliary
container handling a cross-cutting concern (a service-mesh proxy, directly the Spring Cloud
guide's §8.4 service-mesh sidecar discussion; a log-shipping agent; a secrets-fetching
init container, §5). Understanding "pod" as the deployable unit — not "container" — is
foundational to reasoning correctly about Kubernetes' scheduling, networking, and
lifecycle semantics for any non-trivial deployment.
```

---

## 2. Resource Requests & Limits for JVM Workloads

### 2.1 Requests vs Limits — A Distinction With Real JVM Consequences

```yaml
resources:
  requests:
    memory: "1Gi"   # GUARANTEED — the scheduler uses this to decide WHICH NODE can host
    cpu: "500m"        # this pod; this amount is RESERVED, even if unused
  limits:
    memory: "1Gi"      # the HARD CEILING — exceeding it triggers an OOMKill (File 1 §3.2)
    cpu: "1000m"         # exceeding CPU limit causes THROTTLING, not termination — a
                            # MEANINGFULLY different failure mode than exceeding memory
```

> 🔧 **Setting `requests.memory == limits.memory`** (as above) is the standard recommendation for JVM workloads specifically — directly extending the Java JVM Internals guide's §3.6 "set `-Xms == -Xmx` in production" rationale to the CONTAINER level: a JVM that might be scheduled with LESS memory than it requested (if requests and limits differ and the node is under memory pressure) risks the exact container-OOM cycle from File 1 §3.2, while a JVM guaranteed its full limit upfront avoids ever discovering, under load, that its heap-sizing assumptions don't match what's actually available.

### 2.2 The CPU Throttling Trap — A Genuinely Common, Misdiagnosed Java Issue

```
CPU limits are enforced via the Linux CFS (Completely Fair Scheduler) using time-sliced
THROTTLING, NOT by reducing the perceived core count smoothly — a JVM with a CPU limit
of "2" can still legitimately use MULTIPLE cores simultaneously for brief bursts (e.g.,
during a STOP-THE-WORLD GC pause genuinely wanting to parallelize across many threads,
Java JVM Internals guide §4), then get THROTTLED hard for the remainder of that scheduling
period — manifesting as seemingly RANDOM latency spikes that correlate suspiciously well
with GC activity, but are actually a CPU-throttling artifact, not a GC-tuning problem.
Diagnose via the `container_cpu_cfs_throttled_periods_total` metric (if using
cAdvisor/Prometheus, File 6 of this series) BEFORE assuming a latency spike is purely
JVM/GC-internal — a common, costly misdiagnosis that leads teams to tune GC flags
extensively for a problem that's actually a Kubernetes resource-limit configuration issue.
```

### 2.3 Sizing Methodology — Directly Extending the Database Capacity Planning Discipline

```
Apply the EXACT SAME methodology from the Database Architecture File 5 capacity-planning
series to JVM pod sizing: characterize the workload (CPU-bound vs I/O-bound, Java
Concurrency guide §2.6), measure ACTUAL resource consumption under REALISTIC load (via a
load test, not guesswork), and size requests/limits with DELIBERATE headroom — never copy
a "reasonable-sounding" resource block from a tutorial without validating it against your
ACTUAL application's measured behavior under YOUR ACTUAL traffic shape.
```

---

## 3. Probes — Liveness, Readiness & Startup

### 3.1 The Three Probe Types, Precisely

```yaml
livenessProbe:    # "should this pod be RESTARTED?" — directly the Spring Testing &
  httpGet:           # Observability guide's §6.3 distinction: answers "is this process
    path: /actuator/health/liveness   # fundamentally broken" (deadlocked, unrecoverable)
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:    # "should TRAFFIC be routed to this pod RIGHT NOW?" — answers "is it
  httpGet:             # temporarily not ready" (still warming up, a downstream dependency
    path: /actuator/health/readiness   # is briefly unavailable) — failing this REMOVES
    port: 8080                            # the pod from Service load-balancing WITHOUT restarting it
  periodSeconds: 5

startupProbe:        # specifically for SLOW-STARTING applications (a JVM with a large
  httpGet:               # Spring context, lots of @PostConstruct initialization, JIT
    path: /actuator/health/readiness  # warm-up — Java JVM Internals guide §1.2) — liveness/
    port: 8080               # readiness checks are SUSPENDED until this succeeds, preventing
  failureThreshold: 30          # Kubernetes from prematurely killing a pod that's simply
  periodSeconds: 10                # still starting up, not actually unhealthy
```

### 3.2 The Startup-Probe Gap — A Frequent, Avoidable Production Incident

```
WITHOUT a startup probe, a slow-starting Spring Boot application (large dependency
injection graph, substantial cache warm-up, the Java Core Language guide's §10.1 cache-
warmup pattern at scale) can fail its LIVENESS probe simply because it hasn't finished
starting yet — Kubernetes then RESTARTS it, the restart takes the SAME long time to start,
fails the SAME liveness check again, and the pod enters a CRASH-LOOP, never actually
becoming healthy despite the application code being entirely correct. The startup probe's
EXISTENCE for exactly this scenario is a frequently-missed, easily-fixed gap — any
JVM application with non-trivial startup time should have one configured explicitly.
```

### 3.3 Probe Failure Thresholds — Avoiding Over-Eager Restarts

```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3   # requires 3 CONSECUTIVE failures (30 seconds) before restart —
                          # NOT a single transient blip — directly avoiding an over-eager
                          # restart for a momentary GC pause or brief downstream hiccup
                          # that would have recovered on its own within seconds
```

---

## 4. Horizontal Pod Autoscaling & the JVM Warm-Up Tension

### 4.1 The Standard HPA Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: order-service-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: order-service }
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
      # directly the Database Architecture File 5 §5.1 "target 60-70% peak utilization,
      # not 100%" capacity-planning discipline, applied as the AUTOSCALING THRESHOLD itself
```

### 4.2 The JVM-Specific Scaling Tension — Cold Pods Are Slow Pods

```
Directly extending the Java JVM Internals guide's §1.2 JIT warm-up discussion to
AUTOSCALING specifically: a NEWLY-SCALED-UP pod starts execution INTERPRETED (Tier 0),
not yet JIT-compiled — meaning a fresh pod added during a traffic spike is, for its FIRST
few thousand requests, MEASURABLY SLOWER per-request than an already-running, already-
warmed-up pod. Scaling reactively on CPU/latency alone can therefore UNDERSERVE a spike's
PEAK moment specifically — the new capacity arrives, but isn't yet performing at full
speed when it's needed most. Mitigations: scale PROACTIVELY ahead of KNOWN traffic
patterns where possible (scheduled scaling for predictable peaks), tune `minReplicas` generously
enough that reactive scaling is rarely on the critical path, or — directly connecting to the
Java Concurrency guide's §5 discussion — adopt VIRTUAL THREADS where applicable, since
virtual-thread-based concurrency scales WITHIN already-running pods more effectively,
reducing reliance on horizontal pod-count scaling for I/O-bound load spikes specifically.
```

### 4.3 Custom Metrics — Scaling on Business/Application Signals, Not Just CPU

```yaml
metrics:
  - type: Pods
    pods:
      metric: { name: http_requests_queue_depth } # a CUSTOM metric (via Prometheus
      target: { type: AverageValue, averageValue: "10" }  # Adapter, File 6 of this series)
      # directly the Spring Testing & Observability guide's §7.2 "business metrics as an
      # early-warning signal" principle, applied to the SCALING DECISION itself rather
      # than just an alert — scaling on queue depth/request backlog often anticipates
      # load BEFORE it manifests as elevated CPU, giving more proactive scaling behavior
```

---

## 5. ConfigMaps, Secrets & Spring Configuration

### 5.1 ConfigMaps — Externalized, Non-Sensitive Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: order-service-config }
data:
  application.yml: |
    spring:
      jpa:
        hibernate:
          ddl-auto: validate
    app:
      payment:
        timeout-seconds: "5"
```

```yaml
# Mounted into the pod, consumed via Spring Boot's standard externalized-configuration
# precedence (Spring Core guide §4.4) — NO application code changes needed
volumeMounts:
  - name: config-volume
    mountPath: /app/config
volumes:
  - name: config-volume
    configMap: { name: order-service-config }
```

### 5.2 Secrets — Why a ConfigMap Is the Wrong Place for Credentials

```yaml
apiVersion: v1
kind: Secret
metadata: { name: order-service-db-credentials }
type: Opaque
stringData:
  username: app_user
  password: "${DB_PASSWORD}"  # populated via a SEALED SECRETS or External Secrets
                                  # Operator integration in practice — NEVER committed
                                  # to source control in plaintext, directly the Database
                                  # Architecture File 6 §2.3 secrets-management discipline
```

> ⚠️ **Kubernetes Secrets are base64-ENCODED, not ENCRYPTED, by default** — base64 is trivially reversible, not a security control. Production deployments require EITHER encryption-at-rest enabled for the cluster's etcd store (a cluster-admin-level configuration) OR an external secrets manager (HashiCorp Vault, AWS Secrets Manager via the External Secrets Operator) — directly the SAME envelope-encryption/KMS discipline from the Database Architecture File 6 §2.2, here applied to Kubernetes-native secret storage specifically.

### 5.3 Spring Cloud Kubernetes — Native Integration

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>
```

```
Enables a Spring Boot application to WATCH a ConfigMap for changes and trigger a
@RefreshScope-based reload (Spring Cloud guide §1.2) WITHOUT a separate Spring Cloud
Config Server at all — Kubernetes' own ConfigMap mechanism BECOMES the configuration
source, directly the SAME "decouple deployment from configuration change" benefit, with
Kubernetes itself as the configuration store rather than a dedicated Config Server.
```

---

## 6. Deployment Strategies

### 6.1 Rolling Update — The Kubernetes Default

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # at most 1 pod down at a time during the rollout
      maxSurge: 1            # at most 1 EXTRA pod beyond desired count during the rollout
```

```
Directly the Java Testing & Production Readiness guide's §6.2 rolling-deployment strategy,
now expressed as native Kubernetes configuration — OLD and NEW pod versions run
SIMULTANEOUSLY during the rollout, meaning the SAME mixed-version-compatibility constraint
from that section (and the Spring Data guide's §8.2 expand-contract migration discipline)
applies directly: a schema change incompatible with the OLD application version will break
old-version pods mid-rollout, regardless of how careful the Kubernetes-level rollout
configuration is — the database-migration discipline and the deployment-strategy
discipline must be coordinated, not treated as independent concerns.
```

### 6.2 Blue-Green and Canary via Service Mesh or Ingress Annotations

```yaml
# A canary deployment via a service mesh's traffic-splitting capability (Istio/Linkerd,
# Spring Cloud guide §8.4) — directly the Java Testing & Production Readiness guide's
# §6.2 canary strategy, with Kubernetes/the mesh handling the percentage-based traffic split
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  http:
    - route:
        - destination: { host: order-service, subset: v1 }
          weight: 90
        - destination: { host: order-service, subset: v2 }
          weight: 10   # 10% of traffic to the new version — gradually increased as
                          # confidence grows, exactly the canary discipline from that section
```

---

## 7. Graceful Termination & Pod Disruption Budgets

### 7.1 The Termination Sequence, Precisely

```
1. Pod marked for termination — IMMEDIATELY removed from Service endpoints (stops
   receiving NEW traffic) — but EXISTING in-flight requests continue being served
2. SIGTERM sent to the container's main process — directly File 1 §6.2's graceful-
   shutdown handler, triggered here
3. `terminationGracePeriodSeconds` window (default 30s) for the application to finish
   in-flight work and shut down cleanly
4. If still running after that window, SIGKILL — an ABRUPT, non-negotiable termination
```

```yaml
spec:
  terminationGracePeriodSeconds: 45 # tuned to comfortably exceed your application's
                                       # ACTUAL measured graceful-shutdown duration —
                                       # directly the SAME "measure, don't guess" discipline
                                       # as every capacity-planning decision in this series
```

### 7.2 Pod Disruption Budgets — Protecting Availability During VOLUNTARY Disruptions

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: order-service-pdb }
spec:
  minAvailable: 2  # Kubernetes will NOT voluntarily evict pods (node drains, cluster
  selector:           # upgrades) if doing so would drop below this — directly protecting
    matchLabels: { app: order-service }  # against a well-intentioned cluster-maintenance
                                            # operation accidentally taking down MORE
                                            # capacity than the service can tolerate,
                                            # exactly the availability-during-maintenance
                                            # concern the deployment-strategy discipline (§6) addresses
```

---

## 8. StatefulSets — When Java Workloads Need Them

### 8.1 The Distinction From Deployments

```
A Deployment's pods are INTERCHANGEABLE — any pod can be killed and replaced by an
identical one, with NO assumption of stable identity or storage. A STATEFULSET provides
STABLE network identity (predictable pod names: `app-0`, `app-1`, ...) and STABLE,
PERSISTENT storage PER POD (a PersistentVolumeClaim that follows a specific pod identity
across restarts) — directly relevant for self-hosted database/coordination-service
deployments (Database Architecture File 7's polyglot-persistence components, if
self-hosted ON Kubernetes rather than via a managed cloud service) where each replica
needs to retain ITS OWN specific data across restarts, unlike a stateless application tier.
```

### 8.2 Most Java APPLICATION Workloads Should Be Deployments, Not StatefulSets

```
A typical Spring Boot service — even one with SOME local state (an in-process Caffeine
cache, Spring Trending Libraries guide §8) — is usually still correctly modeled as a
STATELESS Deployment: the local cache is DISPOSABLE (rebuilt on restart, or backed by a
shared Redis layer for anything that genuinely needs to survive a pod restart, File 2 of
the database scenario series). Reach for StatefulSet specifically when a workload
genuinely needs PER-INSTANCE persistent identity/storage — self-hosted Kafka brokers,
self-hosted Cassandra nodes, self-hosted database replicas — not as a default for "the
application happens to hold some state in memory."
```

---

## 9. Namespaces & Multi-Tenancy

### 9.1 Namespace-Per-Environment, and the Multi-Tenant Extension

```
Directly extending the Spring Security guide's §8.1 multi-tenancy discussion to the
INFRASTRUCTURE layer: namespaces provide a RESOURCE-QUOTA and RBAC boundary, useful for
separating environments (dev/staging/production) AND, for genuinely isolated multi-tenant
platforms, potentially separating TENANTS at the infrastructure level (directly the
"shard by tenant" pattern from the Java System Design guide's §3.2, here applied to
Kubernetes namespace allocation rather than database sharding) — appropriate for FDE
engagements requiring strong tenant isolation guarantees beyond what application-level
multi-tenancy (Row-Level Security, Spring Security's tenant-scoped authorization) alone provides.
```

### 9.2 ResourceQuotas — Preventing One Namespace From Starving Others

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: tenant-acme-quota, namespace: tenant-acme }
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
```

Directly the Java Concurrency guide's §7.5 Bulkhead pattern, applied at the CLUSTER-RESOURCE level — one tenant's runaway workload (a bug causing unbounded pod scaling) cannot exhaust shared cluster capacity needed to serve OTHER tenants, exactly the naval-compartment isolation metaphor from that section, here enforced by the platform rather than application code.

---

## 10. Worked Manifest — A Production-Grade Spring Boot Deployment

Synthesizing every section above into one complete, annotated manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels: { app: order-service }
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 1, maxSurge: 1 } # §6.1
  selector:
    matchLabels: { app: order-service }
  template:
    metadata:
      labels: { app: order-service }
    spec:
      terminationGracePeriodSeconds: 45 # §7.1
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.4.2@sha256:abc123... # File 1 §8.3 — pinned digest
          ports: [{ containerPort: 8080 }]
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-XX:MaxRAMPercentage=75.0 -XX:+UseContainerSupport" # File 1 §3.1
            - name: DB_PASSWORD
              valueFrom: { secretKeyRef: { name: order-service-db-credentials, key: password } } # §5.2
          volumeMounts:
            - { name: config-volume, mountPath: /app/config } # §5.1
          resources: # §2.1, sized via the Database Architecture File 5 methodology
            requests: { memory: "1Gi", cpu: "500m" }
            limits: { memory: "1Gi", cpu: "1000m" }
          startupProbe: # §3.2
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            failureThreshold: 30
            periodSeconds: 10
          livenessProbe: # §3.1, §3.3
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe: # §3.1
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            periodSeconds: 5
      volumes:
        - { name: config-volume, configMap: { name: order-service-config } }
---
apiVersion: v1
kind: Service
metadata: { name: order-service }
spec:
  selector: { app: order-service }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: autoscaling/v2 # §4.1
kind: HorizontalPodAutoscaler
metadata: { name: order-service-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: order-service }
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
---
apiVersion: policy/v1 # §7.2
kind: PodDisruptionBudget
metadata: { name: order-service-pdb }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: order-service } }
```

Every line in this manifest traces back to a specific, deliberate decision covered above — the Principal-level discipline this series has insisted on throughout: nothing here is a copied-tutorial default left unexamined.
