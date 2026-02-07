# KServe

## Table of Contents
1. [Introduction to KServe](#introduction-to-kserve)
2. [KServe Architecture](#kserve-architecture)
3. [KServe End to End Demo for a sample model](#kserve-end-to-end-demo-for-a-sample-model)
4. [KServe End to End Demo for the Intent Classifier model](#kserve-end-to-end-demo-for-the-intent-classifier-model)
5. [KServe for LLMOps](#kserve-for-llmops)

---

## Introduction to KServe

### What is KServe?

KServe is an **open source platform completely based on Kubernetes** that helps MLOps engineers with:
- Model deployment
- Model serving
- Model inference

### Why KServe?

Let's go back to the **Machine Learning Lifecycle**:

1. **Data Scientists** work on preparing models
   - Get business requirements
   - Prepare datasets
   - Choose algorithms
   - Train the algorithm on data
   - Output: Machine Learning models (stored as `.pkl`, joblib, or framework-specific formats)

2. **Model is Created but Job is Only 50% Done**
   - Model is prepared and saved
   - Model is stored locally or in a centralized registry
   - **But model is NOT served to end users**

3. **MLOps Engineers Come Into Picture**
   - Implement model deployment
   - Implement model serving
   - Handle model inference (when end users request predictions via API)

### The Problem Without KServe

Even with Kubernetes deployment (as we learned in previous sections), there are **many manual activities**:

**Without KServe (Raw Kubernetes approach):**
1. ML engineers create Flask API for the model
2. MLOps engineers implement WSGI
3. MLOps engineers implement Nginx reverse proxy
4. MLOps engineers create Kubernetes deployment manifests
5. MLOps engineers create Kubernetes service manifests
6. Implement autoscaling (HPA, VPA)
7. Implement load balancing
8. Configure networking
9. **Total Time: 2-3 days or more**

### The Solution: KServe

**With KServe, MLOps engineers simply:**
1. Provide the location of the model file (e.g., S3 bucket, GCS bucket, GitHub release)
2. KServe platform automates ALL other activities
3. **Total Time: 2-3 steps**

### Key Advantages of KServe

#### 1. **Multi-Framework Support**
- **Problem:** Data scientists choose different frameworks:
  - Scikit-learn → `.pkl` files
  - TensorFlow → TensorFlow format
  - PyTorch → PyTorch format
  - XGBoost → XGBoost format
  
  Just like DevOps engineers learn Maven for Java, Gradle for Java, Go modules for Go, npm for JavaScript — MLOps engineers had to learn different deployment approaches for each framework.

- **Solution:** KServe supports ALL frameworks
  - No matter which framework data scientists use
  - Just provide the model location
  - KServe takes care of everything
  - **Learning curve for MLOps engineers becomes minimal**

#### 2. **Automatic Scaling**
- **Problem (Raw Kubernetes):** Manual configuration needed
  - Implement HPA (Horizontal Pod Autoscaler)
  - Implement VPA (Vertical Pod Autoscaler)
  - Manually configure Keda or Knative

- **Solution (KServe):**
  - Scales automatically using Knative under the hood
  - Supports both regular autoscaling and serverless scaling
  - Can scale pods from 0 to 10, 20, or more
  - Everything automatic — no manual intervention

#### 3. **Simple Deployment**
- Single inference service manifest instead of 3+ separate manifests
- No need to write custom serving code
- Built-in model serving capabilities

---

## KServe Architecture

### Real-World Scenario: Netflix Recommendation Engine

Imagine Netflix wants to implement KServe:
- Netflix has a recommendation engine (set of ML models)
- Data scientists create recommendation models
- **Goal:** Ship models to production faster
- **Solution:** Implement KServe

### Step-by-Step Architecture Explanation

#### **Step 1: Set up Kubernetes Cluster**
```
MLOps engineers either:
- Create a new Kubernetes cluster, OR
- Reuse existing cluster (shared with DevOps engineers)
```

#### **Step 2: Create KServe Namespace**
```
- Create a dedicated namespace: kserve-namespace
- This separates KServe resources from other workloads
```

#### **Step 3: Install KServe Controller**
```
- Using Helm charts or manifests
- Creates a cluster-scoped resource
- This controller watches ALL namespaces in the cluster
- Example: Recommendation service namespace, Payments namespace, etc.
```

#### **Step 4: KServe Controller Capabilities**
The KServe controller is **cluster-scoped** and can watch multiple namespaces:

```
Kubernetes Cluster
├── Recommendation Service Namespace
│   └── (watched by KServe controller)
├── Payments Namespace
│   └── (watched by KServe controller)
└── X, Y, Z Namespaces
    └── (watched by KServe controller)
```

#### **Step 5: Create InferenceService (Custom Resource Definition)**
- MLOps engineers create an `InferenceService` CRD in their namespace
- Provides model location (where data scientist stored the model)
- Example: S3 bucket, GCS bucket, GitHub release, etc.

#### **Step 6: KServe Controller Watches and Acts**
```
MLOps Engineer creates InferenceService CRD
           ↓
    KServe Controller watches it
           ↓
    Controller reads model location from CRD
           ↓
    Controller automatically creates:
    - Deployment
    - Horizontal Pod Autoscaler (HPA)
    - Service
    - Ingress (or Gateway API)
           ↓
    Pod is created with model container running
```

### Complete Architecture Flow Diagram

```
InferenceService CRD Created
          ↓
  KServe Controller
(watches all namespaces)
          ↓
    Reads model location
          ↓
   Creates Resources:
   ├── Deployment
   ├── HPA (Horizontal Pod Autoscaler)
   ├── Service
   └── Ingress / Gateway API
          ↓
    Pod with Model Container
          ↓
End Users access via
Ingress URL / Gateway API
          ↓
   Model Prediction Response
```

### Key Points

1. **MLOps Engineer's Simple Job:**
   - Create Kubernetes cluster (or use existing)
   - Create namespace for KServe
   - Create KServe controller (one-time setup)
   - For each project: Create InferenceService CRD with model location

2. **KServe Controller Actions:**
   - Watches InferenceService CRDs across all namespaces
   - Automatically creates Deployment with HPA
   - Creates Service
   - Creates Ingress or Gateway API (configurable)

3. **Model Container Support:**
   - Supports multiple frameworks (pkl, joblib, TensorFlow, PyTorch, etc.)
   - Framework-specific model servers handle the actual inference

4. **Access Pattern:**
   - End users access models via Ingress URL or Gateway API
   - Model container serves predictions inside the pod

---

## KServe End to End Demo for a sample model

### Prerequisites
- Kind (Kubernetes in Docker) installed
- Helm installed
- kubectl installed
- Docker running

### Step 1: Create Local Kubernetes Cluster

```bash
kind create cluster --name=mlops-kserve-demo
```

**Expected Output:** Takes 60 seconds to 2 minutes

### Step 2: Verify Cluster Connection

```bash
kubectl config current-context
```

**Why this is important:**
- You might have multiple clusters (prod, dev, staging)
- Ensure kubectl is pointing to the correct cluster
- Prevents accidental deployments to wrong cluster

### Step 3: Install Cert Manager (Prerequisite)

**Why Cert Manager?**
- KServe needs SSL/TLS for HTTPS
- Cert manager automates certificate management
- No need to manually create, maintain, or rotate certificates

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

**Verify installation:**
```bash
kubectl get pods -n cert-manager
```

**Note:** Wait for all pods to be in Running state before proceeding.

### Step 4: Create KServe Namespace

```bash
kubectl create namespace kserve
```

### Step 5: Install KServe Custom Resource Definitions (CRDs)

**Why install CRDs first?**
- CRDs define custom resources (like InferenceService)
- Controller needs CRDs to exist before it can watch them
- Always install CRDs before installing the controller

```bash
helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd \
  --version v0.16.0 \
  -n kserve \
  --wait
```

### Step 6: Install KServe Controller

```bash
helm install kserve oci://ghcr.io/kserve/charts/kserve \
  --version v0.16.0 \
  -n kserve \
  --set kserve.controller.deploymentMode=RawDeployment \
  --wait
```

**Note:** This takes 60 seconds to 2 minutes as controller pod starts.

### Step 7: Verify Controller Installation

```bash
kubectl get pods -n kserve
kubectl get crds | grep kserve
```

### Step 8: Create Application Namespace

```bash
kubectl create namespace ml
```

### Step 9: Create InferenceService

Create a file `inference-service.yml`:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-iris
  namespace: ml
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
      resources:
        requests:
          cpu: "100m"
          memory: "512Mi"
        limits:
          cpu: "1"
          memory: "1Gi"
```

**What this does:**
- Creates an InferenceService named `sklearn-iris`
- Uses SKLearn framework
- Model is stored in Google Cloud Storage
- KServe will download and serve this model

**Deploy it:**
```bash
kubectl apply -f inference-service.yml
```

### Step 10: Verify InferenceService Creation

```bash
kubectl get pods -n kserve

kubectl get inferenceservice -n ml
kubectl describe inferenceservice sklearn-iris -n ml
```

**Check Controller Logs:**
```bash
kubectl logs  <POD-NAME> --all-containers -n kserve
```

**Expected log output:**
- Reconciling inference service sklearn-iris
- Creating deployment
- Creating horizontal pod autoscaler
- Creating service

### Step 11: Verify Resources Created

```bash
# Check horizontal pod autoscaler
kubectl get hpa -n ml

# Check deployment
kubectl get deployment -n ml

# Check service
kubectl get svc -n ml
```

### Step 12: Port Forward Service

```bash
kubectl port-forward -n ml svc/sklearn-iris-predictor 8080:80 --address 0.0.0.0
```

### Step 13: Test Model Prediction

Open new terminal and run:

```bash
curl -s http://localhost:8080/v1/models/sklearn-iris:predict \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [[6.8, 2.8, 4.8, 1.4]]
  }'
```

**Expected Output:**
```json
{"predictions": [2]}
```

**What this means:**
- Prediction value is 2
- Iris flower species classification: 2 = Virginica

### Key Takeaway

**MLOps Engineer's Effort:**
- Create cluster (one-time)
- Install KServe controller (one-time)
- Create InferenceService manifest with model location
- **That's it!** Everything else is automated

---

## KServe End to End Demo for the Intent Classifier model

### Scenario
An organization wants to deploy their **Intent Classifier MLOps** model using KServe:
- Starting from scratch (no cluster)
- Complete end-to-end deployment
- Real-world approach using GitHub releases

### Prerequisites
- Kind installed
- Helm installed
- kubectl installed
- [Intent Classifier MLOps - KServe Branch](https://github.com/tagore8661/intent-classifier-mlops/tree/kserve)

### Step 1: Create Kubernetes Cluster

```bash
kind create cluster --name=kserve-demo-intent
```

### Step 2: Install Cert Manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Verify
kubectl get pods -n cert-manager
```

### Step 3: Create KServe Namespace

```bash
kubectl create namespace kserve
```

### Step 4: Install KServe CRDs

```bash
helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd \
  --version v0.16.0 \
  -n kserve \
  --wait
```

### Step 5: Install KServe Controller

```bash
helm install kserve oci://ghcr.io/kserve/charts/kserve \
  --version v0.16.0 \
  -n kserve \
  --set kserve.controller.deploymentMode=RawDeployment \
  --wait
```

### Step 6: Prepare Model for Deployment

**Where is the model file?**

1. Clone the Intent Classifier repo:
```bash
git clone https://github.com/tagore8661/intent-classifier-mlops
cd intent-classifier-mlops
```

2. Run the Train Data
```bash
python3 model/train.py
```
This creates the model file in `artifacts/intent-model.pkl`

3. Navigate to model artifacts:
```bash
cd model/artifacts
ls -la
```
You should find: `intent-model.pkl`

### Step 7: Upload Model to GitHub Release

**Why GitHub Releases?**
- Real-world approach
- Models are versioned alongside code
- Easy to track model versions
- Download URL is always accessible

**Steps:**
1. Go to GitHub repo: [Intent Classifier MLOps - KServe Branch](https://github.com/tagore8661/intent-classifier-mlops/tree/kserve)
2. Click on **Releases** tab
3. Click **Draft a new release**
4. Fill in:
   - Tag: `v1.0` (or your version)
   - Title: `KServe Deployment v1.0`
5. Scroll down to **Attach binaries**
6. Upload `intent-model.pkl` file
7. Click **Publish Release**

### Step 8: Get Model Download URL

1. Go to the release you just created
2. Right-click on `intent-model.pkl`
3. Select **Copy Link**
4. You now have the model URL: 
   ```
   https://github.com/tagore8661/intent-classifier-mlops/releases/download/v1.0/intent_model.pkl
   ```

### Step 9: Create Intent Namespace

```bash
kubectl create namespace intent
```

### Step 10: Create InferenceService for Intent Classifier

Create file `intent-classifier-kserve.yml`:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: intent-classifier
  namespace: intent
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: "https://github.com/tagore8661/intent-classifier-mlops/releases/download/v1.0/intent_model.pkl"
      resources:
        requests:
          cpu: "100m"
          memory: "512Mi"
        limits:
          cpu: "1"
          memory: "1Gi"
```

**Replace the storageUri with your actual model URL**

### Step 11: Deploy InferenceService

```bash
kubectl apply -f intent-classifier-kserve.yml
```

### Step 12: Verify Deployment

```bash
# Check inference service
kubectl get inferenceservice -n intent

# Check if pods are running
kubectl get pods -n intent

# Check horizontal pod autoscaler
kubectl get hpa -n intent

# Check service
kubectl get svc -n intent
```

### Step 13: Monitor Controller Logs

```bash
kubectl logs <POD-NAME> -n kserve
#(or)
kubectl logs -n kserve -l app=kserve-controller -c kserve-controller --all-containers
```

**Expected logs:**
- Reconciling inference service intent-classifier
- Creating deployment
- Creating service
- Creating HPA

### Step 14: Port Forward Service

```bash
kubectl port-forward -n intent svc/intent-classifier-predictor 8080:80 --address 0.0.0.0
```

### Step 15: Test Intent Classifier Model

Open new terminal:

```bash
# Test 1: Greeting intent
curl -s -X POST http://localhost:8080/v1/models/intent-classifier:predict \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [["Hi, how are you?"]]
  }'

# Test 2: Cancellation intent
curl -s -X POST http://localhost:8080/v1/models/intent-classifier:predict \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [["I want to cancel my subscription"]]
  }'

# Test 3: Different greeting
curl -s -X POST http://localhost:8080/v1/models/intent-classifier:predict \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [["Hello, how are you?"]]
  }'
```

**Expected Output:**
- "Hi, how are you?" → Greeting intent
- "I want to cancel my subscription" → Cancellation intent
- "Hello, how are you?" → Greeting intent

### Summary of Implementation

**What MLOps Engineers Did:**
1.  Created Kubernetes cluster
2.  Installed Cert Manager and KServe (one-time setup)
3.  Prepared model and uploaded to GitHub release
4.  Created InferenceService manifest with model URL
5.  Applied manifest
6.  Model is live and serving predictions

**Time Taken:** Approximately 10 minutes

**Compare with Raw Kubernetes:**
- Raw Kubernetes: Create Dockerfile, Flask API, WSGI config, deployment manifest, service manifest, ingress, setup autoscaling = 2-3 days
- KServe: Provide model location + create InferenceService = 10 minutes

---

## KServe for LLMOps

### KServe is Not Just for MLOps

**KServe is also widely used for LLMOps** (Large Language Models Operations)

### Why Should You Learn KServe for LLMOps?

1. **Current State (MLOps):**
   - You're learning KServe for machine learning models
   - Deploying scikit-learn models, intent classifiers, etc.

2. **Future State (LLMOps):**
   - Same platform (KServe) is used for deploying LLMs
   - Large Language Models like GPT, BERT, LLaMA, etc.
   - Growing field with high demand

3. **Career Advantage:**
   - LLMOps is the future of AI operations
   - If you learn KServe now, you can easily transition to LLMOps
   - Significantly boost your resume
   - Higher market demand = Better opportunities

### Why KServe is Perfect for Both MLOps and LLMOps

**Same Platform, Same Concepts:**
- Framework support (TensorFlow, PyTorch, ONNX, etc.)
- Automatic scaling
- Model versioning
- Canary deployments
- Multi-framework support

**Key Difference:**
- MLOps: Scikit-learn models, XGBoost, small models
- LLMOps: Large Language Models, massive models, GPU requirements
- **But the deployment approach is almost identical!**

---

## Resource Links

- **GitHub Reference:** [Intent Classifier MLOps](https://github.com/tagore8661/intent-classifier-mlops)
- **Branch:** [KServe Branch](https://github.com/tagore8661/intent-classifier-mlops/tree/kserve)
