# CI/CD Pipelines & GitOps — Principal Engineer Reference

> File 3 of 7 in the Cloud-Native & DevOps series. Extends the Java Testing & Production Readiness guide's §6 CI/CD discussion into concrete pipeline design, GitOps architecture, and progressive-delivery automation.

**Series:** 1. Docker & Containerization · 2. Kubernetes for Java · **3. CI/CD & GitOps (this file)** · 4. Cloud Provider Deep Dive · 5. Infrastructure as Code · 6. Observability & SRE · 7. Cost Optimization & FinOps

---

## Table of Contents

1. [Pipeline Stage Design](#1-pipeline-stage-design)
2. [GitOps — Git as the Single Source of Truth](#2-gitops--git-as-the-single-source-of-truth)
3. [ArgoCD / Flux — Reconciliation in Practice](#3-argocd--flux--reconciliation-in-practice)
4. [Progressive Delivery Automation](#4-progressive-delivery-automation)
5. [Artifact Management & Versioning](#5-artifact-management--versioning)
6. [Environment Promotion Strategies](#6-environment-promotion-strategies)
7. [Secrets in CI/CD Pipelines](#7-secrets-in-cicd-pipelines)
8. [Rollback Strategies](#8-rollback-strategies)
9. [A Worked End-to-End Pipeline](#9-a-worked-end-to-end-pipeline)

---

## 1. Pipeline Stage Design

### 1.1 Fail-Fast Ordering, Restated Concretely

```yaml
# GitHub Actions — directly the Java Testing & Production Readiness guide's §6.1
# "order stages from fastest/cheapest to slowest/most expensive" principle
jobs:
  fast-checks:
    steps:
      - run: ./gradlew compileJava            # fails in seconds if it doesn't even compile
      - run: ./gradlew checkstyleMain spotbugsMain  # static analysis — fast, catches real bugs early

  unit-tests:
    needs: fast-checks
    steps:
      - run: ./gradlew test                    # fast, no external dependencies (Testing guide §1)

  integration-tests:
    needs: unit-tests
    steps:
      - run: ./gradlew integrationTest          # Testcontainers-backed (Spring Testing & Observability guide §3) — slower

  build-and-scan:
    needs: integration-tests
    steps:
      - run: ./gradlew bootBuildImage             # File 1's buildpacks/Dockerfile build
      - run: trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:${{ github.sha }} # File 1 §8.2

  deploy-staging:
    needs: build-and-scan
    # ... deployment steps, §6
```

### 1.2 Parallelization Within a Stage

```yaml
unit-tests:
  strategy:
    matrix:
      module: [order-service, inventory-service, payment-service] # for a multi-module
  steps:                                                              # build, Testing &
    - run: ./gradlew :${{ matrix.module }}:test                          # Production Readiness
                                                                             # guide §5.4 — run
                                                                             # independent modules'
                                                                             # test suites CONCURRENTLY
                                                                             # rather than sequentially,
                                                                             # directly cutting pipeline
                                                                             # wall-clock time
```

### 1.3 Caching to Avoid Redundant Work

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.gradle/caches
    key: gradle-${{ hashFiles('**/*.gradle*') }} # cache invalidates ONLY when build files
                                                     # change — directly the SAME dependency-
                                                     # layer-caching principle from File 1 §4.1,
                                                     # applied at the CI-RUNNER level rather
                                                     # than the Docker-image-layer level
```

---

## 2. GitOps — Git as the Single Source of Truth

### 2.1 The Core Principle, Precisely

```
GitOps means: the DESIRED state of EVERY environment (which image version, how many
replicas, which config) is declared ENTIRELY in a Git repository — and a CONTROLLER
(ArgoCD, Flux) continuously RECONCILES the actual cluster state to MATCH what's declared
in Git, rather than a CI pipeline directly executing `kubectl apply` as an IMPERATIVE,
one-shot push. This is a meaningfully different model from traditional "push-based" CI/CD —
directly the SAME declarative-vs-imperative philosophy as Kubernetes manifests themselves
(File 2 of this series): you declare WHAT you want, a controller continuously works to
make it SO, rather than scripting the individual STEPS to get there.
```

### 2.2 Why Reconciliation (Pull-Based) Beats Direct Push

```
A PUSH-based pipeline (CI directly runs `kubectl apply`) has NO ongoing verification that
the cluster's ACTUAL state still matches what was intended — a manual `kubectl edit`
applied directly to production (an emergency hotfix, an accidental change) silently DRIFTS
the cluster away from what Git says should be true, with NOTHING noticing or correcting it.
A PULL-based GitOps controller continuously COMPARES actual-vs-desired state and
AUTOMATICALLY corrects drift — directly the SAME self-healing principle as Kubernetes'
own reconciliation loops (a Deployment controller restoring a manually-deleted pod), just
extended to the ENTIRE deployed configuration, not just pod count.
```

### 2.3 Repository Structure — App Repo vs Config Repo

```
APPLICATION REPOSITORY — source code, Dockerfile/buildpack config, CI pipeline definition.
  CI builds and pushes a versioned IMAGE here, then UPDATES THE CONFIG REPOSITORY (below)
  with the new image tag — it does NOT directly deploy to the cluster itself.

CONFIGURATION REPOSITORY — Kubernetes manifests (or Helm/Kustomize definitions, File 5 of
  this series) for EVERY environment, with the CURRENT desired image tag committed
  explicitly. The GitOps controller watches THIS repository, not the application repository.
```

```
This SEPARATION is deliberate: it gives a CLEAN AUDIT TRAIL of every deployment (every
production change is a CONFIG-REPO commit, independently of how often application CODE
changes), directly the SAME audit-trail discipline as the AI/Agentic guide's File 6 §4
applied to DEPLOYMENT history specifically, and lets you ROLL BACK a deployment by simply
reverting a config-repo commit — without needing to rebuild or even understand the
application code change that's being rolled back (§8 covers this in full).
```

---

## 3. ArgoCD / Flux — Reconciliation in Practice

### 3.1 An ArgoCD Application Definition

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: order-service-production, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/example/order-service-config.git
    targetRevision: main
    path: environments/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true      # automatically REMOVE resources deleted from Git — true reconciliation,
      selfHeal: true     # automatically REVERT manual cluster drift back to Git's declared state
    syncOptions: [CreateNamespace=true]
```

### 3.2 The `selfHeal` Decision — A Real Trade-off, Not a Default to Enable Blindly

```
`selfHeal: true` means ANY manual `kubectl` change to a GitOps-managed resource gets
AUTOMATICALLY REVERTED by the controller — powerful for preventing drift, but it ALSO
means an emergency, hands-on hotfix applied directly to a struggling production pod
(common during active incident response) gets SILENTLY UNDONE moments later by the
reconciliation loop, potentially confusing an on-call engineer mid-incident who doesn't
expect their manual change to vanish. The Principal-level discipline: train the team
explicitly on this behavior, and establish an "emergency bypass" runbook (typically:
pause auto-sync temporarily, make the manual change, THEN commit the equivalent change
to Git before re-enabling auto-sync) rather than discovering this surprising behavior
mid-incident for the first time.
```

### 3.3 App-of-Apps — Managing Many Services' GitOps Configuration Coherently

```yaml
# A "root" Application that itself manages a DIRECTORY of OTHER Application definitions —
# directly the SAME hierarchical-composition principle as the Multi-Agent guide's §3
# hierarchical supervisor pattern, here applied to managing MANY microservices' GitOps
# configuration as one coherent, version-controlled structure rather than ad hoc per-service setup
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: platform-apps }
spec:
  source: { repoURL: https://github.com/example/platform-config.git, path: apps/ }
  # this directory contains ONE Application manifest PER microservice — adding a new
  # service to the platform means adding ONE file here, automatically picked up
```

---

## 4. Progressive Delivery Automation

### 4.1 Argo Rollouts — Automating the Canary Process

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: order-service }
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10           # 10% of traffic to the new version
        - pause: { duration: 5m }   # WAIT — and AUTOMATICALLY evaluate health (§4.2)
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      analysis:
        templates: [{ templateName: success-rate }] # §4.2
```

### 4.2 Automated Analysis — Closing the Loop Without a Human Watching a Dashboard

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate }
spec:
  metrics:
    - name: error-rate
      interval: 1m
      successCondition: result < 0.01  # directly the Java Testing & Production Readiness
      provider:                          # guide's §7.3 RED-metrics discipline, now wired
        prometheus:                        # DIRECTLY into the deployment-progression decision —
          query: |                            # an automated rollback triggers if error rate
            sum(rate(http_requests_total{job="order-service",status=~"5.."}[1m]))   # crosses
            / sum(rate(http_requests_total{job="order-service"}[1m]))                  # this
                                                                                            # threshold,
                                                                                            # with
                                                                                            # NO human
                                                                                            # needing to
                                                                                            # be actively
                                                                                            # watching at 2am
```

```
This is the FULL, automated realization of the canary-deployment discipline from the Java
Testing & Production Readiness guide's §6.2 — combined with the AI/Agentic guide's File 5
§7.1 "A/B testing and gradual rollout" principle, here generalized beyond just prompt/model
changes to ANY deployment. The controller AUTOMATICALLY progresses the rollout on success
and AUTOMATICALLY rolls back on a detected regression, closing the human-reaction-time gap
that a purely manual canary process depends on.
```

---

## 5. Artifact Management & Versioning

### 5.1 Semantic, Immutable Image Tagging

```
NEVER deploy based on a MUTABLE tag (`:latest`, `:staging`) — directly File 1 §8.3's
pinned-digest discipline, now applied across the FULL pipeline: tag every build with an
IMMUTABLE identifier (the Git commit SHA, or a semantic version derived from a release
process) so that "what's running in production RIGHT NOW" is always traceable back to an
EXACT, unambiguous source-code commit — essential for incident investigation (Java JVM
Internals guide §10.4) and for the rollback discipline in §8.
```

### 5.2 A Private Artifact Registry as a Supply-Chain Control Point

```
Directly extending the Java Testing & Production Readiness guide's §5.3 dependency-
management discipline to CONTAINER IMAGES specifically: a private registry (Harbor, AWS
ECR, Google Artifact Registry) with VULNERABILITY SCANNING (File 1 §8.2) and SIGNED-IMAGE
verification (Cosign/Notary) gives a CONTROLLED, AUDITABLE supply chain — directly the
SAME "centralize and specially-protect" principle as the Database Architecture File 6 §2
key-management discussion, here applied to the artifact-distribution layer.
```

---

## 6. Environment Promotion Strategies

### 6.1 The Standard Promotion Flow

```
dev (auto-deployed on every merge to main) → staging (auto-deployed after CI passes,
  used for integration/QA validation) → production (PROMOTED explicitly — either a manual
  approval gate, or automated progressive delivery, §4, following successful staging validation)
```

### 6.2 Promoting a CONFIG, Not Rebuilding an ARTIFACT

```
Directly the System Design guide's general "build once, deploy everywhere" principle,
restated in GitOps terms: the SAME, already-built, already-scanned container image
(immutably tagged, §5.1) is promoted ACROSS environments by updating the CONFIG repository
(§2.3) to reference it in each successive environment — NEVER rebuilt per-environment.
Rebuilding per environment risks the EXACT "it worked in staging but not production"
class of bugs this principle exists to eliminate, since a rebuild could pull a subtly
different dependency version or base image between builds.
```

### 6.3 Database Migration Coordination Across Promotion Stages

```
Directly extending the Spring Data guide's §8.2 expand-contract discipline and the Java
Testing & Production Readiness guide's §6.4 migration timing concern to the PROMOTION
PIPELINE specifically: a migration must be validated against STAGING (with production-
representative data volume, Database Architecture File 5 §8.2's benchmarking discipline)
BEFORE the corresponding application version is promoted to production — and the
migration itself should typically run as its OWN pipeline step, GATING the application
deployment, rather than being silently bundled into application startup (Flyway's
default behavior, Spring Data guide §8) without an explicit, reviewable promotion gate.
```

---

## 7. Secrets in CI/CD Pipelines

### 7.1 Never Echo, Log, or Hardcode Secrets in Pipeline Definitions

```yaml
# CATASTROPHIC — a pipeline definition committed to source control with a literal secret
env:
  DB_PASSWORD: "hardcoded-prod-password" # visible to EVERYONE with repo read access, FOREVER,
                                            # in Git history, even if "fixed" in a later commit

# CORRECT — fetched at RUNTIME from a dedicated secrets manager, never persisted in the
# pipeline definition or logs (most CI platforms also AUTOMATICALLY redact known secret
# values from log output as a backstop, but this should never be the PRIMARY control)
env:
  DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }} # GitHub Actions' native secrets store,
                                                  # itself backed by encryption at rest
```

### 7.2 Scope CI Pipeline Credentials to Least Privilege, Per Environment

```
Directly the Database Architecture File 6 §3.2 least-privilege principle, applied to
DEPLOYMENT credentials: the CI pipeline's credential for deploying to STAGING should NOT
also have PRODUCTION deployment rights — a compromised CI pipeline configuration (a
malicious pull request injecting a build step, a compromised third-party GitHub Action)
should have its blast radius CONTAINED to the LEAST-privileged environment it actually needs,
not an automatic path to production.
```

---

## 8. Rollback Strategies

### 8.1 GitOps Rollback — Revert the Config Commit

```bash
git revert <commit-sha>   # reverts the config-repo commit that bumped the image tag —
git push                   # the GitOps controller (§3) AUTOMATICALLY reconciles the
                              # cluster back to the PREVIOUS, known-good image, with NO
                              # manual kubectl intervention and a CLEAN, auditable Git history
                              # entry recording exactly what was rolled back and when
```

### 8.2 The Database-Compatibility Constraint on Rollback

```
Directly the Java Testing & Production Readiness guide's §6.4 expand-contract discipline,
restated as a ROLLBACK-SPECIFIC concern: rolling back APPLICATION code is easy (§8.1) —
but if a migration that ran ALONGSIDE the now-being-rolled-back deployment already
modified the schema in a way the OLDER application version can't handle (e.g., a CONTRACT-
phase migration that dropped a column the old code still expects), a clean rollback isn't
actually possible without ALSO reverting the migration — precisely why expand-contract
discipline (always deploying the CONTRACT phase only after the OLD code is fully retired,
never bundled with the SAME deployment that introduces the new code) exists: it keeps
rollback genuinely possible at every point in a deployment's lifecycle, not just in theory.
```

### 8.3 Automated Rollback Triggers

```
Directly §4.2's automated-analysis mechanism IS an automated rollback trigger — extending
this principle, ANY deployment pipeline should have a DEFINED, ideally AUTOMATED, rollback
trigger (an error-rate threshold, a latency-SLA breach) rather than relying SOLELY on a
human noticing a dashboard and manually deciding to roll back — directly the same "every
alert needs a clear, actionable response" discipline from the Java Testing & Production
Readiness guide's §7.5, here wired all the way through to an AUTOMATED ACTION rather than
just a notification a human must act on.
```

---

## 9. A Worked End-to-End Pipeline

Synthesizing every section into one coherent flow for a Spring Boot microservice:

```
1. Developer merges a PR to main (application repository)
2. CI pipeline (§1): compile → static analysis → unit tests → integration tests
   (Testcontainers) → build image (buildpacks, File 1 §7.1) → vulnerability scan (§5.2)
   → push to private registry with an IMMUTABLE tag (Git SHA, §5.1)
3. CI pipeline updates the CONFIG repository's `environments/staging/` manifest with the
   new image tag (§2.3, §6.2) and opens an auto-merged PR (or commits directly, per team policy)
4. ArgoCD (§3.1) detects the config-repo change, reconciles staging automatically
5. Automated staging validation (integration/smoke tests against the live staging
   environment) — on success, a PROMOTION step updates `environments/production/`'s
   manifest, typically gated by a manual approval for production specifically (§6.1)
6. ArgoCD reconciles production — but via an Argo Rollouts canary strategy (§4.1), NOT an
   instant full cutover: 10% → automated analysis (§4.2) → 50% → analysis → 100%
7. If automated analysis detects a regression at ANY step, the rollout AUTOMATICALLY
   reverts to the previous version (§4.2, §8.3) — no human intervention required for the
   common case; a human is paged (Java Testing & Production Readiness guide §7.5) only
   for genuinely ambiguous or persistent failures the automation can't resolve on its own
8. Every step of this flow — every image built, every config change, every rollout
   progression — is recorded as an auditable Git history entry and/or ArgoCD application
   history, directly satisfying the deployment-audit-trail expectations any FDE engagement
   in a regulated industry (Database Architecture File 6 §8, AI/Agentic guide's File 6 §4)
   would need to demonstrate to a customer's own compliance function.
```
