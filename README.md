# Q&A: Terraform and DevOps Best Practices on Google Cloud Platform (GCP)


## 1. What are some common Terraform management challenges in real-world setups?

| Challenge | Description | Solution |
|------------|--------------|-----------|
| **State Conflicts** | Multiple engineers updating infrastructure simultaneously. | Use remote backends with locking mechanisms such as **GCS** or **Terraform Cloud**. |
| **Drift** | Manual edits to infrastructure through the GCP Console. | Run regular `terraform plan` checks to detect discrepancies. |
| **Module Versions** | Breaking changes in shared modules impact production. | Apply tags and maintain version control for shared modules. |
| **Secrets** | Sensitive data exposed in Terraform state files or code. | Store secrets in **GCP Secret Manager** or **Vault**. |
| **Large Plans** | Terraform runs slow on big environments. | Split infrastructure by **layers (network, compute, GKE)** or by **environment (dev, prod)**. |

---

## 2. How can you organize a multi-environment Terraform setup on GCP?

You can structure Terraform configurations using module-based architecture and environment isolation.


**Recommended Folder Layout:**
```
terraform/
modules/
vpc/
gke/
compute/
envs/
dev/
main.tf
variables.tf
terraform.tfvars
prod/
main.tf
variables.tf
terraform.tfvars
```
 

**Best Practices:**
- Use **separate state files** for each environment.
- Store state remotely in **Google Cloud Storage (GCS)** for reliability and team collaboration.
- Keep environment-specific `.tfvars` for variables like `project_id`, `region`, and resource settings.
- Optionally, create workspaces for identical environments:
terraform workspace new dev
terraform apply -var-file=dev.tfvars

 

---

## 3. How is a multi-region GCE deployment designed with rollback automation?

**Architecture Overview:**
- Deploy **Managed Instance Groups (MIGs)** across multiple regions for redundancy.
- Use **Global HTTP(S) Load Balancer** to distribute traffic seamlessly.
- **Health checks** ensure traffic is redirected to healthy backends in case of failure.

**Rollback Mechanism:**
- Enable **rolling updates** with health verification in MIGs.
- Integrate **Cloud Build** or **Spinnaker** pipelines to detect deployment failures.
- Automatically restore to the previous instance template on rollback triggers.

---

## 4. What steps should you follow to troubleshoot an ImagePullBackOff error in Kubernetes?

**Step-by-Step Diagnosis:**
1. Describe the problematic Pod:
kubectl describe pod <pod-name>

 
2. Confirm the **image name** and **tag** are correct.
3. Verify **image pull secrets**:
kubectl get secrets

 
4. Test **registry access** from a local machine or node.
5. Check **node DNS** or connectivity issues.
6. Review **kubelet logs** for image pull or authentication errors.

**Summary:**  
Most `ImagePullBackOff` issues occur due to incorrect image references, permission errors, or network problems.

---

## 5. How can you optimize CI/CD image scanning without slowing down builds?

**Using GitHub Actions with Trivy:**

```
jobs:
build:
runs-on: ubuntu-latest
steps: ...
scan:
runs-on: ubuntu-latest
steps:
- name: Cache Trivy DB
uses: actions/cache@v4
with:
path: ~/.cache/trivy
key: ${{ runner.os }}-trivy-db
 
  - name: Scan Image
    run: trivy image myapp:latest
```

**Best Practices:**
- Run security scans **in parallel** with build jobs.
- Use **cache** to store vulnerability databases and avoid redundant downloads.
- Use **`paths` filters** to trigger scans only when container files change.
- Schedule **nightly deep scans** using GitHub **cron triggers**.

---

## 6. What is the ideal team structure for Terraform and DevOps projects?

**Typical Team Composition (3–5 members):**

| Role | Responsibility |
|------|----------------|
| **Infra Lead** | Maintain common modules, CI/CD processes, and architecture consistency. |
| **Terraform Engineers (1–2)** | Manage environments, state files, and module updates. |
| **Automation/Security Engineer** | Implement monitoring, image scanning, and security automation. |

**Why this works:**  
Small teams ensure clear ownership, reduce merge conflicts, and support faster deployments.

---

## 7. What are the best practices summarized across all areas?

| Area | Best Practice |
|------|----------------|
| **Infra Provisioning** | Modular Terraform structure with remote state storage. |
| **Deployment** | Blue-Green rollout strategy with automated rollback. |
| **Reliability** | Multi-region instance groups with health checks. |
| **Security** | Cached image scanning and conditional triggers. |
| **Team Efficiency** | Clear ownership and CI/CD automation. |

# DevOps / SRE Interview Notes

## 1. CI/CD pipeline & zero downtime

### End-to-end CI/CD pipeline

A simple end-to-end pipeline:

- Developer → Git push / PR  
- CI: build + unit tests + lint + security scan  
- Build Docker image  
- Push image to registry (ECR / Docker Hub)  
- Deploy to `dev`  
- Run integration tests  
- Promote same image to `stage`  
- Run E2E / smoke tests  
- Manual approval (optional)  
- Deploy to `prod` using Rolling / Blue-Green / Canary  
- Monitor + auto-rollback on failure  

Key principles:

- **Same artifact** (image digest) goes through all envs.  
- Promotion is driven by tests + approvals, not by rebuilding.  
- Rollback uses the last known good image tag/digest.

### Ensuring zero-downtime deployments

Zero downtime = users do not notice the deployment.

Common techniques:

- **Rolling updates (Kubernetes)**  
  - New pods start while old pods still serve traffic.  
  - Traffic only sent to pods that pass `readinessProbe`.  
  - `livenessProbe` restarts stuck pods.  
  - `maxUnavailable: 0` and multiple replicas so capacity is maintained.

- **Blue-Green deployment**  
  - Blue = current version; Green = new version.  
  - Deploy to Green, run health checks/smoke tests.  
  - Switch traffic to Green only when healthy.  
  - Rollback = switch back to Blue.

- **Canary deployment**  
  - Route a small percentage of traffic (e.g. 5–10%) to the new version.  
  - If metrics are good, gradually increase traffic.  
  - If errors increase, roll back quickly.

- **Safe database changes**  
  - Backward-compatible migrations.  
  - Add new columns first; deploy app that uses both; remove old fields later.  
  - Use feature flags for risky changes and schema transitions.

---

## 2. Multi-environment deployments in CI/CD

### Build once, deploy many

- Build a single Docker image and push to registry.  
- Use the same image for `dev → stage → prod`.  
- Only configs differ per environment (env vars, Helm values, Kustomize overlays).

### Environment separation

Common patterns:

- Separate **Kubernetes namespaces**: `dev`, `stage`, `prod`.  
- Separate **clusters / cloud accounts** for stronger isolation (e.g. separate AWS accounts for prod).  
- Env-specific configuration:
  - Helm: `values-dev.yaml`, `values-stage.yaml`, `values-prod.yaml`.  
  - Kustomize: `overlays/dev`, `overlays/prod`, etc.
```
Example Kustomize layout:
deploy/
base/
deployment.yaml
service.yaml
overlays/
dev/
kustomization.yaml
prod/
kustomization.yaml
```

### Promotion and controls

- Dev deploy: automatic on merge or PR.  
- Stage: after integration tests.  
- Prod: requires manual approval + strong checks (tests, security, change window).

---

## 3. Blue-Green deployment & automatic traffic shift

### Blue-Green flow

- Blue is live and receives production traffic.  
- Deploy the new version to Green.  
- Run health checks and smoke tests on Green.  
- If Green is healthy, switch traffic to Green.  
- Keep Blue around for quick rollback.

### Traffic switching approaches

1. **Kubernetes Service selector switch**

- One stable Service object.  
- Two Deployments: Blue and Green.  
- Example labels:
  - Blue pods: `version=blue`  
  - Green pods: `version=green`  
- Service initially selects `version=blue`.  
- When Green is ready, update Service selector to `version=green`.

2. **AWS ALB / target group switching**

- Blue and Green target groups behind one ALB.  
- Pipeline updates the listener rule to point at Green.  
- Switch only when:
  - Target group health checks pass.  
  - Optional smoke tests succeed.

### Automatic health-based switching

CD system checks:

- Kubernetes readiness probes.  
- ALB target group health.  
- Custom smoke-test endpoint `/health` or `/ready`.  

If all pass, pipeline updates selectors / listener rules.  
Tools that support this well:

- Argo Rollouts.  
- Flagger.  
- Spinnaker (classic CD).

Rollback: point traffic back to Blue.

---

## 4. Testing strategy in CI/CD

### Test layers

- **Fast checks (every PR)**  
  - Linting: eslint, flake8, golangci-lint.  
  - Unit tests.  
  - SAST security scan.  
  - Build/compile.

- **Medium (on merge / staging)**  
  - Integration tests (service + DB).  
  - Contract tests (for microservices).  
  - Container image scan.

- **Slow (before prod / nightly)**  
  - End-to-end UI/API tests.  
  - Performance / load tests.  
  - Optional chaos tests.

### Implementing automatic tests

Typical pipeline:

- PR:
  - Run unit tests, lint, SAST.  
  - Fail pipeline → block merge via branch protection.  
- Merge to main:
  - Build Docker image, run integration tests.  
- Stage:
  - Deploy, run E2E / smoke tests.  
- Prod:
  - Deploy, run post-deploy smoke tests, monitor for rollback.

### Minimizing test time

- **Parallelization**: shard test suites across multiple jobs.  
- **Caching**:
  - Dependencies: `node_modules`, Maven, pip cache, etc.  
  - Docker layers via BuildKit.  
- **Test selection**:
  - Run only impacted tests based on changed files.  
- **Test pyramid**:
  - Many unit tests (fast).  
  - Fewer slow E2E tests.  
- **Schedule heavy tests**:  
  - Full E2E/perf nightly.  
  - Light smoke tests on each deployment.

---

## 5. Git, branching & code quality

### GitFlow vs Trunk-Based Development

**GitFlow**:

- Branches:
  - `main` (production).  
  - `develop` (integration).  
  - `feature/*`, `release/*`, `hotfix/*`.  
- Pros:
  - Clear structure for planned releases.  
  - Good for scheduled / long release cycles.  
- Cons:
  - Long-lived branches → merge conflicts.  
  - Slow delivery.

**Trunk-Based Development**:

- Work off `main`/`trunk` with short-lived branches.  
- Small incremental merges multiple times per day.  
- Use feature flags to hide incomplete work.  
- Pros:
  - Faster releases, ideal for CI/CD.  
  - Smaller PRs, fewer conflicts.  
- Cons:
  - Requires strong automation and discipline.

**What product companies prefer**:

- Most fast-moving product companies (Big Tech etc.) favor Trunk-Based Development because it optimizes for speed, continuous delivery, and simpler branching. [web:5][web:11]

### Debugging with git bisect

Goal: find the first commit that introduced a bug.

Basic flow:
```
git bisect start
git bisect bad # current buggy commit
git bisect good <good_commit> # known good commit
```

## Git checks out a mid-point commit:
run tests manually, then:
```
git bisect bad # if bug is present
git bisect good # if bug is absent
```

## Repeat until Git prints the first bad commit
git bisect reset


### Enforcing code quality & approvals

Use branch protection and repository policies:

- Require PR reviews (e.g. 2 approvals).  
- Use Code Owners for sensitive paths.  
- Require passing CI checks (tests, lint, security scans).  
- Block direct pushes to `main`.  
- Optionally require signed commits.  
- Require branches to be up to date with `main` before merging.

Tools:

- Lint: eslint, pylint, golangci-lint.  
- Security: Snyk, Trivy, CodeQL.  
- Formatters: Prettier, Black, gofmt.  
- Coverage thresholds in CI.

### Restricting Git repo access

Principle: least privilege.

- Manage access via Teams/Groups, not individuals.  
- Role types: read-only, write, maintain, admin.  
- Make repos private if needed; remove old users regularly.  
- Strengthen auth:
  - Enforce SSO and MFA.  
  - Use audit logs.  
- Protect branches and tags:
  - Protected branches for `main`/`release`.  
  - Protected tags for releases.  
- For CI/CD:
  - Use short-lived tokens or OIDC.  
  - Avoid personal access tokens; use deploy keys only when required.

---

## 6. Terraform / IaC practices

### Multi-environment Terraform structure

Example layout:
```
terraform/
modules/
vpc/
eks/
rds/
app_service/
envs/
dev/
main.tf
backend.tf
dev.tfvars
stage/
main.tf
backend.tf
stage.tfvars
prod/
main.tf
backend.tf
prod.tfvars
```

Best practices:

- Put reusable logic in `modules/`.  
- One remote state per environment.  
- Use backends like S3 + DynamoDB for locking.  
- Separate accounts/projects for prod.

### Rollback in Terraform

Terraform does not have an automatic "rollback".

Typical strategy:

- Revert Terraform code to last known good commit.  
- Run `terraform plan` and `terraform apply`.  
- Use state versioning via backend (e.g. S3 object versions) as an extra safety net.  
- Use safe lifecycle flags:
  - `create_before_destroy = true` for replacements.  
  - `prevent_destroy = true` for critical resources.  
- For very risky changes, use blue-green infrastructure (deploy new stack, switch traffic).

### What happens in `terraform plan` and `apply`

`terraform plan`:

- Reads `.tf` files and current state.  
- Refreshes real resource data (unless disabled).  
- Builds a dependency graph.  
- Computes a diff of planned changes.

`terraform apply`:

- Acquires state lock.  
- Applies changes in dependency order.  
- Updates state after each successful change.  
- Releases lock at the end.

### Locking issues & partial failures

State lock issues:

- Occur when two operations conflict or CI dies mid-run.  
- Steps:
  - Confirm no other Terraform is running.  
  - Inspect lock owner (error details).  
  - If truly stale: `terraform force-unlock <LOCK_ID>` (carefully).

Partial apply failures:

- After a failed `apply`:
  - Run `terraform plan` and `terraform apply` again; it will continue from current state.  
- If state is inconsistent:
  - Use `terraform state list`/`state show`.  
  - Use `terraform import` to sync existing resources.

### Drift detection and fix

Drift = resources changed outside Terraform.

- Detect:
  - `terraform plan`  
  - or `terraform plan -refresh-only`  
- Fix:
  - If Terraform is source of truth → `terraform apply`.  
  - If manual change is desired → update .tf files or import.  
- Prevent:
  - Restrict console changes.  
  - Use IAM to limit manual edits.  
  - Run scheduled plans in CI for drift detection.

### Reusable Terraform modules for microservices

Good module characteristics:

- Small, focused, and composable.  
- Clear inputs/outputs; no hardcoded env-specific values.  
- Versioned with tags/releases.

Example responsibilities for a microservice module:

- Service IAM role.  
- Load balancer listener/rules.  
- Autoscaling configs.  
- Env vars / configuration.  
- Logs and alarms.

Example interface:

- Inputs: `service_name`, `cpu`, `memory`, `env_vars`, `tags`, etc.  
- Outputs: `service_url`, `security_group_id`.

---

## 7. Docker & container practices

### Reducing Docker image size & build time

Reduce image size:

- Use minimal base images (Alpine, distroless) when feasible.  
- Clean package caches and build artifacts.  
- Use `.dockerignore` to avoid copying unnecessary files.  
- Keep build tools out of the runtime image.

Improve build time:

- Leverage layer caching (copy dependency files first).  
- Use BuildKit cache in CI.  
- Avoid rebuilding unchanged dependencies.

### Multi-stage builds

Multi-stage builds:

- Stage 1: build/compile in a full builder image.  
- Stage 2: copy only final binaries/artifacts into a small runtime image.  

Use cases:

- Go/Java/Node build pipelines.  
- Need for small, secure production images.

### Docker healthchecks

Dockerfile example:

HEALTHCHECK --interval=30s --timeout=5s --retries=3
CMD curl -f http://localhost:8080/health || exit 1

text

Check health:

docker ps
docker inspect --format='{{json .State.Health}}' <container>

text

In Kubernetes, use readiness and liveness probes instead of Docker `HEALTHCHECK`.

### Troubleshooting container crashes (OOM/CPU/segfault)

Steps:

1. Check logs: `docker logs <container>`.  
2. Inspect container: `docker inspect <container>` to see exit code and reason.

Common issues:

- OOM (exit code 137, killed by kernel):  
  - Increase memory limit, tune memory usage, fix leaks.  
- CPU spike:  
  - Look for tight loops or heavy queries; optimize; add CPU limits; autoscale.  
- Segfaults:  
  - Often due to native library issues; pin versions; use stable base images.

---

## 8. Kubernetes production topics

### Production-grade Kubernetes architecture

Typical production setup:

- Cluster:
  - Multi-AZ worker nodes for high availability.  
  - Managed control plane (EKS/GKE/AKS).  
- Networking:
  - Ingress controller (NGINX / ALB / Traefik).  
  - NetworkPolicies for service-to-service access control.  
- Workloads:
  - Deployments for stateless apps.  
  - StatefulSets for stateful components if needed.  
  - Separate namespaces for `dev`, `stage`, `prod`.  
- Security:
  - RBAC with least privilege.  
  - Pod Security controls (restricted profiles).  
  - Encrypted secrets at rest; image scanning & signing.  
- Operations:
  - GitOps tools (Argo CD / Flux).  
  - Monitoring (Prometheus, Grafana).  
  - Logging (EFK / Loki).  
  - Tracing (Jaeger / Tempo).  
  - Backups (Velero).

### CrashLoopBackOff & ImagePullBackOff

CrashLoopBackOff (pod repeatedly crashes):

- Commands:
  - `kubectl describe pod <pod>`  
  - `kubectl logs <pod> --previous`  
- Causes:
  - Application errors, bad configs, missing env/secrets.  
  - Wrong command/entrypoint.  
  - Aggressive liveness probe.  
  - Insufficient CPU/memory.  
- Fix:
  - Correct configs, adjust probes, increase resources, fix app.

ImagePullBackOff:

- Command: `kubectl describe pod <pod>`.  
- Causes:
  - Wrong image name/tag, image not present.  
  - Registry auth issues (`imagePullSecrets`).  
  - Network/DNS issues.  
- Fix:
  - Fix image reference; set up credentials; verify IAM/permissions.

### Readiness vs liveness probes

- Readiness probe failure:
  - Pod marked NotReady; removed from Service endpoints.  
  - Pod not restarted.  
- Liveness probe failure:
  - Container restarted.  
  - Repeated failures → CrashLoopBackOff.  
- Best practice:
  - Use `startupProbe` for slow-start apps.  
  - Make liveness less strict than readiness.

### Autoscaling: HPA, VPA, Cluster Autoscaler

- HPA:
  - Scales pod replicas (horizontal) based on metrics (CPU/memory/custom).  
  - Requires Metrics Server and optionally Prometheus adapter.  
- VPA:
  - Adjusts CPU/memory requests/limits (vertical).  
  - Useful when apps do not scale well horizontally; be careful with restarts.  
- Cluster Autoscaler:
  - Scales nodes when pods cannot be scheduled due to lack of capacity.  

Common pattern: HPA scales pods; Cluster Autoscaler scales nodes.

### Managing secrets securely

Options:

- External secret managers (recommended):
  - Vault, AWS Secrets Manager, Parameter Store.  
  - Access via CSI driver or External Secrets Operator; supports rotation.  
- Sealed Secrets:
  - Store encrypted secrets in Git; controller decrypts them in cluster.  
  - Good for GitOps.  
- Native Kubernetes Secrets:
  - Base64 only; enable encryption at rest (KMS).  
  - Strong RBAC; avoid logging secrets.

---

## 9. AWS networking & reliability

### VPC design across AZs with NAT/IGW

Common VPC layout:

- AZ A:
  - Public subnet: ALB, NAT Gateway.  
  - Private subnet: app servers / EKS nodes.  
  - DB subnet: RDS (no internet).  
- AZ B:
  - Mirror of AZ A.

Routing:

- Internet Gateway (IGW):
  - Attached to VPC; public subnets route `0.0.0.0/0 → IGW`.  
- NAT Gateway:
  - Lives in public subnet; private subnets route `0.0.0.0/0 → NAT`.  
  - Allows outbound internet from private resources.

Best practices:

- One NAT Gateway per AZ to avoid cross-AZ dependency and extra data charges.  
- Use VPC endpoints (S3, DynamoDB, ECR) to reduce NAT usage and improve security.

---

## 10. Observability, SLIs & SLOs

### SLIs/SLOs definitions

- SLI (Service Level Indicator):
  - Measurable metrics like availability, latency, error rate, throughput.  
- SLO (Service Level Objective):
  - Target values for SLIs, e.g. "99.9% monthly availability", "p95 latency < 300 ms".

### Preventing production incidents

- Error budget:
  - Allowed level of unreliability derived from SLO.  
  - If budget burns too fast, pause risky releases and focus on reliability.  
- SLO-based alerting:
  - Alert on user-impacting symptoms (high error rates, high latency) rather than raw resource metrics.  
- Other practices:
  - Canary / Blue-Green deployments.  
  - Auto rollback.  
  - Runbooks for common incidents.  
  - Postmortems to learn from outages.  
  - Load testing before big launches and capacity planning with autoscaling.
Related

Convert this CI CD pipeline to README.md with sections and code blocks

Create a concise YAML GitHub Actions workflow from the pipeline steps

Generate Helm values and kustomize overlays for dev stage prod

Write example Argo Rollouts config for canary and blue green

Add Kubernetes readiness liveness and startup probe examples for services



_This Q&A guide provides practical patterns for managing scalable, secure, and automated GCP infrastructure using Terraform and modern DevOps pipelines._
