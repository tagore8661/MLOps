
# Understand the Role of MLOps in ML Lifecycle at High Level (Project Based)

## Table of Contents

1. [Introduction to the Project](#introduction-to-the-project)
2. [Useful Links to get started](#useful-links-to-get-started)
3. [Learn How Data Scientists Work Without MLOps Practices](#learn-how-data-scientists-work-without-mlops-practices)
4. [Learn How MLOps Engineers Help Data Scientists](#learn-how-mlops-engineers-help-data-scientists)
5. [Role of ML Engineers in a Project (High Level)](#role-of-ml-engineers-in-a-project-high-level)
6. [Learn How MLOps Engineers Help ML Engineers with Model Deployment](#learn-how-mlops-engineers-help-ml-engineers-with-model-deployment)
7. [Conclusion](#conclusion)

---

## Introduction to the Project

Using a "Hello World" project, let's understand how MLOps engineers help Data Scientists and ML engineers in real-time.

**Business Requirement:**
Imagine an organization planning to build a system to predict flower species by taking the following inputs:
*   Petal length
*   Petal width
*   Sepal length
*   Sepal width

**Workflow Overview:**
1.  **Data Scientists:** Tackle the business requirement (create the model).
2.  **MLOps Engineers:** Help Data Scientists automate training.
3.  **ML Engineers:** Take the model to the next level (create APIs).
4.  **MLOps Engineers:** Help ML Engineers with deployment (Containerization).

---

## Useful Links to Get Started

For more information on the training and deployment workflow, please refer to the `iris-logreg-mlops` repository, which contains the training script, inference runner, CI/CD workflow, and Docker setup:
[https://github.com/tagore8661/iris-logreg-mlops](https://github.com/tagore8661/iris-logreg-mlops)

---

## Learn How Data Scientists Work Without MLOps Practices

### The Manual Process
1.  **Requirement Gathering:** clear understanding from product owners.
2.  **Data Collection:** Gathering data (e.g., Iris dataset).
3.  **Environment Setup:** Installing Python, setting up virtual environments (`venv`).
4.  **Scripting:** Writing Python scripts to load data, split it (train/test), choose an algorithm, train, and save the model.

### Practical Example: Iris Dataset
Data scientists often check data interactively:
```python
from sklearn.datasets import load_iris
import pandas as pd

iris = load_iris()
df = pd.DataFrame(iris.data, columns=iris.feature_names)
df['target'] = iris.target
print(df.head())
print(iris.target_names) # ['setosa', 'versicolor', 'virginica']
```

### The Training Script (`train.py`)
A typical script includes:
1.  Loading the dataset.
2.  Splitting data (80:20 ratio).
3.  Selecting an algorithm (e.g., Logistic Regression).
4.  Training the model.
5.  Saving the model (e.g., to `artifacts/model.pkl`).
6.  Generating metrics.

### Execution (Manual Steps)
Data Scientists manually execute these commands:

1.  **Create Virtual Environment:**
    ```bash
    python -m venv .venv
    source .venv/Scripts/activate
    ```
2.  **Install Dependencies:**
    ```bash
    pip install -r requirements.txt
    ```
3.  **Run Training:**
    ```bash
    python train.py
    ```
    *Result:* Model saved to `artifacts/model.pkl`. Accuracy printed.

4.  **Test Model (`run_model.py`):**
    ```bash
    python run_model.py --input "[5.1,3.5,1.4,0.2]"
    # Output: {"prediction": [0]} #Prediction is 0 (Setosa)
    ```

**Pain Points:**
*   Manual execution of commands every time code changes.
*   "It works on my machine" issues when sharing code.
*   Complex setups for multiple Python versions.

---

## Learn How MLOps Engineers Help Data Scientists

MLOps engineers automate these manual activities using **Version Control** and **CI/CD**.

### The Solution
1.  **Version Control (Git):** Centralized code management with branching strategies and RBAC.
2.  **CI/CD (GitHub Actions):** Automates testing and training on every change.

### Steps to Implement
1.  **Create Repository:** Push code (excluding `.venv`, etc., using `.gitignore`).
2.  **Define Workflow (`.github/workflows/iris-logreg-mlops-ci.yml`):**

**Workflow Content:**
*   **Trigger:** On `push` or `pull_request` to `main` branch.
*   **Jobs (Matrix Strategy):** Run on multiple Python versions (e.g., 3.11, 3.12, 3.13).
*   **Steps:**
    1.  **Checkout Code:** `actions/checkout@v4`
    2.  **Setup Python:** `actions/setup-python` (uses matrix version).
    3.  **Upgrade pip:** `python -m pip install --upgrade pip setuptools wheel`
    4.  **Install Dependencies:** `pip install -r requirements.txt`
    5.  **Train Model:** `python train.py`
    6.  **Upload Artifacts:** `actions/upload-artifact@v4` (saves `model.pkl`).

**Benefits:**
*   **Continuous Training (CT):** Every code change triggers training.
*   **Reproducibility:** Dependencies and environment are defined in code.
*   **Accessibility:** Artifacts (models) are automatically saved and downloadable.

---

## Role of ML Engineers in a Project (High Level)

Once the model is trained, it needs to be accessible to end-users (e.g., via Netflix app or website), not just downloadable as a file.

**Responsibilities:**
1.  **Scalability & Performance:** Optimizing the model.
2.  **API Development:** Wrapping the model in an API (Flask/FastAPI).

### Creating the API (`app.py`)
ML Engineers write an API to serve the model.

**Key Components of `app.py`:**
*   **Load Model:** Load `model.pkl` from artifacts.
*   **Define Route:** `@app.route('/predict', methods=['POST'])`
*   **Logic:**
    1.  Get JSON input (`sepal_length`, etc.).
    2.  Call `model.predict()`.
    3.  Return JSON output (`flower_type`).

**Running the API locally:**
```bash
python app.py
# Running on http://localhost:5000
```

**Testing the API:**
```bash
curl -X POST http://localhost:5000/predict \
     -H "Content-Type: application/json" \
     -d '{"features": [5.1, 3.5, 1.4, 0.2]}'
# Output: {"prediction": 0}
```

---

## Learn How MLOps Engineers Help ML Engineers with Model Deployment

Running `app.py` locally isn't enough for production. MLOps engineers help deploy the model reliably.

### Containerization (Docker)
The first step is to package the application and its dependencies into a Docker container so it runs anywhere.

**Dockerfile Example:**
```dockerfile
# Base Image
FROM python:3.12-slim

# Work Directory
WORKDIR /app

# Install Dependencies first (Caching layer)
COPY requirements.txt .
RUN python -m pip install --upgrade pip
RUN python -m pip install -r requirements.txt

# Copy Source Code
COPY . .

# Expose Port
EXPOSE 5000

# Command to run
CMD ["python", "app.py"]
```

### Build and Run
1.  **Build Image:**
    ```bash
    docker build -t iris-logreg-mlop:latest .
    ```
2.  **Run Container:**
    ```bash
    docker run -d -p 5000:5000 iris-logreg-mlop:latest
    ```
3.  **Verify:**
    Use the same `curl` command. It now hits the Docker container.

### Next Steps (Future topics)
After containerization, MLOps engineers handle:
*   **Kubernetes Manifests:** For orchestration.
*   **Infrastructure as Code:** Creating clusters/VMs.
*   **Deployment Strategies:** Load balancing, security, scaling.

---

## Conclusion

*   **Data Scientists:** Focus on data, algorithms, and training logic.
*   **MLOps Engineers (for DS):** Automate training pipelines (CI/CD) and manage artifacts.
*   **ML Engineers:** Turn models into usable software (APIs) and optimize performance.
*   **MLOps Engineers (for ML):** Handle containerization, infrastructure, and deployment to production (Kubernetes, etc.).

By understanding these roles, you see how MLOps bridges the gap between experimental code and production-grade AI systems.
