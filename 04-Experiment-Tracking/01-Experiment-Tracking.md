# Experiment Tracking and MLflow

## Table of Contents
1. [Introduction to Experiment Tracking](#introduction-to-experiment-tracking)
2. [What is MLflow?](#what-is-mlflow)
3. [Basic MLflow Installation (Local Demo)](#basic-mlflow-installation-local-demo)
4. [MLflow on Kubernetes (Local Demo)](#mlflow-on-kubernetes-local-demo)
5. [MLflow Production Setup (PostgreSQL + Kubernetes)](#mlflow-production-setup-postgresql--kubernetes)
6. [Real-time Tracking with MLflow (DVC + MLflow)](#real-time-tracking-with-mlflow-dvc--mlflow)
7. [Comparing Runs Visually](#comparing-runs-visually)
8. [MLflow Quick Revision](#mlflow-quick-revision)

---

## Introduction to Experiment Tracking

Experiment tracking is the process of recording all metadata, parameters, and results during the ML training journey.

### The "Scenario" (Why Tracking Matters):
Imagine a team aiming for **85% efficiency** on a Wine Prediction model:
*   **Experiment 1:** Random Forest + CSV -> **67%** (Low).
*   **Experiment 2:** Logistic Regression + same CSV -> **79%** (Better).
*   **Experiment 3:** XGBoost + Cleaned CSV + Linux High-Memory instance -> **87%** (**Winner!**).
*   **Experiments 4-25:** Various tweaks -> **81%** (Lower).

**The Problem:** If you didn't track Experiment #3, you won't remember exactly which version of the code, which hyper-parameters, or which cleaned dataset version resulted in that 87%. You are forced to settle for the inferior 81%.

### What is tracked in Real-time?
1.  **Parameters (Hyperparameters):** Learning rate, max depth, n_estimators.
2.  **Code Version:** Git Commit ID (source of truth for the algorithm).
3.  **Dataset Version:** DVC Hash (source of truth for the data).
4.  **Metrics:** Accuracy, MSE, RMSE, R2.
5.  **Artifacts:** The actual model file (`model.pkl`), charts, and debug logs.
6.  **System Information:** OS (Linux vs. Windows), CPU, and Memory usage (critical for 1-2% performance gains).

---

## What is MLflow?

MLflow is the industry-standard platform for experiment tracking, model versioning, and deployment.

### Anti-MLOps Pattern: Excel Sheets
*   **Human Error:** Forgetting to record the 67th (best) experiment out of 100.
*   **Decentralization:** Teams A and B having different Excel formats/locations (Messy).
*   **Scalability:** Impossible to query or compare 1,000+ runs manually.

### Shared Responsibility Model:
*   **MLOps Engineers:** Set up and maintain a **Centralized MLflow Server** (VM, Fargate, or Kubernetes).
*   **Data Scientists:** Perform **Instrumentation** (adding MLflow code to Python scripts) to log data automatically.

---

## Basic MLflow Installation (Local Demo)

For proof of concept (PoC) on your machine.

1.  **Installation:**
    ```bash
    mkdir mlflow-basic && cd mlflow-basic
    python -m venv .venv
    source .venv/bin/activate
    pip install mlflow
    ```

2.  **Run MLflow Server:**
    ```bash
    mlflow ui --backend-store-uri sqlite:///mlflow.db --port 7006
    ```
    *   `--backend-store-uri`: Uses a local SQLite file to store run metadata.
    *   `--port 7006`: Default is 5000, but you can specify any.

---

## MLflow on Kubernetes (Local Demo)

Using **Kind** and **Helm** (Community Charts).

1.  **Create Cluster:**
    ```bash
    kind create cluster --name basic-mlflow-cluster
    ```
    ***Note:***
    * Make sure kind is installed (kind version).
    * Ensure Docker is running, since kind uses Docker containers to simulate Kubernetes nodes.

3.  **Install with Helm:**
    ```bash
    helm repo add community-charts https://community-charts.github.io/helm-charts
    helm repo update
    helm install mlflow community-charts/mlflow
    ```
4.  **Get the Pods:**
    ```bash
    kubectl get pods
    ```

5.  **Access UI:**
    ```bash
    kubectl port-forward pod/<pod-name> 7006:5000 --address 0.0.0.0
    ```
6.  **Delete Cluster:**
    ```
    kind delete cluster --name basic-mlflow-cluster
    ```

---

## MLflow Production Setup (PostgreSQL + Kubernetes)

In production, we use a **Stateful Architecture**.

### Why PostgreSQL?
Pods in Kubernetes are **ephemeral**. If the pod hosting a local SQLite DB crashes, all experiment history is lost. RDS (AWS) or managed Postgres provides persistent, robust storage with backups. Equivalent services on Azure (Azure Database for PostgreSQL) or GCP (Cloud SQL) can also be used.

### Implementation Steps:
1.  **RDS Setup:** Create a PostgreSQL instance in AWS. Enable **Public Access** if connecting from outside the VPC.
2.  **Connect to RDS:**
    ```bash
    psql -h <RDS-ENDPOINT> -p 5432 -U <USERNAME>
    ```
3.  **DB Configuration:**
    ```sql
    CREATE DATABASE mlflow;
    CREATE USER mlflow_user WITH PASSWORD 'mlflow_password';
    GRANT ALL PRIVILEGES ON DATABASE mlflow TO mlflow_user;
    GRANT ALL PRIVILEGES ON SCHEMA public TO mlflow_user;
    ```
4.  **Deploy MLflow Server:**
    ```bash
    # Create Cluster
    kind create cluster --name production-mlflow-cluster
    
    # Create the namespace
    kubectl create ns mlflow

    # Install using Helm
    helm install mlflow community-charts/mlflow \
      --namespace mlflow \
      --set backend.store.postgres.host=<RDS_ENDPOINT> \
      --set backend.store.postgres.user=mlflow_user \
      --set backend.store.postgres.password=mlflow_password \
      --set backend.store.postgres.db=mlflow

    # Get the Pods
    kubectl get pods -n mlflow -w

    #Access UI
    kubectl -n mlflow port-forward pod/<pod-name> 7006:5000 --address 0.0.0.0
    
    ```
    ***Note:*** The Helm chart uses an **Init Container** to verify the DB connection before the server starts.
    ```bash
    kubectl edit pod/<pod-name> -n mlflow
    ```

---

## Real-time Tracking with MLflow (DVC + MLflow)

### 1. Connecting from Terminal (Python CLI)
```python
import mlflow
mlflow.set_tracking_uri("http://localhost:7006")
mlflow.set_experiment("wine-prediction-mlops")
```

### 2. Full Workflow in `train.py`
Integrated with DVC for data versioning:
1.  `git clone` and `dvc pull` (fetch the data).
    ```bash
    # Clone the repo and switch to it
    git clone https://github.com/tagore8661/wine-prediction-mlops
    cd wine-prediction-mlops
    ```
2.  **Instrumentation in Code:**
    ```python
    import mlflow

    mlflow.set_tracking_uri("http://localhost:7006")
    mlflow.set_experiment("wine-prediction-mlops")

    with mlflow.start_run(run_name="Run_v1"):
        # Log Hyperparameters
        mlflow.log_param("test_size", 0.2)
        mlflow.log_param("alpha", 0.5)
        
        # ... Training Logic ...
        
        # Log Metrics
        mlflow.log_metric("rmse", 0.32)
        mlflow.log_metric("r2", 0.85)

        # Log Model Artifact
        mlflow.log_artifact("artifacts/model.pkl")
    ```

### 3. Difference between Params and Metrics:
*   **log_param:** Used for inputs/config (e.g., `test_size`, `algorithm`). Appears in the "Parameters" box.
*   **log_metric:** Used for training results (e.g., `accuracy`, `loss`). Appears in the "Metrics" box and can be plotted.

---

## Comparing Runs Visually

The MLflow UI allows management and engineers to:
*   Select multiple runs and click **"Compare"**.
*   View **Box Plots** or **Scatter Plots** to identify trends.
*   Verify the exact **Git Commit** linked to a specific run.
*   Check stored **Artifacts** (the actual produced model).
*   See **duration** and **system specs** for each execution.

---

## MLflow Quick Revision

1.  **Problem:** Manual tracking (Excel) is unreliable and decentralized.
2.  **Solution:** MLflow automates tracking of everything (Params, Metrics, Artifacts, Code/Data version).
3.  **Deployment:** MLOps engineers deploy a **Production (Postgres/RDS)** instance.
4.  **Usage:** Data Scientists use the Python module to **log** information.
5.  **Benefit:** Centralized UI for comparison and 100% reproducibility of ML experiments.
6.  **Security:** Can be integrated with **SSO** for organizational access.

---

### Resources
- GitHub Reference: [Wine Prediction MLOps](https://github.com/tagore8661/wine-prediction-mlops)
- Documentation: [MLflow Basic Installation Doc](https://community-charts.github.io/docs/charts/mlflow/basic-installation)
- Documentation: [MLflow PostgreSQL Backend Installation](https://community-charts.github.io/docs/charts/mlflow/postgresql-backend-installation)
- Official Documentation: [MLflow Community Charts](https://community-charts.github.io/helm-charts)