# Customer Churn MLOps Project - End-to-End MLOps Implementation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Real-Time Use Case](#real-time-use-case)
3. [Project Flow in Organization](#project-flow-in-organization)
4. [Technology Stack](#technology-stack)
5. [Project Setup](#project-setup)
6. [Data Version Control with DVC](#data-version-control-with-dvc)
7. [Model Deployment with KServe](#model-deployment-with-kserve)
8. [CI/CD Pipeline with GitHub Actions](#cicd-pipeline-with-github-actions)
9. [GitOps with Argo CD](#gitops-with-argo-cd)

---

## Project Overview

### Customer Churn Model Implementation
- **Industry Application**: Telecom, retail, e-commerce, quick commerce
- **Business Problem**: Customer retention and churn prediction
- **Solution**: ML model to predict customer churn probability
- **MLOps Approach**: End-to-end automation using open-source stack

### Why This Project?
- **Real-world relevance**: Widely used across industries
- **Resume value**: Demonstrates practical MLOps experience
- **Learning outcome**: Complete MLOps implementation without Kubeflow
- **Industry demand**: Popular stack in job descriptions

---

## Real-Time Use Case

### Telecom Industry Example
**Companies**: AT&T, Verizon, T-Mobile, etc.

#### Business Problem
- **High Competition**: Customers easily switch between providers
- **Customer Retention**: Need to identify potential churners
- **Support Team Challenge**: Non-technical users need prediction capability

#### Solution Workflow
1. **Support Team** identifies need for churn prediction
2. **Development Team** recognizes ML requirement
3. **Data Science Team** builds prediction model
4. **ML Engineers** create API interface
5. **MLOps Engineers** automate deployment and scaling

#### Business Impact
- **Proactive Retention**: Target offers to potential churners
- **Customer Experience**: Personalized offers and discounts
- **Revenue Protection**: Reduce customer attrition rate

### Data Points Used
- **Age**: Younger customers more likely to churn
- **Tenure**: Longer tenure = lower churn probability
- **Monthly/Yearly Charges**: Higher costs may increase churn
- **Support Cases**: More support tickets = higher churn risk
- **Usage Patterns**: Service utilization metrics

---

## Project Flow in Organization

### Management Decision
```
Management → Data Science Team → ML Engineers → MLOps Engineers
```

### Data Science Team Responsibilities
1. **Data Collection**: Historical customer data (3-10 years)
2. **Dataset Preparation**: Feature engineering and selection
3. **Model Training**: Algorithm selection (Random Forest, Logistic Regression)
4. **Pattern Recognition**: Mathematical function development
5. **Model Validation**: Accuracy testing and evaluation

### ML Engineers Responsibilities
1. **API Development**: FastAPI/Flask interfaces
2. **Performance Optimization**: Response time and scalability
3. **Containerization**: Docker packaging
4. **Documentation**: API specs and usage guides

### MLOps Engineers Responsibilities
1. **Data Version Control**: DVC implementation
2. **Model Registry**: S3-based model storage
3. **CI/CD Automation**: GitHub Actions workflows
4. **Deployment**: KServe on Kubernetes
5. **Scaling**: HPA and resource management
6. **GitOps**: Argo CD for continuous deployment

---

## Technology Stack

### Core Components
- **DVC**: Data version control and remote storage
- **KServe**: Model serving and inference
- **Kubernetes**: Container orchestration and scaling
- **GitHub Actions**: CI/CD automation
- **Argo CD**: GitOps continuous deployment
- **AWS S3**: Model registry and data storage

### Architecture Flow
```
GitHub Repository → GitHub Actions → S3 Bucket → Argo CD → Kubernetes Cluster → KServe → Model API
```

### Stack Advantages
- **Open Source**: No vendor lock-in
- **Industry Standard**: Widely adopted in enterprises
- **Scalable**: Kubernetes-based infrastructure
- **Automated**: End-to-end CI/CD pipeline
- **Version Controlled**: Complete traceability

---

## Project Setup

### Local Development Setup
```bash
# Clone repository
git clone https://github.com/tagore8661/customer-churn-mlops
cd customer-churn-mlops

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Generate training data
python generate_data.py
# This generates 1000 samples and stores them in data/churn_data.csv

# Train model
python train.py
# This trains the model and stores it in models/churn_model.pkl

# Start API service
python api.py
```

### API Usage
```bash
# Test model prediction
curl -X POST "http://localhost:8000/predict" \
-H "Content-Type: application/json" \
-d '{
  "age": 70,
  "tenure_months": 60,
  "monthly_charges": 30,
  "total_charges": 1000,
  "num_support_calls": 1
}'

#or 
Visit http://localhost:8000/docs for the interactive Swagger UI.
```

### Expected Output
```json
{
  "churn": 0,
  "churn_probability": 0.28
}
```

---

## Data Version Control with DVC

### DVC Setup
```bash
# Install DVC
pip install dvc
pip install dvc-s3

# Initialize DVC
dvc init

# Configure remote storage
dvc remote add -d s3remote s3://customer-churn-mlops-tagore
```

### Data Versioning Workflow
```bash
# Add data to DVC
dvc add data/churn_data.csv

# Track model with DVC
dvc add models/churn_model.pkl

# Push to remote storage
dvc push

# Track metadata in Git
git add data/churn_data.csv.dvc models/churn_model.pkl.dvc .gitignore
git commit -m "Add dataset and model version control"
git push
```

### DVC Benefits
- **Version Control**: Track dataset and model changes
- **Remote Storage**: Large files (CSV + model) in S3
- **Team Collaboration**: Consistent data and model versions
- **Metadata Tracking**: Checksums and file info for both artifacts
- **Complete ML Artifact Management**: Both training data and trained models versioned together

---

## Model Deployment with KServe

### Kubernetes Cluster Setup
```bash
# Create KIND cluster
kind create cluster --name=churn-model-cluster

# Install Cert Manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Verify
kubectl get pods -n cert-manager

# Create KServe Namespace
kubectl create namespace kserve

# Install KServe CRDs
helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd \
  --version v0.16.0 \
  -n kserve \
  --wait

# Install KServe Controller
helm install kserve oci://ghcr.io/kserve/charts/kserve \
  --version v0.16.0 \
  -n kserve \
  --set kserve.controller.deploymentMode=RawDeployment \
  --wait

# Verify
kubectl get pods -n kserve

```

### Service Account Configuration

**Why Service Account is Needed**: We can't deploy the model from the S3 bucket even if we write the `inference.yml` file, because our S3 bucket is private. That's why we will create the `serviceaccount.yml` file in k8s and grant access to the S3 bucket using AWS credentials. Within the serviceaccount, we will grant the S3-related permissions.

Create a file `k8s/serviceaccount.yml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: churn

---

apiVersion: v1
kind: Secret
metadata:
  name: s3-secret
  namespace: churn
  annotations:
    serving.kserve.io/s3-endpoint: s3.amazonaws.com
    serving.kserve.io/s3-usehttps: "1"
    serving.kserve.io/s3-region: us-east-1
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: <ACCESS_KEY_ID>
  AWS_SECRET_ACCESS_KEY: <SECRET_ACCESS_KEY>

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-s3-access
  namespace: churn
secrets:
- name: s3-secret
```
*Command*: `kubectl apply -f serviceaccount.yml`

**Verification**: `kubectl get sa -n churn`

### KServe Inference Service

**Important Note**: DVC stores files in a hidden hash format (`files/md5/...`) that KServe cannot use directly, so we must copy the model to a normal S3 path like `s3://.../models/churn_model.pkl` for serving. This separates **versioning (DVC)** from **serving (KServe)**.

```bash
# Pull model from DVC storage
dvc pull models/churn_model.pkl

# Copy model to serving path
aws s3 cp models/churn_model.pkl s3://customer-churn-mlops-tagore/models/churn_model.pkl

# Now deploy inference service
```

Create a file `k8s/inference.yml`
```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: churn-predictor
  namespace: churn
spec:
  predictor:
    serviceAccountName: sa-s3-access
    sklearn:
      storageUri: s3://customer-churn-mlops-tagore/models/churn_model.pkl
```
*Command*: `kubectl apply -f inference.yml`

**Verification**: `kubectl get pods -n churn -w` `kubectl get svc -n churn`

### Port Forward Service
```bash
kubectl port-forward -n churn svc/churn-predictor-predictor 8081:80 --address 0.0.0.0
```

### Test Deployed Model
Open a new terminal and test the deployed model:
```bash
curl -X POST http://localhost:8081/v1/models/churn-predictor:predict \
-H "Content-Type: application/json" \
-d '{
  "instances": [
    [70, 60, 79.99, 1920.00, 3]
  ]
}'
```
#### Expected Output
```json
{
  "predictions": [0]
}
```

---

## CI/CD Pipeline with GitHub Actions

### Workflow Configuration
Create a file `.github/workflows/mlops-pipeline.yml`
```yaml
name: MLOps Pipeline

on:
  push:
    branches: [mlops]

env:
  AWS_REGION: us-east-1
  S3_BUCKET: customer-churn-cicd-mlops
  # For AWS Credentials we are Using Secrets

jobs:
  train-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write # for Updating the Version of model in inference.yml or Creating any new file
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      with:
        token: ${{ secrets.GITHUB_TOKEN }} # Use built-in Token
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        
    - name: Generate dataset
      run: python generate_data.py
    
    - name: Train model
      run: python train.py
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Push model to S3 Bucket Directly
      run: |
        aws s3 cp models/churn_model.pkl s3://${{ env.S3_BUCKET }}/models/
        echo "Model Pushed to S3 Bucket is Success"
    
    - name: Update inference.yml with S3 model path
      run: |
        MODEL_URI="s3://${{ env.S3_BUCKET }}/models/churn_model.pkl"
        sed -i "s|storageUri: .*|storageUri: $MODEL_URI|g" k8s/inference.yml
    
    - name: Commit updated inference.yml
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"
        git add k8s/inference.yml
        git commit -m "Update model S3 path [skip ci]" || echo "No changes"
        git push || echo "No changes to push"
```

### GitHub Secrets Setup

**Steps to Add Secrets:**

1. Go to your GitHub repository
2. Click **Settings** tab → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Enter name: `AWS_ACCESS_KEY_ID` and your AWS access key
5. Click **Add secret**
6. Repeat for `AWS_SECRET_ACCESS_KEY` with your AWS secret key

**Required Secrets:**
1. **AWS_ACCESS_KEY_ID**: Your AWS access key ID
2. **AWS_SECRET_ACCESS_KEY**: Your AWS secret access key
3. **GITHUB_TOKEN**: Default GitHub token (automatically available)

### Pipeline Benefits
- **Automation**: No manual intervention
- **Consistency**: Same process every time
- **Version Control**: Track model changes
- **Integration**: Seamless deployment workflow

---

## GitOps with Argo CD

### Argo CD Installation
```bash
# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verify
kubectl get pods -n argocd

# Access Argo CD UI
kubectl port-forward svc/argocd-server 8080:80 -n argocd --address 0.0.0.0
# Visit "http://localhost:8080"

# Get UserID and Password
kubectl get secrets -n argocd
# We will see `argocd-initial-admin-secret`

# To Read the Secret
kubectl edit secrets/argocd-initial-admin-secret -n argocd

# Decode the Encoded Base64 Password
echo <PASSWORD> | base64 --decode

```
### ArgoCD AWS Secrets Configuration

**Why we need to edit secrets**: In our GitHub repo's `serviceaccount.yml` file, we didn't pass actual AWS credentials (they were placeholders `<ACCESS_KEY_ID>`). Without real credentials, KServe cannot access the private S3 bucket, causing CrashLoopBackOff errors.

```bash
# Two approaches for secret management:
# 1. Edit secrets locally (simple, for development)
# 2. Use secret manager solutions like Seal Secrets Operator or CSI Secret Store Operator (production)

# Edit secrets locally (for development/demo)
kubectl edit secret s3-secret -n churn
# Replace placeholder values with your actual AWS credentials
# This opens a text editor - save and close after making changes
```

### Manual Setup to Create Application in ArgoCD

**UI-Based Application Creation:**
-> Click on NEW APP in ArgoCD UI

**General Tab:**
- Application Name: kserve
- Project Name: default
- Sync Policy: Automatic - Enable Auto-Sync

**Source Tab:**
- Repo URL: https://github.com/tagore8661/customer-churn-mlops.git
- Revision: mlops #Branch Name
- Path: k8s

**Destination Tab:**
- Cluster URL: https://kubernetes.default.svc  # Same Cluster
- Namespace: churn

-> Click on CREATE

### Argo CD Application Setup
Create a file `argocd/argocd-application.yml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kserve
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/tagore8661/customer-churn-mlops.git
    targetRevision: mlops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: churn
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Verify ArgoCD Deployment
```bash
# Check if pods are running successfully
kubectl get pods -n churn

# Check services to get the correct service name
kubectl get svc -n churn

# Port Forward to Access the Model
# Replace <POD_NAME> with the actual pod name from 'kubectl get pods'
kubectl port-forward svc/<POD_NAME> 8082:80 --address 0.0.0.0 -n churn
```

### Test ArgoCD-Deployed Model
Open a new terminal and test the deployed model:
```bash
curl -X POST http://localhost:8082/v1/models/churn-predictor:predict \
-H "Content-Type: application/json" \
-d '{
  "instances": [
    [70, 60, 79.99, 1920.00, 3]
  ]
}'
```
#### Expected Output
```json
{
  "predictions": [0]
}
```

### GitOps Workflow
1. **Code Change**: Push to mlops branch
2. **GitHub Actions**: Triggers pipeline
3. **Model Update**: New version pushed to S3
4. **Argo CD**: Detects changes in k8s/ directory
5. **Auto Deployment**: Updates Kubernetes cluster

### Argo CD Benefits
- **Continuous Sync**: Automatic deployment
- **Version Control**: Git as single source of truth
- **Rollback**: Easy version management
- **Multi-cluster**: Deploy to different environments

---

## Cleanup Resources

### Delete Kubernetes Cluster
```bash
# Delete the KIND cluster to clean up all resources
kind delete cluster --name=churn-model-cluster
```

**Note**: This will remove:
- All Kubernetes pods and services
- KServe deployment
- Argo CD installation
- All created namespaces and resources
- The entire KIND cluster

Use this command when you want to completely clean up your local development environment.
