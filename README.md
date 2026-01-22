A Project of GitOps (based on Github + ArgoCD + Kubernetes Cluster at AWS)
Introduction of GitOps:

GitOps is a set of practices that uses a Git repository as the "single source of truth" for your infrastructure and application deployment.

Given your background with DevOps, Kubernetes, and Terraform, you can think of GitOps as the evolution of Infrastructure as Code (IaC). It takes the principles you already use for code (version control, collaboration, compliance) and applies them to the operational state of your Kubernetes clusters.


The Core Concept: "Push" vs. "Pull"
This is the biggest shift from the traditional CI/CD pipelines (like Jenkins) you are used to.

Traditional (Push): Your Jenkins pipeline runs a script that connects to your cluster and runs kubectl apply -f deployment.yaml. This requires giving Jenkins "admin" access to your cluster (a security risk).

GitOps (Pull): You install a "GitOps Controller" (agent) inside your Kubernetes cluster. This agent watches your GitHub repository. When you commit a change to the manifest in Git, the agent detects the divergence and "pulls" the change into the cluster to make the reality match the code.


Key Principles
Declarative: You describe the entire desired state of your system (apps, configs, dashboards) in files (YAML, Helm charts) stored in Git.

Versioned: Since everything is in Git, you have a complete history. If a deployment breaks, you don't run a complex rollback script; you simply git revert the commit, and the cluster syncs back to the previous stable state.

Continuous Reconciliation: The GitOps agent doesn't just deploy once. It constantly compares the Live State (what's running in K8s) with the Desired State (what's in Git). If someone manually deletes a pod or changes a configuration via kubectl (drift), the agent detects it and fixes it automatically.


Why You Should Care (Based on your stack)
No more kubectl apply from local: You previously dealt with kubectl authentication issues from your Ubuntu machine. With GitOps, you don't apply changes manually. You push to Git; the cluster updates itself.

Security: You don't need to store cluster credentials in your CI tool (Jenkins/GitHub Actions). The cluster pulls from Git; it doesn't accept external push commands.

Disaster Recovery: If your cluster crashes, you can point a new empty cluster at your Git repo, and the GitOps agent will reinstall everything exactly as it was.

Popular Tools
Since you are using Kubernetes, these are the industry standards:

ArgoCD: The most popular choice. It has a great UI that visualizes your application topology.

Flux: A more lightweight, "headless" alternative often used by platform teams.


Feature	Traditional DevOps (likely your current flow)	GitOps
Trigger	CI Pipeline pushes changes	Cluster agent pulls changes
Security	CI needs Cluster Admin access	Cluster needs Read-Only Git access
Drift	Unknown until next deployment	Automatically detected & corrected
Rollback	Complex manual process or scripts	git revert

Full GitOps Lab guide.
Prerequisites- Phases
1.	Infrastructure (Terraform): To create the 3 Virtual Machines on Azure.
2.	Bootstrapping (Kubeadm): To install Kubernetes and join the nodes together.
3.	CLI: kubectl configured to talk to K8s Cluster.
4.	Git: Your GitHub account (masimgul81).
________________________________________
Phase 1: Infrastructure as Code (Terraform)
This Terraform code creates 3 Ubuntu servers (k8s-Control-Plane, k8s-Node1, k8s-node2) in AWS. It also handles the networking (VPC, Subnets) and Security Groups (opening port 6443 for K8s API and 22 for SSH).
File: main.tf
# Define the required Terraform version and AWS provider
terraform {
  required_providers {
    aws = {
        source = "hashicorp/aws"
        version = ">= 5.0.0"
    }
  }
}

# Define the AWS provider configuration
provider "aws" {
  region = var.aws_region
  access_key = "Bring your own"
  secret_key = "Bring your own"
}

# Define Key pair resource
resource "aws_key_pair" "k8s_key_pair" {
  key_name   = "id_ed25519_k8s"
  public_key = file("id_ed25519.pub")
}

# Define Security Group resource
resource "aws_security_group" "k8s_security_group" {
  name        = "k8s_security_group"
  description = "Security group for Kubernetes cluster"
    vpc_id      = aws_vpc.k8s_vpc.id
    ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port   = 6443
        to_port     = 6443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port   = 10250
        to_port     = 10250
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port   = 30000
        to_port     = 32767
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
}   

# Define VPC resource
resource "aws_vpc" "k8s_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "k8s_vpc"
  }
}   

# Define Subnet resource
resource "aws_subnet" "k8s_subnet" {
  vpc_id            = aws_vpc.k8s_vpc.id
  cidr_block        = "10.0.1.0/24"
    availability_zone = "${var.aws_region}a"
    map_public_ip_on_launch = true
    tags = {
    Name = "k8s_subnet"
  }
}   

# Define Internet Gateway resource
resource "aws_internet_gateway" "k8s_igw" {
  vpc_id = aws_vpc.k8s_vpc.id
  tags = {  
    Name = "k8s_igw"
  }
}

# Define Route Table resource
resource "aws_route_table" "k8s_route_table" {
  vpc_id = aws_vpc.k8s_vpc.id
    route { 
        cidr_block = "0.0.0.0/0"    
        gateway_id = aws_internet_gateway.k8s_igw.id
    }
}   

# Associate Route Table with Subnet
resource "aws_route_table_association" "k8s_route_table_association" {
  subnet_id      = aws_subnet.k8s_subnet.id
  route_table_id = aws_route_table.k8s_route_table.id
}

# Define EC2 Instance resource for Master Node
resource "aws_instance" "k8s_master" {
    ami                         = var.ami_id
    instance_type               = var.instance_type
    key_name                    = aws_key_pair.k8s_key_pair.key_name
    subnet_id                   = aws_subnet.k8s_subnet.id
    vpc_security_group_ids      = [aws_security_group.k8s_security_group.id]
    associate_public_ip_address = true
    tags = {
        Name = "k8s_master"
    }
# private key to access the instance
    connection {
     type = "ssh"
        user = "ubuntu"
        private_key = file("id_ed25519")
        host = self.public_ip
        timeout = "2m"

    }
}

# Define EC2 Instance resource for Worker Nodes

resource "aws_instance" "k8s_worker" {
    count                       = var.worker_count
    ami                         = var.ami_id
    instance_type               = var.instance_type
    key_name                    = aws_key_pair.k8s_key_pair.key_name
    subnet_id                   = aws_subnet.k8s_subnet.id
    vpc_security_group_ids      = [aws_security_group.k8s_security_group.id]
    associate_public_ip_address = true
    tags = {
        Name = "k8s_worker_${count.index + 1}"
    }
# private key to access the instance
    connection {
     type = "ssh"
        user = "ubuntu"
        private_key = file("id_ed25519")
        host = self.public_ip
        timeout = "2m"
    }
}   

# Output the public IPs of the instances
output "master_public_ip" {
    value = aws_instance.k8s_master.public_ip
}           

output "worker_public_ips" {
    value = aws_instance.k8s_worker.*.public_ip
}   

File: variables.tf
# Define variables for AWS configuration
variable "aws_region" {
  description = "The AWS region to deploy resources in"
  type        = string
  default     = "us-east-1"
}

# Define variable for instance type
variable "instance_type" {
  description = "The type of instance to use for the Kubernetes nodes"
  type        = string
  default     = "m7i-flex.large"
}

# Define variable for number of worker nodes
variable "worker_count" {
  description = "The number of worker nodes in the Kubernetes cluster"
  type        = number
  default     = 2
}

# Define AMI ID variable
variable "ami_id" {
  description = "The AMI ID to use for the Kubernetes nodes"
  type        = string 
    default     = "ami-0360c520857e3138f" 
}



Deploy Logic:
1.	Run terraform init, terraform plan, terraform apply.
2.	Note the 3 Public IPs outputted at the end.


Phase 2: Bootstrapping Kubernetes (Kubeadm)
It must install the components manually. SSH into all 3 servers and perform the steps below.
Step 2A: On Control Plane
Run these commands to set up the environment.
K8s_master.sh
#!/bin/bash
# Kubernetes Master Setup
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y containerd.io

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd


# Install kubeadm, kubelet, kubectl

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Initialize cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Setup kubectl for ubuntu user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel network
#su - ubuntu -c "kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Allow scheduling on master
su - ubuntu -c "kubectl taint nodes --all node-role.kubernetes.io/control-plane-"



Step 2B: On K8s Nodes(Node1 + Node2) ONLY
K8s_node.sh
#!/bin/bash
# Kubernetes Master Setup
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y containerd.io

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd


# Install kubeadm, kubelet, kubectl

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Join the cluster
# Note: Replace <master-ip> and <token> with actual values
#kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
# Example:
# kubeadm join  


Phase 3: Verify & Deploy
Back on the Control Plane machine:
1.	Check Nodes:
Bash
kubectl get nodes
You should see all 3 nodes. Status might take a minute to change from NotReady to Ready while Calico starts.
2.	Install ArgoCD (Standard GitOps Setup):
Bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
3.	Access ArgoCD (Expose via NodePort for simplicity): Since you don't have localhost access to Azure easily, change the ArgoCD server to a NodePort service so you can access it via the Public IP.
Bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
Check the port:
Bash
kubectl get svc -n argocd
Look for port mapping (e.g., 443:31234/TCP). You can now access ArgoCD at https://<Control-Plane-Public-IP>:31234.
Wait 1-2 minutes for the pods to start. You can check status with: kubectl get pods -n argocd
Keep this terminal open. Now open your browser to: https://localhost:8080 (Click "Advanced" -> "Proceed" if you see a security warning).
1.	Get the Initial Password: Open a new terminal window and run this command to decode the default password:
Bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
o	Username: admin
o	Password: (The output from the command above)
Some Troubleshooting and tweaks
When you create a NodePort in Kubernetes, it opens port 30715 on the Linux OS of the node, but it does not automatically open that port on the AWS Firewall protecting the EC2 instance.
Here is how to fix it and access your ArgoCD UI.
Step 1: Understand the Ports
Look at your output: 443:30715/TCP
•	443: The internal port ArgoCD listens on (HTTPS).
•	30715: The external port opened on your AWS Nodes.
•	Target: You need to hit Port 30715 to reach the secure UI.
Step 2: Open the Port in AWS (The Fix)
You need to manually allow traffic on port 30715.
1.	Log in to your AWS Console.
2.	Go to EC2 Dashboard > Instances.
3.	Click on your k8s-Node1 (or any worker node) instance.
4.	In the details pane below, click the Security tab.
5.	Click the Security Group link (e.g., sg-012345...).
6.	Click Edit inbound rules.
7.	Click Add rule:
o	Type: Custom TCP
o	Port range: 30715
o	Source: 0.0.0.0/0 (Anywhere)
8.	Click Save rules.
Step 3: Access with the Correct URL
You must use the Public IP of the node and the HTTPS protocol.
1.	Get the Public IP of your node:
Bash
kubectl get nodes -o wide
# OR look at the "Public IPv4 address" in the AWS Console for k8s-Node1
2.	Open your browser and go to: https://<YOUR-NODE-PUBLIC-IP>:30715
Important Note:
•	You will see a "Your connection is not private" warning. This is normal because ArgoCD uses a self-signed certificate. Click Advanced -> Proceed.
•	Do not use HTTP. If you type http://...:30715, it will fail because the port expects SSL/TLS.

Phase 4: The Application Code (Git)
Repo: github.com/masimgul/argocd-lab
4.1. Directory Structure
Plaintext
argocd-lab/
├── my-app/
│   ├── deployment.yaml  (Defines the Pods/Replicas)
│   └── service.yaml     (Defines Network Access/NodePort)
4.2. Manifests
deployment.yaml
YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
service.yaml
YAML
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-service
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007 # Fixed port for easy access
  selector:
    app: my-nginx

________________________________________
Phase 5: The "GitOps Glue" (Argo Application)
This manifest tells ArgoCD to sync the Git repo with the Cluster.
Run this on the cluster (Control Plane):
Bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-nginx-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/masimgul/argocd-lab.git
    targetRevision: HEAD
    path: my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

________________________________________
Verification
1.	ArgoCD UI: Check https://<NodeIP>:30715 → Status should be "Synced" & "Healthy".
2.	Application: Check http://<NodeIP>:30007 → Should see "Welcome to Nginx".
3.	GitOps Workflow:
o	Change replicas: 3 to replicas: 5 in GitHub.
o	Wait 3 minutes (or click Refresh).
o	Observe 5 pods running in the cluster automatically.


This is a massive milestone! You have successfully built a production-grade GitOps pipeline on custom AWS infrastructure.
To recap what you have achieved:
1.	Infrastructure as Code: You used Terraform to provision EC2 instances.
2.	Cluster Ops: You bootstrapped Kubernetes manually with kubeadm (the "hard way").
3.	Networking: You navigated complex AWS Security Groups and CNI (Flannel) issues.
4.	GitOps: You installed ArgoCD and established a "Pull" based deployment from GitHub.
The "Full Cycle" DevOps View
Right now, you have the CD (Continuous Deployment) part working perfectly.
•	Current Flow: Change YAML in Git $\rightarrow$ ArgoCD sees change $\rightarrow$ Cluster updates.

However, you are still manually editing the YAML to change the image tag (e.g., nginx:1.21). In a real job, this happens automatically.

o use your custom container g3niuz/myflix, you simply need to update the image field in your deployment.yaml.
Since your repository is public on Docker Hub, Kubernetes can pull it without any special authentication (like imagePullSecrets).
Here is your updated deployment.yaml.
Updated Deployment Manifest
File: my-app/deployment.yaml
YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: myflix-container      # I renamed this from 'nginx' to be more accurate
        image: g3niuz/myflix:latest # <--- CHANGED HERE
        ports:
        - containerPort: 80         # Ensure your 'myflix' app listens on port 80 internally

Next Steps (GitOps Workflow)
Since you are using ArgoCD, do not apply this with kubectl. instead:
1.	Commit & Push: Update the file in your local folder and push it to GitHub.
Bash
git add my-app/deployment.yaml
git commit -m "Update image to g3niuz/myflix"
git push origin main
2.	Sync: Go to your ArgoCD UI and click Refresh (or wait 3 minutes).
3.	Verify: ArgoCD will pull the new image. You can verify the new pod is running with:
Bash
kubectl describe pod -l app=my-nginx
# Look for "Image: g3niuz/myflix"
Important Note: If your myflix app listens on a port other than 80 (e.g., 3000 or 8080), make sure you update containerPort: 80 in the YAML above to match your app's code.


Now if I wanna add one more deployment/app, should i need to add another deployment file such as "deployment_bucks.yaml" and a new SVC too...
can it be added in the same GIT repo or need to create a new repo ?

It worked! You are now successfully operating a GitOps workflow.
To answer your questions:
1.	Files: Yes, you need a new Deployment and Service file for the new app.
2.	Repo: You can (and should) use the same Git repository. You don't need a new repo for every app.
The standard industry practice is the "Folder-per-App" strategy (Monorepo). You simply create a new folder inside your existing repo.
Here is the step-by-step guide to adding your second app ("bucks").
________________________________________
Step 1: Organize Your Git Repo
On your laptop (inside argocd-lab), create a new folder for the new app.
Current Structure:
Plaintext
argocd-lab/
├── my-app/             <-- Your existing Nginx/MyFlix app
│   ├── deployment.yaml
│   └── service.yaml
New Structure:
Plaintext
argocd-lab/
├── my-app/             <-- Existing
│   ├── ...
├── bucks-app/          <-- NEW FOLDER
│   ├── deployment.yaml
│   └── service.yaml
Step 2: Create the Manifests
Create the files inside the bucks-app folder.
File 1: bucks-app/deployment.yaml (Let's use a different image, e.g., httpd (Apache) or another custom image, so you can tell them apart).
YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bucks-deployment
  labels:
    app: bucks
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bucks
  template:
    metadata:
      labels:
        app: bucks
    spec:
      containers:
      - name: bucks-container
        image: httpd:alpine  # Or use g3niuz/bucks if you have one
        ports:
        - containerPort: 80

File 2: bucks-app/service.yaml Crucial: You must use a different NodePort (e.g., 30008) because 30007 is already taken by your first app.
YAML
apiVersion: v1
kind: Service
metadata:
  name: bucks-service
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30008  # <--- MUST BE UNIQUE
  selector:
    app: bucks

Step 3: Push to Git
ArgoCD can't see it until it's on GitHub.
Bash
git add bucks-app/
git commit -m "Add new bucks application"
git push origin main
________________________________________
Step 4: Tell ArgoCD (Create the 2nd App)
Just because the folder exists doesn't mean ArgoCD will deploy it automatically. You need to create a new Application resource that points to this specific folder.
Run this command on your cluster (or click "New App" in the UI):
Bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bucks-app           # Unique Name in Argo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/masimgul81/argocd-lab.git
    targetRevision: HEAD
    path: bucks-app         # <--- POINT TO THE NEW FOLDER
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

Step 5: Verify & Access
1.	ArgoCD UI: You will now see two tiles: my-nginx-app and bucks-app.
2.	AWS Security Group: Open port 30008 in your AWS Firewall (just like you did for 30007).
3.	Browser: Go to http://<Node1-Public-IP>:30008.
Summary: You keep one "Infrastructure Repo" (argocd-lab) and just add folders for every new project (backend, frontend, db, bucks, etc.). ArgoCD manages them all side-by-side.


