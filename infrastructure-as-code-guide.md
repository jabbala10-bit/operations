# Infrastructure as Code (Terraform / Helm / Pulumi) — Principal Engineer Reference

> File 5 of 7 in the Cloud-Native & DevOps series. Covers declarative infrastructure management — the practice that makes the GitOps discipline from File 3 extend beyond application deployments to the underlying infrastructure itself.

**Series:** 1. Docker & Containerization · 2. Kubernetes for Java · 3. CI/CD & GitOps · 4. Cloud Provider Deep Dive · **5. Infrastructure as Code (this file)** · 6. Observability & SRE · 7. Cost Optimization & FinOps

---

## Table of Contents

1. [Why Infrastructure as Code, Precisely](#1-why-infrastructure-as-code-precisely)
2. [Terraform — State, Plan, Apply](#2-terraform--state-plan-apply)
3. [Module Design & Reusability](#3-module-design--reusability)
4. [Helm — Packaging Kubernetes Applications](#4-helm--packaging-kubernetes-applications)
5. [Kustomize vs Helm — A Real Choice](#5-kustomize-vs-helm--a-real-choice)
6. [Pulumi — General-Purpose-Language IaC](#6-pulumi--general-purpose-language-iac)
7. [Drift Detection & Remediation](#7-drift-detection--remediation)
8. [Testing Infrastructure Code](#8-testing-infrastructure-code)
9. [A Worked Module — Provisioning a Production Database](#9-a-worked-module--provisioning-a-production-database)

---

## 1. Why Infrastructure as Code, Precisely

### 1.1 The Chaos It Replaces

```
Manually click-ops-provisioned infrastructure (console-clicked VPCs, hand-configured
security groups, a database created "real quick" through a cloud console) has NO durable
record of WHY a given configuration exists, NO peer review before it's applied, and NO
reliable way to reproduce it in a second environment — directly the SAME "single source
of truth, never hand-edited out-of-band" discipline as the Database Architecture File 7
§5.2 schema-migration principle, here applied to the INFRASTRUCTURE layer itself.
```

### 1.2 The Direct Parallel to GitOps (File 3)

```
Infrastructure as Code is, architecturally, the SAME declarative-reconciliation philosophy
as File 3's GitOps discussion — applied ONE LAYER LOWER: GitOps (File 3) reconciles
APPLICATION deployments against Kubernetes manifests in Git; Infrastructure as Code
reconciles the UNDERLYING CLOUD RESOURCES (VPCs, databases, IAM roles, the Kubernetes
CLUSTER itself) against a declarative definition in Git. Both answer the question "what
SHOULD exist" from a version-controlled source, with a TOOL (Terraform, Pulumi) doing the
reconciliation — directly extending the Java System Design guide's broader declarative-
over-imperative theme one more layer down the stack.
```

---

## 2. Terraform — State, Plan, Apply

### 2.1 The Core Workflow

```hcl
resource "aws_db_instance" "orders" {
  identifier        = "orders-production"
  engine            = "postgres"
  engine_version    = "16.1"
  instance_class    = "db.r6g.xlarge"  # sized via the Database Architecture File 5
  allocated_storage = 500                # capacity-planning methodology, not guessed
  multi_az          = true
  storage_encrypted = true              # Database Architecture File 6 §1.1
}
```

```bash
terraform plan   # computes the DIFF between current STATE and desired CONFIGURATION —
                   # shows EXACTLY what will change, BEFORE anything actually changes —
                   # directly the "review before applying" discipline this entire series
                   # has insisted on for code changes, here extended to infrastructure changes
terraform apply    # applies ONLY the computed diff, idempotently
```

### 2.2 State — Why It's the Single Most Important, Most Fragile Concept

```
Terraform's STATE FILE is its record of what it BELIEVES currently exists — every
plan/apply operation is computed AGAINST this state, not by re-querying the cloud
provider for ground truth on every operation. This means: (1) state MUST be stored in a
SHARED, LOCKED backend (S3 + DynamoDB for locking, Azure Storage, GCS) for ANY team larger
than one person — a LOCAL state file edited by two engineers simultaneously is a direct
recipe for state corruption and conflicting infrastructure changes; (2) state can DRIFT
from actual cloud reality if resources are modified OUTSIDE Terraform (directly the §7
drift-detection discussion); (3) state files often contain SENSITIVE values (database
passwords set via a resource attribute) and must be treated with the SAME access-control
rigor as the Database Architecture File 6 §2.3 secrets-management discipline — never
committed to source control in plaintext.
```

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "production/database/terraform.tfstate"
    region         = "eu-central-1"  # Frankfurt — directly the File 4 §7.1 EU data-
    encrypt        = true               # residency consideration, applied to WHERE THE
    dynamodb_table = "terraform-locks"     # STATE ITSELF lives, not just application data
  }
}
```

### 2.3 Plan Review as a Mandatory Gate — Never `apply` Without Reviewing `plan`

```
Directly the Java Testing & Production Readiness guide's §6.1 "review before production
deploy" discipline, applied to infrastructure: a `terraform plan` output showing an
UNEXPECTED "destroy and recreate" action on a STATEFUL resource (a database, a persistent
volume) is a CRITICAL signal to STOP and investigate before approving — Terraform will,
in some circumstances (a changed attribute that FORCES resource replacement rather than
an in-place update), silently plan to DELETE AND RECREATE a database if the configuration
change isn't understood to require it — a `plan` review is the LAST checkpoint before
that becomes an irreversible, data-loss-causing `apply`.
```

---

## 3. Module Design & Reusability

### 3.1 Encapsulating a Reusable Pattern

```hcl
# modules/postgres-rds/main.tf — a REUSABLE module, parameterized, encapsulating the
# Database Architecture File 6 security defaults (encryption, private networking) and
# File 5 capacity-planning conventions as BUILT-IN, hard-to-bypass defaults
variable "instance_class" { type = string }
variable "allocated_storage" { type = number }
variable "environment" { type = string }

resource "aws_db_instance" "this" {
  identifier            = "${var.environment}-${var.service_name}"
  instance_class        = var.instance_class
  allocated_storage     = var.allocated_storage
  storage_encrypted     = true                  # NOT a variable — a non-negotiable default,
  publicly_accessible   = false                    # directly the Database Architecture File 6
  deletion_protection   = var.environment == "production" ? true : false  # §1.1 baseline,
}                                                                              # enforced by the
                                                                                  # MODULE itself,
                                                                                  # not left to each
                                                                                  # caller's discretion
```

```hcl
# Usage — a calling team gets the security/architectural defaults "for free," and can
# only customize the EXPLICITLY exposed variables
module "orders_db" {
  source            = "../modules/postgres-rds"
  service_name      = "orders"
  environment       = "production"
  instance_class    = "db.r6g.xlarge"
  allocated_storage = 500
}
```

### 3.2 Why Module-Level Defaults Beat Documentation-Level Conventions

```
Directly the Database Architecture File 6 §4.3 "defense in depth" principle, applied to
INFRASTRUCTURE PROVISIONING itself: a written "always enable encryption" policy in a wiki
page relies on EVERY engineer remembering and following it, every time; a hard-coded
default WITHIN the reusable module makes the secure configuration the ONLY EASY PATH —
an engineer would have to ACTIVELY work against the module (or avoid it entirely) to
provision an unencrypted, publicly-accessible database, directly the SAME "make the safe
path the default, not an opt-in" discipline as the Java System Design guide's §10.4 agent-
tool ALLOWLIST framing, here applied to infrastructure security posture.
```

---

## 4. Helm — Packaging Kubernetes Applications

### 4.1 Templating Kubernetes Manifests for Reuse Across Environments

```yaml
# templates/deployment.yaml — a Helm template, parameterized via values
apiVersion: apps/v1
kind: Deployment
metadata: { name: {{ .Release.Name }}-order-service }
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: order-service
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            requests: { memory: {{ .Values.resources.requests.memory }}, cpu: {{ .Values.resources.requests.cpu }} }
```

```yaml
# values-production.yaml — environment-specific overrides, directly the SAME
# externalized-configuration precedence philosophy as the Spring Core guide's §4.4,
# now applied to the KUBERNETES MANIFEST TEMPLATE layer rather than application properties
replicaCount: 5
image: { repository: registry.example.com/order-service, tag: "1.4.2" }
resources: { requests: { memory: "1Gi", cpu: "500m" } }
```

```bash
helm install order-service ./order-service-chart -f values-production.yaml
```

### 4.2 Helm Charts as the GitOps Config-Repository Artifact

```
Directly connecting to File 3 §2.3: the CONFIGURATION repository GitOps watches often
contains, or REFERENCES, a Helm chart + a per-environment values file, rather than raw,
fully-expanded Kubernetes YAML — giving the File 2 manifest's full templating/reuse power
WITHIN the GitOps reconciliation model, rather than treating templating and GitOps as
separate, unrelated concerns.
```

---

## 5. Kustomize vs Helm — A Real Choice

### 5.1 The Philosophical Difference

```
HELM — a TEMPLATING engine: define a chart with PLACEHOLDER values, render different
  output per environment by substituting DIFFERENT values — powerful, but the templates
  themselves can become genuinely complex (conditional logic embedded in YAML-generating
  Go templates), and the chart author must anticipate EVERY axis of variation upfront.

KUSTOMIZE — a PATCHING/OVERLAY engine: define a BASE set of plain, valid Kubernetes YAML,
  then apply environment-specific PATCHES (add a label, change a replica count, add an
  env var) on top — no templating language at all, just structural overlays on real YAML.
```

### 5.2 The Practical Decision

```
Favors Kustomize: teams wanting to avoid a templating language entirely, preferring to
  work with PLAIN, valid Kubernetes YAML at every layer (easier to read/`kubectl diff`
  directly), and where the variation between environments is genuinely simple (a few
  patched fields) rather than deeply parameterized.
Favors Helm: genuinely COMPLEX, widely-reused applications (especially THIRD-PARTY
  software you're CONSUMING, not authoring — most public Helm charts exist for exactly
  this reason) needing extensive, deeply-parameterized configurability, and teams that
  benefit from Helm's broader ecosystem (a public chart repository for common
  infrastructure components: ingress controllers, cert-manager, Prometheus).
Many platforms use BOTH: Helm for consuming third-party infrastructure components,
  Kustomize for their OWN application manifests' environment-specific overlays — directly
  the SAME "the right tool for the right layer" discipline as every other technology-
  choice discussion throughout this series, here resolved as "not mutually exclusive."
```

---

## 6. Pulumi — General-Purpose-Language IaC

### 6.1 Infrastructure Defined in Java Itself

```java
// Pulumi lets infrastructure be defined in an ACTUAL general-purpose language — directly
// relevant for a Java-centric platform team wanting infrastructure code to live in the
// SAME language, tooling, and code-review conventions as the application code itself
public class InfraStack {
    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var dbInstance = new DbInstance("orders-db", DbInstanceArgs.builder()
                .engine("postgres")
                .instanceClass("db.r6g.xlarge")
                .storageEncrypted(true)
                .build());
            ctx.export("dbEndpoint", dbInstance.endpoint());
        });
    }
}
```

### 6.2 The Trade-off Versus Terraform's HCL

```
Favors Pulumi: a team wanting to apply ORDINARY SOFTWARE ENGINEERING practices (unit
  testing infrastructure logic with JUnit, Java Testing guide §1-2; reusable functions/
  classes rather than HCL's more limited module system; the team's EXISTING Java IDE/
  tooling fluency) directly to infrastructure code, and genuinely complex, programmatic
  infrastructure-generation logic (looping, conditional resource creation based on
  complex business logic) that HCL's more constrained expression language handles awkwardly.
Favors Terraform: the LARGER, more mature ecosystem (provider coverage, community modules,
  hiring-market familiarity — directly the Database Architecture File 7 §3.2 "talent
  availability" factor, applied to IaC tooling choice itself) and a deliberately
  CONSTRAINED, declarative-only language that's HARDER to accidentally write overly-clever,
  hard-to-reason-about infrastructure logic in (a genuine, if double-edged, benefit of HCL's
  relative simplicity compared to a full general-purpose language).
```

---

## 7. Drift Detection & Remediation

### 7.1 Why Drift Happens Despite Best Intentions

```
Directly extending File 3 §3.2's GitOps-drift discussion to the INFRASTRUCTURE layer:
an emergency manual change (a security-group rule added directly via the cloud console
during an active incident), a different team's separate Terraform configuration
accidentally touching the SAME resource, or a cloud provider's own automatic changes
(a managed service's background maintenance modifying SOME attribute) can all cause the
ACTUAL infrastructure to diverge from what the Terraform STATE believes is true.
```

### 7.2 Detecting Drift Proactively

```bash
terraform plan -refresh-only   # refreshes STATE from actual cloud reality and shows
                                  # the DIFFERENCE — directly a DRIFT REPORT, run on a
                                  # SCHEDULE (e.g., nightly, in CI) rather than only when
                                  # someone happens to suspect a problem — the SAME proactive-
                                  # detection discipline as the Java JVM Internals guide's
                                  # §9.4 jstat continuous-monitoring recommendation, applied
                                  # to infrastructure-configuration drift specifically
```

### 7.3 The Remediation Discipline

```
Upon detecting drift: NEVER simply `terraform apply` to forcibly revert it without
understanding WHY it happened first — directly the Java JVM Internals guide's §10.4
"name the specific reference chain/root cause" discipline: a drift caused by a legitimate,
necessary emergency change should be REFLECTED BACK into the Terraform configuration
(updating the CODE to match the new, intentional reality) rather than blindly reverted,
while a drift caused by an accidental or unauthorized change SHOULD be reverted — the
distinction matters, and conflating them risks either silently undoing a legitimate fix
or permanently codifying an unauthorized change as if it were intended.
```

---

## 8. Testing Infrastructure Code

### 8.1 Static Validation — The Fastest, Cheapest Check

```bash
terraform validate    # syntax/internal-consistency check — catches malformed
tflint                  # configuration before it ever reaches a real cloud API call —
checkov -d .              # directly the Java Testing & Production Readiness guide's §6.1
                            # "fail fast, cheapest checks first" pipeline-ordering principle,
                            # here applied to infrastructure CI specifically; checkov/tfsec-
                            # style tools additionally scan for SECURITY misconfigurations
                            # (a publicly-accessible database, an overly-permissive IAM
                            # policy) statically, before any resource is ever provisioned
```

### 8.2 Integration Testing — Provisioning Real (Ephemeral) Infrastructure

```go
// Terratest (Go-based, but testing Terraform/HCL infrastructure regardless of the
// APPLICATION's language) — directly the Spring Testing & Observability guide's §3
// Testcontainers philosophy, applied to INFRASTRUCTURE rather than application
// dependencies: provision REAL, ephemeral cloud resources in a test, validate actual
// behavior, then DESTROY them — genuine fidelity, at real cloud-cost and TIME expense,
// reserved for the most critical, high-stakes modules (network topology, security
// configurations) rather than every single infrastructure change
func TestDatabaseModule(t *testing.T) {
    terraformOptions := &terraform.Options{ TerraformDir: "../modules/postgres-rds" }
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    endpoint := terraform.Output(t, terraformOptions, "endpoint")
    // assert connectivity, encryption-in-transit, etc. against the REAL provisioned resource
}
```

---

## 9. A Worked Module — Provisioning a Production Database

Synthesizing every section into a complete, production-grade Terraform module, directly implementing the Database Architecture File 5 capacity-planning and File 6 security disciplines as enforced infrastructure code:

```hcl
# modules/postgres-rds/main.tf
variable "service_name" { type = string }
variable "environment" { type = string }
variable "instance_class" { type = string }     # sized via Database Architecture File 5
variable "allocated_storage" { type = number }    # capacity-planning methodology
variable "vpc_id" { type = string }
variable "subnet_ids" { type = list(string) }

resource "aws_db_subnet_group" "this" {
  name       = "${var.environment}-${var.service_name}"
  subnet_ids = var.subnet_ids  # PRIVATE subnets only — File 4 §4.3 private-connectivity discipline
}

resource "aws_security_group" "db" {
  name   = "${var.environment}-${var.service_name}-db"
  vpc_id = var.vpc_id
  ingress {
    from_port = 5432
    to_port   = 5432
    protocol  = "tcp"
    security_groups = [var.app_security_group_id] # ONLY the application's own security
  }                                                    # group may connect — least privilege,
}                                                          # Database Architecture File 6 §3.2

resource "aws_kms_key" "db" {
  description         = "${var.environment}-${var.service_name}-db-encryption"
  enable_key_rotation = true  # Database Architecture File 6 §2.3 key-rotation discipline
}

resource "aws_db_instance" "this" {
  identifier              = "${var.environment}-${var.service_name}"
  engine                  = "postgres"
  engine_version          = "16.1"
  instance_class          = var.instance_class
  allocated_storage       = var.allocated_storage
  storage_encrypted       = true
  kms_key_id              = aws_kms_key.db.arn
  multi_az                = var.environment == "production"
  publicly_accessible     = false
  db_subnet_group_name    = aws_db_subnet_group.this.name
  vpc_security_group_ids  = [aws_security_group.db.id]
  backup_retention_period = var.environment == "production" ? 30 : 7  # Database
  deletion_protection     = var.environment == "production"             # Architecture
  performance_insights_enabled = true                                      # File 6 §9
}

output "endpoint" { value = aws_db_instance.this.endpoint }
```

Every line traces to a specific, deliberate decision from earlier in this series — exactly the discipline this entire reference set has insisted on: infrastructure code is not exempt from the same rigor applied to application code.
