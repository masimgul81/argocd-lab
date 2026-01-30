# 🚀 End-to-End Kubernetes GitOps Platform
**Project Owner:** Masim Gul
**Infrastructure:** AWS EC2 (Bare Metal K8s)
**Architecture:** Microservices with GitOps & DevSecOps

## 📋 Project Overview
This project implements a production-grade Kubernetes platform from scratch. It demonstrates the full DevOps lifecycle: provisioning infrastructure, bootstrapping a cluster, automating deployments via GitOps, securing the CI pipeline, and managing traffic via Ingress.

### 🛠️ Technology Stack
| Layer | Tool | Description |
| :--- | :--- | :--- |
| **Infrastructure** | AWS & Terraform | 3-Node Cluster (t3.medium) |
| **OS** | Ubuntu 22.04 LTS | Host Operating System |
| **Orchestrator** | Kubernetes v1.30 | Managed via Kubeadm |
| **GitOps** | ArgoCD | Pull-based Continuous Delivery |
| **CI / DevSecOps** | GitHub Actions + Trivy | Build, Scan, and Push Docker Images |
| **Container Registry**| Docker Hub | `g3niuz` |
| **Networking** | Nginx Ingress Controller | Host-based Routing (`myflix.local`) |
| **Observability** | Kubernetes Metrics Server | Resource Monitoring (CPU/RAM) |

---

## 🏗️ Phase 1: Cluster Architecture

The cluster consists of 3 nodes hosted on AWS.

* **Control Plane:** `k8s-master` (API Server, Scheduler, Controller Manager)
* **Worker Node 1:** `k8s-node1` (Workloads, Ingress)
* **Worker Node 2:** `k8s-node2` (Workloads, Ingress)

**Network Configuration:**
* **CNI Plugin:** Calico (Pod Networking)
* **Ingress Port:** `31232` (NodePort exposed to the internet)

---

## 🔄 Phase 2: GitOps Workflow (ArgoCD)

We use the **"App of Apps"** pattern where ArgoCD monitors a configuration repository and automatically syncs changes to the cluster.

### Repositories
1.  **Source Code Repo (`myflix-source`):** Contains `index.html`, `Dockerfile`, and CI Pipeline.
2.  **Config Repo (`argocd-lab`):** Contains Kubernetes YAMLs (`deployment.yaml`, `service.yaml`, `ingress.yaml`).

### Deployment Strategy
1.  Developer pushes code to `myflix-source`.
2.  **CI Pipeline** builds the Docker image and scans it for vulnerabilities.
3.  If secure, the image is pushed to Docker Hub (`g3niuz/myflix:<git-sha>`).
4.  The CI pipeline automatically updates the `deployment.yaml` in `argocd-lab` with the new image tag.
5.  **ArgoCD** detects the change in `argocd-lab` and updates the live cluster.

---

## 🛡️ Phase 3: DevSecOps Pipeline

**Location:** `.github/workflows/ci.yaml`
**Security Gate:** AquaSecurity Trivy

The pipeline enforces a "Quality Gate". If the Docker image contains **CRITICAL** or **HIGH** vulnerabilities, the build fails, and the deployment is blocked.

```yaml
    - name: Run Trivy Vulnerability Scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'g3niuz/myflix:${{ github.sha }}'
        exit-code: '1'        # Fail pipeline on error
        severity: 'CRITICAL,HIGH'