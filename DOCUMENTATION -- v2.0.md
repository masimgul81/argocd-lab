📋 1. Project Overview
This platform is a production-grade Kubernetes environment built from scratch on AWS. It demonstrates the full "Zero to Hero" DevOps lifecycle, including Infrastructure as Code, GitOps delivery, automated security scanning, and traffic management.

🛠️ Technology Stack
Layer	Tool	Description
Infrastructure	AWS & Terraform	3-Node Cluster (Ubuntu 22.04, t3.medium)
Orchestrator	Kubernetes v1.30	Bootstrapped via Kubeadm
GitOps	ArgoCD	Pull-based Continuous Delivery (App of Apps pattern)
CI / Security	GitHub Actions + Trivy	Automated Build, Security Scan (CVEs), and Push
Networking	Nginx Ingress Controller	Host-based Routing (myflix.local)
Observability	Metrics Server	Resource Monitoring (CPU/RAM)
Elasticity	Horizontal Pod Autoscaler	Automated scaling based on CPU load
🏗️ 2. Cluster Architecture
Node Layout
Control Plane: k8s-master (API, Scheduler, Etcd)

Worker Node 1: k8s-node1 (Workloads, Ingress Entrypoint)

Worker Node 2: k8s-node2 (Workloads, Ingress Entrypoint)

Network Configuration
CNI Plugin: Calico (Pod-to-Pod communication)

Ingress Entrypoint: Port 31232 (Opened in AWS Security Group)

Service Discovery: CoreDNS

🔄 3. GitOps Workflow (ArgoCD)
Repository Strategy:

Source Repo (myflix-source): Contains application code (index.html) and Dockerfile.

Config Repo (argocd-lab): Contains Kubernetes Manifests (deployment.yaml, ingress.yaml).

The Sync Flow:

Developer pushes code to myflix-source.

CI Pipeline builds Docker image & updates deployment.yaml in argocd-lab.

ArgoCD detects the Drift (difference between Git and Cluster).

ArgoCD syncs the new image version to the cluster automatically.

🛡️ 4. DevSecOps & CI Pipeline
Pipeline Location: .github/workflows/ci.yaml Security Policy: Block deployment if CRITICAL or HIGH vulnerabilities are found.

Key Stages:

Checkout & Login: Authenticate with Docker Hub.

Build (Local): Build image locally for scanning.

Trivy Scan:

YAML
- name: Run Trivy Vulnerability Scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'g3niuz/myflix:${{ github.sha }}'
    exit-code: '1' # FAIL the build if vulnerabilities are found
    severity: 'CRITICAL,HIGH'
Push & Update: If scan passes, push to Registry and update GitOps repo.

🌐 5. Ingress Routing Map
Traffic is handled by Nginx Ingress Controller, eliminating the need for random NodePorts.

Domain	Target Service	Internal Port	Access URL
myflix.local	my-nginx-service	80	http://myflix.local:31232
bucks.local	my-bucks-service	80	http://bucks.local:31232
argocd.local	argocd-server	443	https://argocd.local:31232
Note: argocd.local requires SSL Passthrough annotations in the Ingress rule.

📈 6. Elasticity (HPA)
The cluster utilizes Horizontal Pod Autoscaling to handle traffic spikes.

Metric: CPU Utilization

Threshold: 50%

Scale Range: Min 1 replica / Max 10 replicas

Dependencies: Requires metrics-server and resources.requests defined in Deployment.

🔧 7. Operational Maintenance (Troubleshooting)
Critical: Since AWS t3.medium instances have small root volumes (8GB), the following maintenance is required to prevent "Disk Pressure" evictions.

Routine Cleanup Command
Run this on Worker Nodes if pods remain in Pending state:

Bash
# 1. Clean Docker/Containerd garbage
sudo crictl rmi --prune
# OR
sudo docker system prune -a --volumes -f

# 2. Vacuum System Logs
sudo journalctl --vacuum-time=1s

# 3. Restart Kubelet (to reset disk pressure checks)
sudo systemctl restart kubelet
Access Recovery
If argocd or ingress pods fail to start, check the Node Status:

Bash
kubectl describe node k8s-node1 | grep Pressure
# Should be: DiskPressure=False
🚀 8. Deployment Commands (Cheatsheet)
Bootstrap Ingress:

Bash
kubectl apply -f [https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml)
Bootstrap Metrics:

Bash
kubectl apply -f [https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml](https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml)
# Patch for self-signed certs:
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
Stress Test (Load Generator):

Bash
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://my-nginx-service; done"

Maintained by: Masim Gul (masimgul81)



