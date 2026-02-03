# Model Deployment and Serving using Kubernetes (K8s)

## Table of Contents
1. [Overview](#overview)
2. [Preparing Dockerfile for the Project](#preparing-dockerfile-for-the-project)
3. [Running and Testing the Docker Image Locally](#running-and-testing-the-docker-image-locally)
4. [Pushing the Image to a Model Registry](#pushing-the-image-to-a-model-registry)
5. [Creating and Managing a Kubernetes Cluster](#creating-and-managing-a-kubernetes-cluster)
6. [Preparing Kubernetes Manifests for Model Deployment](#preparing-kubernetes-manifests-for-model-deployment)
7. [Model Serving - Overview](#model-serving---overview)
8. [Ingress Controller Deployment](#ingress-controller-deployment)
9. [Model Serving Using Ingress](#model-serving-using-ingress)

---

## Overview

Previous we covered model deployment and serving using virtual machines.now we will implement the same using a **Kubernetes cluster**.

### Implementation Steps
1.  **Prepare the Dockerfile**: Create a container image for the application since Kubernetes requires containers for deployment.
2.  **Access Model Registry**: Identify a registry (e.g., Docker Hub, ECR, GCR) to store the model images.
3.  **Build and Push**: Build the image from the Dockerfile and push it to the selected Model Registry.
4.  **Prepare Kubernetes Cluster**: Create a Kubernetes cluster and understand best practices for managing it (e.g., multi-tenancy).
5.  **Prepare Deployment Manifests**: Create Kubernetes manifests (`Deployment`, `Namespace`, etc.) to deploy the model.
6.  **Deploy to Kubernetes**: Apply the manifests to deploy the model in the cluster.
7.  **Serve the Model**: Implement an Ingress Controller to expose the model to external users.
8.  **Deploy Ingress Controller**: Deploy the controller that manages external access.
9.  **Create Ingress Resource & Test**: Create the ingress resource and inference the model via an external API call.

---

## Preparing Dockerfile for the Project

To deploy the model on Kubernetes, we first need to bundle it into a Docker container. Below is the step-by-step logic and the implementation of the `Dockerfile`.

### Key Considerations for the Dockerfile
1.  **Base Image**: Use `python:3.10-slim`.
    *   *Reason*: It comes with Python installed and is lightweight (fewer unnecessary dependencies), reducing vulnerability risks (CVEs).
2.  **Working Directory**: Set `/app` as the working directory.
3.  **Layer Caching Strategy**:
    *   Copy `requirements.txt` and install dependencies **before** copying the source code.
    *   *Reason*: Docker caches layers. If source code changes but requirements do not, Docker skips the expensive installation step during rebuilds, speeding up the process.
4.  **Model Training**: Run `train.py` inside the container to generate the model artifact (`.pkl`).
5.  **Expose Port**: Expose port `6000` (this serves as metadata for the user).
6.  **CMD (Entrypoint)**: Use **Gunicorn** instead of the default Flask server (`python app.py`).
    *   *Reason*: Gunicorn supports parallelism (workers) and is production-grade, whereas the Flask development server is single-threaded and not suitable for production.

### Dockerfile Implementation
File: `Dockerfile`

```dockerfile
# 1. Base Image
FROM python:3.10-slim

# 2. Set Environment Variables
# Prevents Python from writing .pyc files to disc
ENV PYTHONDONTWRITEBYTECODE=1
# Prevents Python from buffering stdout and stderr
ENV PYTHONUNBUFFERED=1

# 3. Set Working Directory
WORKDIR /app

# 4. Copy Requirements
COPY requirements.txt .

# 5. Install Dependencies
# Update system dependencies and install python packages
# Note: Combining apt-get update/install and pip install reduces the number of layers and image size
RUN apt-get update \
 && apt-get install -y --no-install-recommends gcc libc-dev \
 && pip install --no-cache-dir -r requirements.txt \
 && apt-get remove -y gcc libc-dev \
 && apt-get autoremove -y \
 && rm -rf /var/lib/apt/lists/*

# 6. Copy Source Code
COPY . .

# 7. Train Model (Generate Artifacts)
RUN python3 model/train.py

# 8. Expose Port (Metadata only)
EXPOSE 6000

# 9. Start Application with Gunicorn
CMD ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:6000", "app:app"]
```

---

## Running and Testing the Docker Image Locally

Before deploying to a cluster, it is crucial to test the image locally to ensure it works as expected.

### Docker Lifecycle
1.  **Dockerfile**: The blueprint.
2.  **Image**: The built artifact (snapshot).
3.  **Container**: The running instance of the image.

### 1. Build the Docker Image
The `docker build` command reads the Dockerfile instructions and creates an image.
```bash
# Syntax: docker build -t <image_name>:<tag> <location_of_dockerfile>
docker build -t mlops-demo:latest .
```
*   **-t**: Tags the image.
*   **.**: Specifies the current directory as the build context.

### 2. Verify the Image Usage
Check if the image was created successfully.
```bash
docker images | grep mlops-demo
```

### 3. Run Container
Run in detached mode (`-d`) and map port 6000 (`-p`).
```bash
# Syntax: docker run -d -p <host_port>:<container_port> <image_name>
docker run -d -p 6000:6000 mlops-demo:latest
```
*   **-d**: Runs in detached mode (background).
*   **-p 6000:6000**: Maps port 6000 on the local machine to port 6000 in the container.

### 4. Test the API (Inference)
Use `curl` to send a request to the locally running container.
```bash
curl -X POST http://localhost:6000/predict \
     -H "Content-Type: application/json" \
     -d '{"text": "Hi, How are you?"}'
```
*Expected Output*: `{"intent": "greeting"}`

---

## Pushing the Image to a Model Registry

A **Model Registry** (or Container Registry) is essential for storing and versioning model images. It acts like Git for Docker images, allowing easy sharing across teams and environments (QA, Staging, Production).

### Why Model Registry?
*   **Versioning**: Track different versions of the model (v1, v2, etc.).
*   **Auditing**: Know who pushed which image and when.
*   **RBAC**: Control access (who can pull/push/delete).
*   **Centralization**: Avoid sharing images via manual file transfers.

### Steps to Push to Docker Hub

1.  **Login to Docker Hub**:
    *   Create a Personal Access Token (PAT) in Docker Hub (Account Settings -> Personal Access Token -> Create New Token).
    *   Login via terminal:
        ```bash
        # Syntax: docker login -u <username>
        docker login -u tagore8661
        # Paste PAT when prompted
        ```

2.  **Tag the Image**:
    *   Images must be tagged with the registry username and repository name.
    ```bash
    # Syntax: docker tag <local_image> <username>/<repo_name>:<tag>
    docker tag mlops-demo:latest tagore8661/intent-classifier-mlops:v1
    ```

3.  **Push the Image**:
    ```bash
    docker push tagore8661/intent-classifier-mlops:v1
    ```

4.  **Verification**:
    *   Visit Docker Hub repositories; you should see the new image. Anyone can now pull this image using `docker pull`.

---

## Creating and Managing a Kubernetes Cluster

There are two primary ways to set up Kubernetes:
1.  **Local Clusters**: Using tools like ***Kind***, ***Minikube***, ***Micro K8's*** or Docker Desktop. Great for development (zero cost), but Ingress is restricted to the local network.
2.  **Managed/Cloud Clusters**: Using **AWS EKS**, Azure AKS, or GKE. Required for production and real-world exposure via public load balancers.

### Setup using AWS EKS (Production Approach)
We will use `eksctl`, a CLI tool for easily creating EKS clusters.

**1. Prerequisites on Cloud Shell**:
*   `aws-cli` (installed)
*   `kubectl` (installed)
*   `eksctl` (install via documentation if missing) - it is a command line utility for managing EKS provided by AWS

**2. Create Cluster Command**:
```bash
eksctl create cluster \
  --name demo-cluster \
  --region ap-south-1 \
  --version 1.32 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```
*   **Note**: This process takes ~15-20 minutes to provision VPCs, Subnets, Control Plane, and Worker Nodes, IAM roles & required resources...
*   You Can Open **CloudFormation** and Check the ***Stack*** that is Creating all Setup

**3. Update Kubeconfig**:
Connect your local `kubectl` to the new cluster.
```bash
aws eks update-kubeconfig --name demo-cluster --region ap-south-1
# Shows the current Kubernetes context (cluster/user/namespace)
kubectl config current-context
```

**4. Delete Cluster(Cleanup)**:
```bash
eksctl delete cluster --name demo-cluster --region ap-south-1
```

### Managing Clusters in Organizations (Interview Guide)

This is a critical topic in interviews. Candidates often fail to clearly articulate **Multi-Tenancy** strategies. Here is the industry-standard architecture for managing shared clusters.

**1. Cluster Strategy (Per Environment)**
   *   **Standard Pattern**: Organizations typically maintain **one shared cluster per environment** (e.g., `dev-cluster`, `staging-cluster`, `prod-cluster`).
   *   **Anti-Pattern**: Creating a separate cluster for every team. This results in "Cluster Sprawl," causing massive operational overhead (patching, upgrading, security auditing) and wasted resources (idle control planes).

**2. Isolation Layers (The "Defense in Depth" Approach)**
   We use a **Soft Multi-Tenancy** model where teams share the infrastructure but remain isolated logically and via networking.

   *   **Physical Layer (Nodes)**: *The Foundation*
       *   **Role**: Underlying compute capacity (EC2/VMs). These are generally **shared** across all teams to maximize utilization.
       *   **Analogy**: The **Nodes** are the physical building structure (floors, electricity) that everyone uses.
   
   *   **Logical Layer (Namespaces)**: *The Workspaces*
       *   **Role**: Each Domain/Team gets a dedicated Namespace.
       *   **Real-World Naming**:
           *   **Checkout Team**: Operates in `checkout-ns`. Manages services like `payment-gateway`, `order-processor`.
           *   **Inventory Team**: Operates in `inventory-ns`. Manages services like `catalog-api`, `stock-syncer`.
       *   **Analogy**: The **Namespaces** are the secure, badge-access rooms. Team A cannot accidentally modify Team B's work.

   *   **Network Layer (Network Policies)**: *The Firewall*
       *   **Role**: By default, K8s allows all traffic. We use **NetworkPolicies** to enforce boundaries.
       *   **Rule**: "Deny all traffic to `checkout-ns` unless it comes from `frontend-ns`." This prevents lateral movement if a non-critical service is compromised.

**3. Resource Governance (FinOps & Stability)**
   *   **Resource Quotas (Namespace Level)**: *The Budget*
       *   **Concept**: The Platform team gives the Checkout team a "Budget" of CPU/RAM.
       *   **Implementation**: 
           ```yaml
           # checkout-ns Quota
           requests.cpu: "10"    # Guaranteed 10 Cores total for all their pods
           limits.memory: "20Gi" # Hard limit of 20GB RAM
           ```
   *   **Requests & Limits (Pod Level)**: *The Allocation*
       *   **Concept**: The Checkout team decides how to spend their budget.
       *   **Example**: They assign `2.0 CPU` to the heavy `payment-gateway` but only `0.1 CPU` to the light `notification-worker`.

**4. Security (RBAC)**
   *   **RoleBinding**: We bind the `checkout-dev-group` (from IDP like Okta/AD) to the `admin` Role **only** within `checkout-ns`.
   *   **Outcome**: A developer can run `kubectl delete pod` in `checkout-ns`, but gets `403 Forbidden` if they try it in `inventory-ns`.

---

## Preparing Kubernetes Manifests for Model Deployment

We cannot deploy a container directly; we must use Kubernetes objects. The standard deployment order is: **Namespace -> Deployment -> Service**.

### 1. Namespace (`namespace.yaml`)
**Purpose**: Creates a logical "workspace" or "bucket" for our project.

*   **Why use it?**
    *   **Isolation**: Keeps our resources (pods, services) separate from other teams (e.g., preventing name collisions).
    *   **Resource Management**: Allows us to attach Resource Quotas (CPU/RAM limits) to the entire project.
    *   **Access Control**: We can give a user access *only* to this namespace.

```yaml
apiVersion: v1              # API Version for Namespace
kind: Namespace             # Type of Resource
metadata:
  name: intent-namespace    # Name of our isolated workspace
```
*Command*: `kubectl apply -f namespace.yaml`

### 2. Deployment (`deployment.yaml`)
**Purpose**: Manages the lifecycle of our Application/Model Pods. It is the "Manager" that ensures the pods are healthy and running.

*   **Why not create a Pod directly?**
    *   Pods are fragile. If they crash or the node dies, they are gone forever.
    *   **Self-Healing**: A Deployment creates a **ReplicaSet**, which ensures a specified number of replicas (e.g., 2) are *always* running. If a pod dies, the Controller immediately starts a new one to replace it.
*   **Key Fields**:
    *   **`replicas`**: The target number of identical pods (Load Balancing).
    *   **`selector`**: How the Deployment finds which pods belong to it (must match `template.labels`).
    *   **`template`**: The blueprint for creating the actual Pods (contains the container image info).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: intent-classifier         # Name of the Deployment
  namespace: intent-namespace     # Deploys into our specific workspace
spec:
  replicas: 2                     # Maintain 2 copies for high availability
  selector:
    matchLabels:
      app: intent-classifier      # MANAGING pods with this label (Critical Link)
  template:                       # The Pod Blueprint (What to run)
    metadata:
      labels:
        app: intent-classifier    # Label assigned to every new Pod
    spec:
      containers:
      - name: intent-classifier
        image: tagore8661/intent-classifier-mlops:v1 # The Image from Docker Hub
        ports:
        - containerPort: 6000     # The port the container listens on
```
*Command*: `kubectl apply -f deployment.yaml -n intent-namespace`

**Check pod is Running**: `kubectl get pods -n ntent-namespace`

### 3. Service (`service.yaml`)
**Purpose**: Acts as a stable "Front Door" (Static IP/DNS) for our dynamic Pods. It acts as a Load Balancer.

*   **The Problem (The "Address Change" Issue)**:
    *   Pods are **ephemeral** (temporary). When they restart, they get a **new IP Address** (e.g., `172.61.0.1` -> `172.16.0.5`).
    *   You cannot give a user the Pod IP because it will change tomorrow.
*   **The Solution**:
    *   A Service provides a **Stable IP** that *never* changes.
    *   It listens for traffic and **forwards** it to essentially "Current Healthy Pods".
*   **Key Fields**:
    *   **`selector`**: The most important field. It tells the Service *which* pods to send traffic to (must match `app: intent-classifier`).
    *   **`ports`**: Maps the Service's input port to the Container's target port.

**Traffic Flow**: User -> **Service** (Static IP) -> **Pod** (Dynamic IP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: intent-service
  namespace: intent-namespace
spec:
  type: NodePort                  # Exposes IP on Node (Use 'ClusterIP' for internal only)
  selector:
    app: intent-classifier        # LINKS Service to Deployment Pods (Critical Link)
  ports:
    - port: 80                    # The Service listens on Port 80 (external)
      targetPort: 6000            # Forwards traffic to Container Port 6000 (internal)
```
*Command*: `kubectl apply -f service.yaml -n intent-namespace`

**Verification**: `kubectl get all -n intent-namespace`

### Summary: The Big Picture

| Resource | Role | Why is it needed? | Real-World Analogy |
| :--- | :--- | :--- | :--- |
| **Namespace** | **Logical Isolation** | To isolate teams (Payments vs Login) and Resources (Quotas) | **Rooms** in an office building. |
| **Deployment** | **Pod Manager** | Ensures Pods are running (Self-healing). Updates App versions. | **Manager** who ensures 2 workers are always present. |
| **Service** | **Stable Proxy** | Handling ephemeral IPs. Providing a single stable address. | **Receptionist** who routes calls to valid desks. |

---

## [Model Serving] - Overview

Why do we need sophisticated **Model Serving** (Ingress) if a Service (NodePort) works?

### The Problem with NodePort
In a secure production environment (e.g., AWS VPC), nodes are often in **Private Subnets**. This means they have no public IP addresses. External users (internet) cannot access `NodeIP:Port` directly.

### The Solution: Ingress
**Ingress** provides a bridge between the outside world and the internal services.
1.  **Ingress Controller**: A specialized Load Balancer (e.g., Nginx, Traefik, ALB) deployed in a public subnet. It listens for traffic from the internet.
2.  **Ingress Resource**: A configuration file that defines rules (e.g., "Send logic for `example.com/predict` to `intent-service`").

**Traffic Flow**:
Internet (User) -> **Load Balancer (Ingress Controller)** -> **Service** -> **Pod** -> **Model**

---

## Ingress Controller Deployment

We will use **Traefik** as our Ingress Controller. It is modern, lightweight, and efficient. 
*   The Ingress Controller is usually a **cluster-wide** installation (managed by Platform/MLOps teams).
*   It watches for Ingress resources created by individual project teams and automatically updates routing rules.

### Installation via Helm
1. **Install Helm If Not Exit**
    ```bash
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```
2.  **Add Helm Repository**:
    ```bash
    helm repo add traefik https://helm.traefik.io/traefik
    helm repo update
    ```
3.  **Install Traefik**:
    ```bash
    helm install traefik traefik/traefik --create-namespace --namespace traefik
    ```
4.  **Verify Installation**:
    ```bash
    kubectl get pods -n traefik
    ```
*   You should see the Traefik controller running.
5.  **Retrieve the Load Balancer Endpoint**:
    ```bash
    kubectl get svc -n traefik
    ```
*   Look for the traefik service of type LoadBalancer, and use the external DNS name to access applications via Ingress.

---

## Model Serving Using Ingress

With the controller ready, we define specific routing rules for our model.

**Keep the Logs for Floating**
```bash
kubectl get pods -n traefik
#Copy the <TRAEFIK_POD_NAME>
kubectl logs <TRAEFIK_POD_NAME> -n traefik -f
#Here we saw the Logs, Open New Terrminal and Run the following commands
```

### Ingress Resource (`ingress.yaml`)
This file tells Traefik: "If request comes to `example.com/predict`, forward it to `intent-service`".

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: intent-ingress
  namespace: intent-namespace
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  ingressClassName: traefik
  rules:
  - host: example.com
    http:
      paths:
      - path: /predict
        pathType: Prefix
        backend:
          service:
            name: intent-service
            port:
              number: 80
```
*Command*: `kubectl apply -f ingress.yaml`

### Testing Real-World Access (The `curl` Hack)
Since we don't own the domain `example.com` and haven't bought a DNS record, we can simulate DNS resolution using `curl`.

1.  **Get Load Balancer IP**:
    Find the external IP/DNS of the Traefik service.
    ```bash
    kubectl get svc -n traefik
    #or
    kubectl get ingress -n intent-namespace
    ```
    *Copy the External IP (e.g., `a1b2c3d4...elb.amazonaws.com` or `35.x.x.x`)*.

2.  **Resolve Locally**:
    If it's a hostname (AWS ELB), find its IP:
    ```bash
    nslookup <YOUR_LOAD_BALANCER_HOSTNAME>
    #or
    getent ahosts <INGRESS_ADDRESS>
    ```

3.  **Curl Command with `--resolve`**:
    Force `curl` to resolve `example.com` to your Load Balancer's IP.
    ```bash
    # Syntax: curl --resolve <host>:<port>:<LB_IP> ...
    
    curl -X POST http://example.com/predict \
         --resolve example.com:80:<LOAD_BALANCER_IP> \
         -H "Content-Type: application/json" \
         -d '{"text": "Hello, how are you?"}'
    ```

**Result**: You should receive a prediction (e.g., `{"intent": "greeting"}`), confirming that the model is successfully served via the production-grade Ingress architecture.

---

### Resource Link
- **GitHub Reference:** [Intent Classifier MLOps](https://github.com/tagore8661/intent-classifier-mlops)
- **Branch:** [K8s Branch](https://github.com/tagore8661/intent-classifier-mlops/tree/k8s)
