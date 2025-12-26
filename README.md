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

---

_This Q&A guide provides practical patterns for managing scalable, secure, and automated GCP infrastructure using Terraform and modern DevOps pipelines._
