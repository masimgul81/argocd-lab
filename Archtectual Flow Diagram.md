# 📘 AWS Kubernetes GitOps Platform (v2.0)

**Project Owner:** Masim Gul (`masimgul81`)
**Docker Hub:** `g3niuz`
**Infrastructure:** AWS EC2 (Bare Metal K8s)
**Architecture:** Microservices with GitOps, DevSecOps & Auto-Scaling.

---

## 🏗️ Architectural Diagram

```mermaid
graph TD
    %% Define Styles
    classDef aws fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:white;
    classDef k8s fill:#326CE5,stroke:#fff,stroke-width:2px,color:white;
    classDef devops fill:#e1e4e8,stroke:#333,stroke-width:2px;

    subgraph "External World"
        User["User / Laptop"]
        Dev["Developer"]
        DockerHub["Docker Hub Registry"]
    end

    subgraph "CI/CD Pipeline (GitHub)"
        SourceRepo["Source Code Repo"]
        ConfigRepo["Config Repo"]
        Action["GitHub Actions"]
        Trivy["Trivy Security Scanner"]
    end

    subgraph "AWS Cloud (VPC)"
        SG["Security Group Firewall"]

        subgraph "Kubernetes Cluster (v1.30)"
            subgraph "Control Plane"
                API["API Server"]
                ArgoCD["ArgoCD Controller"]
            end

            subgraph "Worker Nodes"
                Ingress["Nginx Ingress Controller<br/>Port: 31232"]
                
                subgraph "Namespace: Default"
                    Svc1["My-Nginx Service"]
                    Pod1["Pod: MyFlix<br/>(HPA: 1-10 Replicas)"]
                    Svc2["My-Bucks Service"]
                    Pod2["Pod: Bucks<br/>(Replica x2)"]
                end
                
                subgraph "Namespace: Monitoring"
                    Metrics["Metrics Server"]
                end
            end
        end
    end

    %% DevOps Flow
    Dev -->|"1. Push Code"| SourceRepo
    SourceRepo -->|"2. Trigger"| Action
    Action -->|"3. Build & Scan"| Trivy
    Trivy -->|"Pass"| Action
    Action -->|"4. Push Image"| DockerHub
    Action -->|"5. Update YAML"| ConfigRepo
    ArgoCD -->|"6. Detect Change"| ConfigRepo
    ArgoCD -->|"7. Sync & Apply"| API

    %% Traffic Flow
    User -->|"http://myflix.local:31232"| SG
    SG --> Ingress
    Ingress -->|"Host: myflix.local"| Svc1
    Svc1 --> Pod1
    Ingress -->|"Host: bucks.local"| Svc2
    Svc2 --> Pod2
    
    %% Styles
    class SG,EC2 aws;
    class API,Ingress,Pod1,Pod2,Svc1,Svc2,ArgoCD,Metrics k8s;
    class SourceRepo,ConfigRepo,Action,Trivy,DockerHub devops;