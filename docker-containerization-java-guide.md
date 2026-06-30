# Docker & Containerization for Java Workloads — Principal Engineer Reference

> File 1 of 7 in the Cloud-Native & DevOps series, a companion to the Java, Spring, AI/Agentic, and Database Architecture series. Covers containerizing JVM applications correctly — a surprisingly easy thing to get subtly wrong given the JVM's own resource-management assumptions predate widespread container adoption.

**Series:** **1. Docker & Containerization (this file)** · 2. Kubernetes for Java · 3. CI/CD & GitOps · 4. Cloud Provider Deep Dive · 5. Infrastructure as Code · 6. Observability & SRE · 7. Cost Optimization & FinOps

---

## Table of Contents

1. [Why Containerizing a JVM App Isn't Just "Add a Dockerfile"](#1-why-containerizing-a-jvm-app-isnt-just-add-a-dockerfile)
2. [Multi-Stage Builds for Java](#2-multi-stage-builds-for-java)
3. [JVM Container-Awareness & Memory Sizing](#3-jvm-container-awareness--memory-sizing)
4. [Image Layering & Build Caching Strategy](#4-image-layering--build-caching-strategy)
5. [Distroless & Minimal Base Images](#5-distroless--minimal-base-images)
6. [Health Checks & Graceful Shutdown in Containers](#6-health-checks--graceful-shutdown-in-containers)
7. [Spring Boot's Native Container Support](#7-spring-boots-native-container-support)
8. [Security Hardening for Java Container Images](#8-security-hardening-for-java-container-images)

---

## 1. Why Containerizing a JVM App Isn't Just "Add a Dockerfile"

### 1.1 The Historical Mismatch

```
The JVM predates widespread container adoption by roughly two decades — its DEFAULT
resource-detection logic historically read the HOST machine's total memory/CPU count, not
the CONTAINER's cgroup-imposed limits. Running a JVM in a container with a 512MB memory
limit, on a host with 64GB of RAM, could result in the JVM sizing its default heap as if
64GB were available — directly setting up the EXACT OutOfMemoryError-then-OOMKilled cycle
covered in the Java JVM Internals guide's §7.2 container-awareness flag discussion, here
explained as the CONTAINERIZATION-specific root cause rather than just a JVM flag to set.
```

### 1.2 The Good News — Modern JDKs Fixed This, With a Caveat

```
JDK 10+ enabled container-awareness by default (the JVM Internals guide's §7.2
-XX:+UseContainerSupport flag is now ON by default) — the JVM correctly reads cgroup
limits for memory AND CPU count on modern JDKs. The CAVEAT, still relevant: this only
helps if the CONTAINER ITSELF has an explicit memory limit set — a container run WITHOUT
a memory limit (or with one set far higher than the actual available host memory) gives
the JVM nothing correct to detect, reproducing the original problem despite running on a
modern JDK. ALWAYS set explicit container memory limits — this is not optional infrastructure
hygiene, it's a direct prerequisite for the JVM's own sizing logic to function correctly.
```

---

## 2. Multi-Stage Builds for Java

### 2.1 The Problem Multi-Stage Builds Solve

```dockerfile
# NAIVE single-stage Dockerfile — bakes the ENTIRE build toolchain (Maven/Gradle, the
# full JDK, source code, build caches) into the FINAL runtime image — bloating image
# size (slower pulls, slower cold starts in elastic/scale-to-zero environments, directly
# the Spring Trending Libraries guide's §5.3 startup-latency concern) and INCREASING THE
# ATTACK SURFACE (a build toolchain has no business existing in a production runtime image)
FROM maven:3.9-eclipse-temurin-21
WORKDIR /app
COPY . .
RUN mvn clean package
CMD ["java", "-jar", "target/app.jar"]
```

```dockerfile
# MULTI-STAGE — the build toolchain exists ONLY in an intermediate, DISCARDED stage;
# the final image contains ONLY the compiled artifact + a minimal JRE (not even a full JDK)
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline                # cache dependency resolution as its OWN layer —
COPY src ./src                                  # see §4's layer-ordering discipline
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine AS runtime  # JRE only — no compiler, no build tools
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 2.2 Why This Matters Beyond Just Image Size

```
The SECURITY dimension matters as much as the size dimension: a build toolchain inside a
production image is an UNNECESSARY attack surface (Database Architecture File 6 §7's
"every additional component is additional exposed surface" principle, applied to container
images specifically) — a vulnerability in Maven, Gradle, or any build-time dependency
becomes IRRELEVANT to runtime security once it's excluded from the shipped image entirely,
rather than merely "not actively used" while still being present and scannable/exploitable.
```

---

## 3. JVM Container-Awareness & Memory Sizing

### 3.1 Setting Explicit, Deliberate JVM Memory Flags — Don't Rely on Defaults Alone

```dockerfile
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-XX:+UseContainerSupport", \
  "-jar", "app.jar"]
```

```
Directly extending the Java JVM Internals guide's §3.6 heap-sizing discussion to the
CONTAINER-PERCENTAGE-BASED sizing model: rather than a fixed -Xmx value (which becomes
stale the moment the container's resource LIMIT changes across environments — dev, staging,
production, each potentially differently sized), -XX:MaxRAMPercentage expresses heap size
as a PERCENTAGE of whatever memory limit the container ACTUALLY has, automatically scaling
correctly across environments without per-environment flag tuning. The remaining 25%
(in the example above) is reserved for METASPACE, THREAD STACKS, DIRECT BUFFERS, and JVM
overhead generally — exactly the non-heap memory categories from the JVM Internals guide's
§3.1 memory map, which a heap-only sizing calculation would otherwise dangerously ignore.
```

### 3.2 The Concrete Container OOM Diagnosis Flow

```
A container repeatedly OOMKilled (visible via `kubectl describe pod` showing
"OOMKilled" as the last termination reason, or the Docker equivalent) is NOT necessarily
a Java heap problem — directly the JVM Internals guide's §9.3 "native memory leak shows as
container OOM, not a normal heap OOM" pattern: check NATIVE memory usage (direct ByteBuffers,
JNI allocations, thread-stack count × per-thread stack size) via -XX:NativeMemoryTracking,
not just heap dumps, before assuming the fix is simply "increase the container's memory limit" —
which often just delays the same underlying leak's eventual recurrence rather than fixing it.
```

### 3.3 CPU Limits and the JVM's Perceived Core Count

```
Container CPU limits (e.g., Kubernetes `resources.limits.cpu: "2"`) directly affect how
many cores Runtime.availableProcessors() reports to the JVM — which in turn affects
default thread-pool sizing for libraries that size pools based on this value (directly
the Java Concurrency guide's §2.6 CPU-core-based sizing formula). A JVM "thinking" it has
8 cores (the HOST's count) while actually CGROUP-LIMITED to 2 will OVER-PROVISION
CPU-bound thread pools, causing excessive context-switching contention within an artificially
small CPU allocation — confirm `Runtime.availableProcessors()` reports the CONTAINER's
actual limit, not the host's, on your specific JDK/container-runtime combination.
```

---

## 4. Image Layering & Build Caching Strategy

### 4.1 Layer Ordering — Least-Frequently-Changed First

```dockerfile
# CORRECT ordering — each layer changes LESS frequently than the one below it,
# maximizing Docker's build-cache reuse across successive builds
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY build.gradle settings.gradle ./        # changes RARELY — cached almost always
RUN ./gradlew dependencies --no-daemon        # dependency resolution — changes rarely
COPY src ./src                                  # changes on EVERY commit — least cacheable,
RUN ./gradlew build --no-daemon                   # placed LAST so it doesn't invalidate
                                                      # the (expensive) dependency-resolution
                                                      # layer above it on every single build
```

```
Directly the SAME caching-effectiveness principle as a CDN/cache hierarchy (Java System
Design guide §2.6) — placing frequently-changing source code BEFORE the dependency-
resolution step would invalidate Docker's cache for EVERY layer below it on every single
commit, forcing a full dependency re-download on every build — a genuinely common,
avoidable CI-pipeline slowdown (Java Testing & Production Readiness guide §6.1's "order
stages from cheapest to most expensive" principle, here applied to DOCKERFILE layer order
specifically rather than pipeline STAGE order).
```

### 4.2 Dependency Layer Caching for Maven/Gradle Specifically

```dockerfile
# Gradle's dependency cache directory, mounted as a BuildKit cache mount — persists
# ACROSS builds (not just within one Dockerfile's layer cache), even when source changes
# RUN --mount=type=cache,target=/root/.gradle/caches \
RUN --mount=type=cache,target=/root/.gradle/caches ./gradlew build --no-daemon
```

This is a genuinely high-leverage CI-pipeline speed optimization — without it, a CI runner spinning up a fresh container for every build re-downloads the ENTIRE dependency tree every single time, even with the layer-ordering discipline from §4.1, since Docker's LAYER cache doesn't persist across genuinely fresh build environments the way a BuildKit cache MOUNT does.

---

## 5. Distroless & Minimal Base Images

### 5.1 The Spectrum of Base Image Choices

```
FULL OS BASE (ubuntu, debian) + JRE installed — largest, most attack surface, but easiest
  to debug interactively (a full shell, package manager, standard Unix tools available).
ALPINE-BASED JRE IMAGES — smaller, musl-libc-based — historically had SOME compatibility
  edge cases with specific native-library-dependent Java libraries (verify compatibility
  for YOUR specific dependency set, particularly anything using JNI, before committing to
  Alpine as a default).
DISTROLESS (Google's distroless/java images) — contains ONLY the JRE and its direct runtime
  dependencies — NO shell, NO package manager, NO general-purpose OS tooling at all —
  the SMALLEST attack surface, at the cost of being meaningfully HARDER to debug
  interactively (you generally can't `docker exec` into a shell that doesn't exist).
```

### 5.2 The Trade-off, Stated Explicitly

```dockerfile
FROM gcr.io/distroless/java21-debian12
COPY app.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

```
Directly the Database Architecture File 6 §1.1 "every additional component is additional
exposed surface" principle, applied to OS-level container tooling: a shell, a package
manager, and general Unix utilities are EXACTLY the tools an attacker would want after
achieving SOME initial compromise — distroless images remove them ENTIRELY, meaningfully
raising the bar for any post-compromise lateral movement, at the real cost of needing a
SEPARATE debugging strategy (ephemeral debug containers attached via `kubectl debug`,
or relying more heavily on the observability tooling from File 6 of this series, rather
than ad hoc interactive shell access) for production troubleshooting.
```

---

## 6. Health Checks & Graceful Shutdown in Containers

### 6.1 Container-Level Health Checks vs Application-Level Probes

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD curl -f http://localhost:8080/actuator/health/liveness || exit 1
```

```
This Docker-native HEALTHCHECK directive is USEFUL for plain Docker/Compose deployments,
but for Kubernetes deployments specifically, the orchestrator's OWN liveness/readiness
probes (covered in full in File 2 of this series, directly building on the Spring Testing
& Observability guide's §6.3 liveness-vs-readiness distinction) supersede it — don't rely
on BOTH mechanisms disagreeing about health silently; pick the orchestration layer your
deployment target actually uses as the SOURCE OF TRUTH for health determination.
```

### 6.2 Graceful Shutdown — Respecting SIGTERM

```java
// Directly the Java Concurrency guide's §2.7 graceful-shutdown sequence, now triggered
// by the CONTAINER RUNTIME's SIGTERM signal specifically (sent on `docker stop` or a
// Kubernetes pod termination) rather than a JVM shutdown hook triggered some other way
@Component
public class GracefulShutdownHandler {
    @PreDestroy // Spring Core guide §2.1 — invoked as part of the Spring context's own
    public void onShutdown() {  // shutdown sequence, itself triggered by the JVM's shutdown
        log.info("Received shutdown signal, draining in-flight requests...");  // hook reacting to SIGTERM
        // Spring Boot's embedded Tomcat/Netty server already handles connection draining
        // by default — this hook is for ADDITIONAL cleanup: closing Kafka consumers
        // cleanly (committing final offsets), flushing any buffered metrics, etc.
    }
}
```

> ⚠️ **A container that doesn't handle SIGTERM gracefully gets forcibly SIGKILLed after a grace period** (Kubernetes defaults to 30 seconds, configurable) — any in-flight request still processing at that moment is abruptly terminated, a genuinely common source of intermittent, hard-to-reproduce "occasional failed request during deployment" issues that graceful-shutdown handling directly prevents.

---

## 7. Spring Boot's Native Container Support

### 7.1 Buildpacks — Skip Hand-Writing a Dockerfile Entirely

```bash
./gradlew bootBuildImage --imageName=myapp:latest
# Spring Boot's Cloud Native Buildpacks integration produces a PRODUCTION-GRADE,
# multi-layered, security-conscious container image WITHOUT a hand-written Dockerfile at
# all — automatically applying many of §2-5's best practices (layer separation, a
# minimal base image, non-root user) by default, maintained and updated by the buildpacks
# project rather than requiring every team to independently rediscover them
```

### 7.2 When to Use Buildpacks vs a Hand-Written Dockerfile

```
Favors buildpacks: teams wanting Spring Boot's "convention over configuration" philosophy
  (Spring Core guide §3.2) extended to CONTAINERIZATION itself — sensible, regularly-
  updated defaults with minimal hand-maintained Dockerfile logic.
Favors a hand-written Dockerfile: genuinely custom requirements (a non-standard base
  image for organizational/compliance reasons, additional OS-level dependencies a Java
  buildpack doesn't anticipate, fine-grained control over every layer for a specific
  optimization) — the SAME "convention is great until you have a specific reason to
  deviate" framing applied throughout the Spring series to auto-configuration generally.
```

### 7.3 Layered JARs — Buildpacks' and Hand-Written Dockerfiles' Shared Foundation

```bash
java -Djarmode=layertools -jar app.jar extract
# Spring Boot's layered-JAR support splits a fat JAR into separate layers (dependencies,
# Spring Boot loader, application classes/resources) by DEPENDENCY-CHANGE FREQUENCY —
# directly the SAME §4.1 "least-frequently-changed first" layering principle, but
# automatically derived from the build itself rather than manually ordered in a Dockerfile
```

---

## 8. Security Hardening for Java Container Images

### 8.1 Run as a Non-Root User — Always

```dockerfile
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup app.jar app.jar
USER appuser
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```
Directly the Database Architecture File 6 §3.2 least-privilege principle, applied to the
container's OWN process identity: a container running as root, if compromised via an
application-level vulnerability, gives an attacker root-equivalent access WITHIN the
container (and, depending on container-runtime configuration, potentially a meaningfully
easier path toward HOST-level privilege escalation) — running as a dedicated, unprivileged
user contains the blast radius significantly, at near-zero implementation cost.
```

### 8.2 Scan Images for Known Vulnerabilities in CI

```yaml
# A CI pipeline step (Trivy, Grype, or a cloud provider's native scanner) — directly
# extending the Java Testing & Production Readiness guide's §6.1 "fail fast, cheapest
# checks first" pipeline-ordering principle to SECURITY scanning specifically
- name: Scan image for vulnerabilities
  run: trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:${{ github.sha }}
```

### 8.3 Pin Base Image Versions — Never Float on `:latest`

```dockerfile
# AVOID — :latest is a MOVING TARGET; a rebuild six months from now could silently pull
# a DIFFERENT, untested base image version, breaking compatibility or introducing a
# regression with no corresponding code change to explain it
FROM eclipse-temurin:latest

# PREFER — pinned to a specific, known-good version (ideally by DIGEST, not just tag,
# for full reproducibility — a tag CAN be reassigned to a different image; a digest cannot)
FROM eclipse-temurin:21.0.4_7-jre-alpine@sha256:abc123...
```

### 8.4 Never Bake Secrets Into an Image Layer

```dockerfile
# CATASTROPHIC — even if a LATER layer "removes" the secret file, it remains permanently
# recoverable from the EARLIER layer's content in the image's layer history
COPY .env /app/.env       # AVOID — secrets baked directly into a layer
RUN rm /app/.env            # does NOT actually remove it from the image's history!
```

```
Directly extending the Database Architecture File 6 §2.3 secrets-management discipline
to container images specifically: secrets must be injected at RUNTIME (environment
variables from a secrets manager, Kubernetes Secrets mounted as files — covered fully in
File 2 of this series) — NEVER baked into any image layer, since Docker layers are
immutable and a "later removal" doesn't erase the secret from the image's actual stored history.
```

---

## Cross-Cutting Best Practices Checklist

- [ ] **Multi-stage builds** separating build toolchain from runtime image, always
- [ ] **Explicit container memory limits set**, with JVM percentage-based heap flags (`-XX:MaxRAMPercentage`) matching them
- [ ] **Layer ordering** placing least-frequently-changed content first
- [ ] **Non-root user** for the running process
- [ ] **Pinned, digest-referenced base images** — never `:latest`
- [ ] **Vulnerability scanning** as a blocking CI step
- [ ] **No secrets baked into any image layer** — runtime injection only
- [ ] **Graceful SIGTERM handling** with explicit, tested shutdown behavior
- [ ] **Distroless/minimal base images** evaluated explicitly, with a deliberate debugging-strategy trade-off accepted, not defaulted into without consideration
