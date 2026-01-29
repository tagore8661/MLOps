# Fundamentals of Model Deployment and Model Serving

## Table of Contents
1. [Introduction to Model Deployment and Model Serving](#introduction-to-model-deployment-and-model-serving)
2. [Popular Deployment and Serving Strategies](#popular-deployment-and-serving-strategies)
3. [Sample Model Analysis: Intent Classifier](#sample-model-analysis-intent-classifier)
4. [Step-by-Step Local Execution Guide](#step-by-step-local-execution-guide)

---

## Introduction to Model Deployment and Model Serving

**Model Deployment and Serving** is the process of taking a trained Machine Learning model from a data scientist's local environment and making it available to end-users in a production setting.

### Key Concepts:
*   **Deployment:** The act of packaging (e.g., .pkl format or container), storing (Model Registry), and placing a specific version of a model into an environment.
*   **Serving:** The infrastructure required to make the model functional for users. This includes providing the **runtime environment** (often via a **model server**).
    *   **Resources:** Providing CPU or GPU resources for inference.
    *   **Exposure:** Making the model API accessible via Ingress, Load Balancers, and API Gateways.
    *   **DevOps Integration:** Implementing Autoscaling, CDN (for global access), and Monitoring.

**Real-world Example:** When **Netflix** builds a recommendation model, the Data Scientists work locally (training, evaluation). ML Engineers then create the API. MLOps engineers take over to ensure this model is deployed and served so millions of users can see recommendations on their front-end devices.

**Core Insight:** Just deploying a model (putting it on a server) isn't enough; you must also provide the **serving infrastructure** for it to be useful to actual users.

---

## Popular Deployment and Serving Strategies

MLOps engineers generally follow four popular paths to bring models to production:

### 1. Virtual Machines (VMs)
*   **How it works:** Run scripts (Flask/FastAPI) directly on a VM (e.g., AWS EC2).
*   **Implementation:** MLOps engineers implement **WSGI gateways**, Load Balancing, Autoscaling, and secure the infrastructure within a **Virtual Private Cloud (VPC)**.
*   **Best for:** Small teams, low to moderate traffic, or simple projects.

### 2. Kubernetes (K8s)
*   **How it works:** Package the model into a Docker container -> Deploy as a Pod -> Use K8s Services/Ingress for exposure.
*   **Best for:** Large-scale organizations requiring high scalability (up to 10k nodes).
*   **Responsibility:** Requires deep Kubernetes knowledge to manage manifests, networking, and clusters.

### 3. Managed MLOps (e.g., Amazon SageMaker)
*   **How it works:** Use a cloud provider's fully managed service to handle infrastructure.
*   **Best for:** Teams with fewer MLOps engineers or those who want to offload infrastructure maintenance.
*   **Trade-off:** Less control over the underlying infrastructure but faster speed-to-market.

### 4. KServe (Cloud Native Serving)
*   **How it works:** A Kubernetes-native approach that uses **CRDs (Custom Resource Definitions)** on top of Knative.
*   **Best for:** Organizations wanting K8s power without the complexity of writing detailed low-level manifests.
*   **Key Feature:** Excellent for **Canary Deployments** (routing traffic between different model versions).

---

## Sample Model Analysis: Intent Classifier

To understand these concepts, we use a **Tiny Text Classifier**.

*   **UseCase:** A chatbot service that predicts the tone of user messages.
*   **Intents:**
    *   *Greeting:* "Hi", "Hello"
    *   *Question:* "How to reset password?"
    *   *Complaint:* "I want to cancel my subscription."
    *   *Praise:* "Great service!"
*   **Structure:**
    *   `train.py`: Trains the algorithm and saves the `intent_model.pkl` artifact.
    *   `app.py`: A Flask API that loads the `.pkl` file and serves predictions on the `/predict` endpoint.

---

## Step-by-Step Local Execution Guide

### 1. Environment Setup
```bash
# Clone the repo and switch to it
git clone https://github.com/tagore8661/intent-classifier-mlops
cd intent-classifier-mlops

# Setup Virtual Environment
python -m venv .venv
source .venv/bin/activate # Linux/Mac
# or .venv\Scripts\activate # Windows

# Install Dependencies
# Requirements include: flask, scikit-learn, joblib, pytest
pip install -r requirements.txt
```

### 2. Model Training
This process generates the mathematical function and saves it as an artifact.
```bash
python model/train.py
```
*Output: `artifacts/intent_model.pkl` created.*

### 3. Model Serving (Local)
Run the Flask server to expose the API on port 6000.
```bash
python app.py
```

### 4. Testing the API (Inference)
Use `curl` to pass inputs and get predictions from the model.
```bash
# Test a Complaint
curl -X POST http://localhost:6000/predict \
     -H "Content-Type: application/json" \
     -d '{"text": "I want to cancel my subscription"}'

# Test a Greeting
curl -X POST http://localhost:6000/predict \
     -H "Content-Type: application/json" \
     -d '{"text": "Hi, How are you?"}'
```

---

### Resources
- **GitHub Repository:** [Intent Classifier MLOps](https://github.com/tagore8661/intent-classifier-mlops)