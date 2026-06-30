# Cloud Provider Deep Dive (AWS / Azure / GCP) for Java Workloads — Principal Engineer Reference

> File 4 of 7 in the Cloud-Native & DevOps series. A decision-framework-first comparison across the three major clouds for Java/Spring platforms — with explicit attention to EU data-residency considerations given the relocation and BFSI/healthcare context this series has built toward throughout.

**Series:** 1. Docker & Containerization · 2. Kubernetes for Java · 3. CI/CD & GitOps · **4. Cloud Provider Deep Dive (this file)** · 5. Infrastructure as Code · 6. Observability & SRE · 7. Cost Optimization & FinOps

---

## Table of Contents

1. [Compute Options Compared](#1-compute-options-compared)
2. [Managed Kubernetes — EKS vs AKS vs GKE](#2-managed-kubernetes--eks-vs-aks-vs-gke)
3. [Serverless Java](#3-serverless-java)
4. [Networking Fundamentals, Cross-Cloud](#4-networking-fundamentals-cross-cloud)
5. [Managed Database Services](#5-managed-database-services)
6. [Identity & Access Management](#6-identity--access-management)
7. [EU Data Residency & Sovereign Cloud Considerations](#7-eu-data-residency--sovereign-cloud-considerations)
8. [Multi-Cloud & Cloud-Agnostic Architecture](#8-multi-cloud--cloud-agnostic-architecture)
9. [A Decision Framework for FDE Engagements](#9-a-decision-framework-for-fde-engagements)

---

## 1. Compute Options Compared

### 1.1 The Spectrum, Across All Three Providers

```
RAW VMs (EC2 / Azure VMs / Compute Engine) — full control, full operational burden — directly
  the System Design guide's §3.3 "managed vs self-hosted" build-vs-buy spectrum at its
  MOST self-hosted extreme. Appropriate when you need OS-level control a managed service
  doesn't permit, or are migrating an existing VM-based deployment with minimal re-architecture.

CONTAINER ORCHESTRATION (EKS/ECS / AKS / GKE) — covered in depth in File 2 (Kubernetes)
  of this series — the dominant choice for genuinely cloud-native Java/Spring platforms today.

PLATFORM-AS-A-SERVICE (Elastic Beanstalk / Azure App Service / App Engine) — deploy a
  JAR/WAR directly, the PROVIDER manages the underlying compute/scaling/patching entirely —
  the LEAST operational burden, at the cost of the LEAST infrastructure control (no custom
  sidecar containers, more constrained networking customization).

SERVERLESS FUNCTIONS (Lambda / Azure Functions / Cloud Functions) — covered in §3 —
  appropriate for genuinely event-driven, intermittent workloads, with real JVM-specific
  cold-start considerations.
```

### 1.2 The Decision Heuristic

```
Directly extending the Java System Design guide's §4.3 monolith-vs-microservices judgment-
call framing to the COMPUTE-PLATFORM choice itself: default to managed Kubernetes (§2) for
any genuinely microservices-shaped, multi-team platform with sustained traffic; default to
PaaS (Elastic Beanstalk/App Service) for a SIMPLER, single-team service where Kubernetes'
operational complexity isn't yet justified by the System Design guide's §4.3 standards
("a team of 5 engineers... is very likely paying the complexity tax for benefits they
don't actually need yet"); reach for serverless (§3) specifically for intermittent,
event-driven workloads where PAYING FOR IDLE CAPACITY (the dominant cost model for VMs/
Kubernetes nodes sitting at low utilization) is the primary problem to solve.
```

---

## 2. Managed Kubernetes — EKS vs AKS vs GKE

### 2.1 What's Actually Different Beneath a Largely-Identical Kubernetes API Surface

```
The Kubernetes API itself (File 2 of this series) is LARGELY IDENTICAL across all three —
the genuine differentiators are in the SURROUNDING managed-service integration:

EKS (AWS) — deepest integration with the broadest AWS service catalog (IAM Roles for
  Service Accounts, ALB Ingress Controller, EBS/EFS CSI drivers) — the natural choice
  for a platform ALREADY committed to AWS infrastructure broadly.

AKS (Azure) — notably tight integration with Azure AD for cluster RBAC (directly relevant
  to enterprise customers ALREADY standardized on Azure AD/Entra ID for identity, a common
  reality in EU enterprise/BFSI environments specifically) and Azure Policy for governance —
  often the path of least resistance for engagements where the CUSTOMER's existing identity
  infrastructure is Microsoft-centric.

GKE (Google Cloud) — generally regarded as the most "Kubernetes-native" experience, since
  Google originated Kubernetes — strong Autopilot mode (a more fully-managed node-operations
  experience, reducing the File 2 §2 node-level capacity-planning burden further than the
  other two providers' equivalent offerings).
```

### 2.2 The Practical FDE Reality

```
Directly the Database Architecture File 7 §3.3 "managed service choice is frequently
constrained by the customer's existing cloud-provider relationship rather than a purely
technical preference" framing, applied here: for an FDE engagement, the choice is
OVERWHELMINGLY determined by WHICH cloud the customer ALREADY operates in (and, per §7 of
this file, which cloud satisfies their specific EU data-residency posture) — fluency
across all three, rather than a strong preference for one, is the genuinely valuable
Principal/FDE skill, since you rarely get to choose the customer's existing cloud commitment.
```

---

## 3. Serverless Java

### 3.1 The Cold-Start Problem — A Direct, Severe Consequence of JVM Warm-Up

```
Directly extending the Java JVM Internals guide's §1.2 JIT-warm-up discussion to its MOST
consequential real-world manifestation: a serverless function's COLD START (a fresh JVM
process, fresh classloading, fresh interpreted-then-JIT-compiling execution, Java JVM
Internals guide §2) can add HUNDREDS of milliseconds to SECONDS of latency to the FIRST
invocation after a period of inactivity — directly the scenario the Java JVM Internals
guide's §1.2 flagged as disproportionately punishing "short-lived CLI tools/serverless
cold-starts" specifically, now explained as a first-order ARCHITECTURAL consideration for
choosing serverless Java at all, not a minor footnote.
```

### 3.2 Mitigations, Ranked by Effectiveness

```
1. GRAALVM NATIVE IMAGE (Spring Trending Libraries guide §5) — eliminates classloading/
   JIT-warm-up cold start almost entirely, since there's no JIT to warm up at all —
   the SINGLE most effective mitigation, at the real cost of that section's native-image
   trade-offs (build complexity, loss of runtime JIT specialization for sustained workloads).
2. PROVISIONED CONCURRENCY (AWS Lambda) / "Always Ready" instances (Azure Functions Premium) —
   the cloud provider keeps a configured number of execution environments WARM continuously,
   directly paying for the JVM Internals guide's §1.2 warm-up cost UPFRONT and CONTINUOUSLY,
   trading the "pay only when invoked" serverless cost benefit for consistent low latency —
   a genuine, deliberate trade-off, not a free improvement.
3. MINIMIZING DEPENDENCY/CLASSPATH SIZE — fewer classes to load means less classloading
   overhead (Java JVM Internals guide §2) even WITHOUT native image — a real, if more
   modest, lever available regardless of which other mitigations are applied.
```

### 3.3 When Serverless Java Is (and Isn't) the Right Choice

```
Favors serverless: genuinely INTERMITTENT, EVENT-DRIVEN workloads (a webhook handler
  invoked a few times per hour, a scheduled batch job) where idle-capacity cost (§7 of
  this series, FinOps) would otherwise dominate, and where COLD-START LATENCY is
  ACCEPTABLE for the use case (an asynchronous background job tolerates it far better
  than a synchronous, user-facing API call).
Favors a long-running compute platform (§1-2): sustained traffic, latency-sensitive
  synchronous APIs, or workloads where the JIT's STEADY-STATE optimization benefit
  (Java JVM Internals guide §5) — only available to a LONG-RUNNING process — meaningfully
  outweighs serverless's idle-cost savings.
```

---

## 4. Networking Fundamentals, Cross-Cloud

### 4.1 The Conceptual Mapping

| Concept | AWS | Azure | GCP |
|---|---|---|---|
| Isolated network | VPC | VNet | VPC |
| Subnet-level traffic control | Security Groups | Network Security Groups | Firewall Rules |
| Layer 7 load balancing | Application Load Balancer | Application Gateway | Cloud Load Balancing (HTTP(S)) |
| Private connectivity to managed services | VPC Endpoints | Private Endpoints | Private Service Connect |
| Cross-region/VPC connectivity | Transit Gateway | Virtual WAN | Network Connectivity Center |

### 4.2 Why This Matters for the Resilience Patterns Already Established

```
Every System Design guide §7 resilience pattern (timeouts, circuit breakers, bulkheads)
assumes a NETWORK TOPOLOGY where service-to-service latency/reliability characteristics
are reasonably predictable — directly connecting cloud NETWORKING choices to those
patterns' actual EFFECTIVENESS: a poorly-designed VPC/VNet topology (excessive cross-AZ
or cross-region hops for what should be co-located services) can introduce LATENCY
VARIANCE that makes timeout-tuning (System Design guide §7.5) genuinely harder to get
right, since the "normal" latency baseline itself becomes less predictable.
```

### 4.3 Private Connectivity to Managed Services — A Security AND Latency Concern

```
Directly extending the Database Architecture File 6 §1.2 "TLS between every hop"
discipline: using VPC Endpoints/Private Endpoints/Private Service Connect to reach a
managed database/storage service WITHOUT traversing the public internet at all gives
BOTH a security benefit (traffic never exposed to public routing) AND, often, a LATENCY
benefit (a shorter, more predictable network path) — a genuinely high-value, frequently-
underused configuration that should be a DEFAULT consideration for any managed-service
integration, not an advanced, rarely-applied option.
```

---

## 5. Managed Database Services

### 5.1 The Mapping to the Database Architecture Series

| Category (Database Architecture series) | AWS | Azure | GCP |
|---|---|---|---|
| Relational (File 1) | RDS (Postgres/MySQL), Aurora | Azure Database for PostgreSQL/MySQL | Cloud SQL, AlloyDB |
| Document (File 2) | DocumentDB | Cosmos DB (API for MongoDB) | Firestore |
| Key-Value/Cache (File 2) | ElastiCache (Redis), DynamoDB | Azure Cache for Redis, Cosmos DB | Memorystore |
| Wide-Column (File 2) | Keyspaces (Cassandra-compatible) | Cosmos DB (Cassandra API) | Bigtable |
| Graph (File 3) | Neptune | (Cosmos DB Gremlin API) | (No first-party managed graph DB — third-party on GKE) |
| Vector (File 4) | OpenSearch (vector engine), RDS pgvector | Azure AI Search, Cosmos DB DiskANN | AlloyDB/Cloud SQL pgvector, Vertex AI Vector Search |

### 5.2 Aurora & Cloud-Native Relational Engines — A Genuine Architectural Difference, Not Just "Managed Postgres"

```
AWS Aurora (and similarly, GCP's AlloyDB) separate COMPUTE from STORAGE at the architecture
level — storage is REPLICATED across multiple availability zones by the underlying
distributed storage layer ITSELF, independent of how many "read replica" compute nodes
exist — a genuinely different replication architecture from the traditional streaming-
WAL-replication model (Database Architecture File 1 §7.1) underlying standard RDS
PostgreSQL/MySQL. This typically gives FASTER failover (a new compute node can attach to
ALREADY-replicated storage near-instantly, rather than needing to catch up via WAL replay)
and easier read-replica scaling — a concrete reason "Aurora vs RDS Postgres" is a genuine
architectural decision, not just a branding difference, worth understanding before
defaulting to one or the other for an EU enterprise customer's relational workload.
```

### 5.3 Cosmos DB's Multi-Model Nature — A Distinctive Azure Capability

```
Cosmos DB's ability to expose the SAME underlying storage through MULTIPLE APIs (a
MongoDB-compatible document API, a Cassandra-compatible wide-column API, a Gremlin graph
API, a native SQL/NoSQL API) is architecturally distinctive — directly relevant when an
Azure-committed customer wants polyglot-persistence-style flexibility (Database
Architecture File 7 §1) WITHOUT operating multiple genuinely separate database PRODUCTS —
a real, Azure-specific trade-off worth knowing: less battle-tested per-API-model maturity
than a purpose-built engine (e.g., Cosmos DB's Cassandra API doesn't have CASSANDRA's full
two-decade operational track record), in exchange for genuine operational consolidation.
```

---

## 6. Identity & Access Management

### 6.1 The Conceptual Mapping

| Concept | AWS | Azure | GCP |
|---|---|---|---|
| Identity provider | IAM | Azure AD / Entra ID | Cloud IAM |
| Workload identity (pod-to-cloud-service auth) | IRSA (IAM Roles for Service Accounts) | Workload Identity (AKS) | Workload Identity (GKE) |
| Fine-grained resource policy | IAM Policies | Azure RBAC + ABAC | IAM Policies + Conditions |

### 6.2 Workload Identity — Eliminating Static Cloud Credentials Entirely

```yaml
# Directly extending the Database Architecture File 6 §2.3 secrets-rotation discipline to
# its STRONGEST possible form: rather than injecting a static cloud-provider access key as
# a Kubernetes Secret (File 2 §5.2, itself a real but lesser improvement over hardcoding),
# Workload Identity lets a pod's Kubernetes SERVICE ACCOUNT be DIRECTLY, NATIVELY mapped to
# a cloud IAM identity — the pod authenticates to AWS/Azure/GCP services with NO static
# credential material EVER present in the cluster at all, eliminating an entire category
# of credential-leakage risk (a long-lived access key committed to a config file, a Secret
# accidentally exposed) by removing the static credential from existence entirely
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-service-role
    # (the equivalent annotation differs slightly for AKS/GKE workload identity, same principle)
```

### 6.3 Why This Should Be the Default, Not an Advanced Optimization

```
Given the Database Architecture File 6 §2.3 framing of credential rotation/management as
a genuinely high-value, frequently-under-implemented control, Workload Identity represents
the LOGICAL endpoint of that discipline — it's worth treating as the DEFAULT expectation
for any new Kubernetes-based platform, with static, long-lived cloud credentials treated
as a deliberate, justified exception (e.g., a legacy system genuinely unable to use it)
rather than the default starting point.
```

---

## 7. EU Data Residency & Sovereign Cloud Considerations

> As with the AI/Agentic guide's File 6 and the Database Architecture guide's File 6, this section is informational/architectural, not legal advice — confirm specific obligations with qualified legal/compliance counsel for any actual engagement.

### 7.1 Region Selection as a First-Order Architecture Decision

```
Directly extending the Java System Design guide's §10.7 EU AI Act data-residency
discussion to CLOUD-REGION selection broadly: for ANY EU customer engagement (directly
relevant given the Germany/Netherlands relocation context this entire reference series
has been built around), confirm EARLY which specific EU region(s) a workload must run
in, and whether ANY data (including backups, logs, and — directly the AI/Agentic guide's
File 6 §1.1 framing — embeddings/derived data) is permitted to transit or be processed
OUTSIDE the EU at all, even transiently (e.g., a managed service's control-plane
metadata, or a third-party SaaS dependency's own infrastructure footprint).
```

### 7.2 Sovereign Cloud Offerings — A Distinct, Evolving Category

```
All three major providers offer (or partner on) SOVEREIGN CLOUD variants specifically
targeting EU data-sovereignty requirements beyond standard regional deployment — AWS
European Sovereign Cloud, Microsoft's EU Data Boundary commitments (and Azure's broader
sovereign-cloud partnerships), and Google Cloud's sovereign-controls offerings (often via
local partners in specific EU jurisdictions). These typically address requirements BEYOND
simple regional data storage — operational/support-staff residency, independence from
non-EU control-plane dependencies — directly relevant for the MOST stringent BFSI/public-
sector engagements, and worth flagging as a DISTINCT evaluation category from "just pick
an EU region," since standard regional deployment does NOT automatically satisfy every
sovereign-cloud-grade requirement a specific customer or regulator might impose.
```

### 7.3 BaFin and EU Banking-Sector Cloud Outsourcing Rules

```
Directly extending the Spring Cloud guide's §8 (closing FDE scenario) and the Database
Architecture File 6 §5.1 BaFin discussion to cloud-PROVIDER selection specifically: EU
banking-sector regulation (EBA Guidelines on Outsourcing, BaFin's specific implementing
guidance) imposes REQUIREMENTS on cloud-provider relationships themselves — audit rights,
exit-strategy planning (can the workload genuinely be migrated AWAY from this cloud
provider if needed), and concentration-risk considerations (is the institution becoming
overly dependent on a SINGLE cloud provider in a way regulators scrutinize) — these are
CONTRACTUAL and ARCHITECTURAL considerations that should be raised with the customer's
own compliance/legal function EARLY in a cloud-provider selection process, not discovered
during a later regulatory review.
```

---

## 8. Multi-Cloud & Cloud-Agnostic Architecture

### 8.1 The Honest Cost-Benefit, Restated for Cloud Provider Choice

```
Directly the SAME "every additional distinct technology adds real cost" discipline from
the Database Architecture File 7 §6.1, applied to CLOUD PROVIDERS specifically:
genuine multi-cloud architecture (workloads designed to run identically across two-plus
providers) trades AWAY each provider's most differentiated, highest-value managed
services (since cloud-agnostic code, by definition, can't depend on any one provider's
unique capability) in exchange for negotiating leverage and reduced single-provider
dependency risk — a trade-off that's GENUINELY justified for specific cases (the §7.3
BaFin concentration-risk concern, or a customer's explicit multi-cloud mandate) and
GENUINELY costly when adopted reflexively "to avoid lock-in" without a concrete,
articulated requirement driving it.
```

### 8.2 Kubernetes as the Practical Multi-Cloud Abstraction Layer

```
Where SOME multi-cloud portability genuinely is required, standardizing on Kubernetes
(File 2) as the deployment target — using ONLY broadly-portable Kubernetes-native
primitives, and treating cloud-specific managed services (§5's database mapping, for
instance) behind a Hexagonal-Architecture-style Port (Java Design Patterns guide §4.2)
that COULD be swapped — gives a meaningfully more achievable level of portability than
attempting to abstract over EVERY cloud-specific capability uniformly, directly the SAME
"abstract at the right LAYER, not everywhere" discipline as the Spring Cloud guide's §6.1
Spring Cloud Stream messaging-abstraction discussion.
```

---

## 9. A Decision Framework for FDE Engagements

```
1. Is the customer ALREADY committed to a specific cloud provider? → that is overwhelmingly
   the deciding factor (§2.2) — fluency across all three, not a preference for one, is the
   actual differentiating skill.

2. Does the engagement involve EU-regulated data (BFSI, healthcare)? → confirm region AND
   sovereign-cloud requirements (§7) EXPLICITLY and EARLY, with qualified legal/compliance
   counsel involved, before significant architecture investment.

3. Is the workload sustained-traffic and latency-sensitive, or genuinely intermittent/
   event-driven? → managed Kubernetes (§2) vs serverless (§3), per §1.2's heuristic.

4. Does the engagement have an explicit multi-cloud or cloud-exit-strategy requirement
   (often BFSI-regulatory-driven, §7.3)? → invest deliberately in the Kubernetes-centric
   portability discipline (§8.2); otherwise, embrace the chosen provider's differentiated
   managed services fully rather than constraining the architecture defensively.

5. For database/persistence choices specifically → apply the Database Architecture File 7
   decision framework FIRST (what category of database fits the access pattern), THEN map
   that category to the specific managed-service offering (§5.1's table) on the customer's
   chosen cloud — never let "which managed service happens to be easiest to provision"
   drive the underlying data-modeling decision.
```
