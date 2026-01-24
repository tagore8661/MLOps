# Introduction to MLOps

## Table of Contents
1. [What is Machine Learning and What is a Model?](#what-is-machine-learning-and-what-is-a-model)
2. [How is a Model Created in Realtime (Detailed Steps)](#how-is-a-model-created-in-realtime-detailed-steps)
3. [What is MLOps?](#what-is-mlops)
4. [Machine Learning Lifecycle Overview](#machine-learning-lifecycle-overview)
5. [Data Scientist vs. ML Engineer vs. MLOps Engineer](#data-scientist-vs-ml-engineer-vs-mlops-engineer)

---

## What is Machine Learning and What is a Model?

**What is machine learning?**
In simple words, machine learning is a process of training a machine or computer with a huge amount of data so that it identifies patterns and predicts output.

**Example:**
Imagine you want to build a system that can predict the type of a flower. You provide the system with four input parameters:
*   Length of a petal
*   Width of the petal
*   Length of the sepal
*   Width of the sepal

Using these parameters, your system should predict the type of flower.

In the general programming world (without machine learning), you would identify a programming language and write multiple `if-else` conditions.
*   *Example:* `if length of petal is 4 and width of sepal is 6 -> Rose`
*   *Example:* `if width of petal is 6 and length of sepal is 2 -> Jasmine`

**The Problem with Traditional Programming:**
*   You end up writing endless `if-else` conditions because no two flowers (even of the same type) are exactly the same.
*   Your conditions will go out of control.
*   If a user gives inputs not handled by your conditions, the program will error out.

**The Machine Learning Approach:**
In machine learning, data scientists identify a dataset (huge amount of data) and an algorithm. They train the algorithm with the dataset.
*   The algorithm identifies patterns (some invisible to humans) and develops a mathematical function.
*   This **mathematical function** is called a **Model**.

**What is a Model?**
A model is the output of the machine learning process. It is basically a mathematical function capable of making predictions based on inputs.
*   It is *not* the data.
*   It is *not* the algorithm.
*   It is the function derived from training the algorithm on the data.

---

## How is a Model Created in Realtime (Detailed Steps)

**Scenario:** An organization wants to build a system to predict the type of a flower.

### Step 1: Data Collection
Data scientists start with a **dataset**. A dataset contains information about different features and the corresponding label (type of flower).
*   *Example Entry:* `4 (Petal Length), 3 (Petal Width), 5 (Sepal Length), 6 (Sepal Width) -> Jasmine`
*   Data scientists collect this huge amount of data (often from free datasets or internal sources).

### Step 2: Data Splitting
They split the dataset, typically following an **80:20 ratio**:
*   **80%** for **Training**: Used to train the algorithm.
*   **20%** for **Testing**: Used to test the model's accuracy.

### Step 3: Algorithm Selection
Data scientists choose an appropriate algorithm. Popular ones include:
*   Logistic Regression (LR)
*   Decision Tree (Good for this specific flower problem)
*   K-Nearest Neighbor (KNN)

*Note: MLOps engineers usually don't decide this; it's the Data Scientist's call.*

### Step 4: Model Training
The algorithm is trained on the 80% training data.
*   The model learns patterns (e.g., "If sepal length and width are small -> Rose").
*   It develops a mathematical function (the Model).

### Step 5: Model Testing
The model is tested using the remaining 20% of the data.
*   This ensures the model returns accurate results on data it hasn't seen before.

### Step 6: Retraining (if necessary)
If the model performs poorly (wrong results), data scientists must:
*   Retrain the model.
*   Change the dataset.
*   Or change the algorithm.

### Step 7: Packaging/Saving the Model
Once successful, the model is saved/packaged. Common formats:
*   `.pkl` (Pickle) - Very popular
*   `joblib`
*   `onnx`

### Step 8: Deployment/Integration
Software developers or ML engineers develop an API for the model or package it into a software application (web or mobile) so end users can consume it.

---

## What is MLOps?

**MLOps** stands for **Machine Learning Operations**.
*   In simple words: **MLOps is DevOps for Machine Learning.**
*   **DevOps** - It is a set of best practices that unify model development and operations to ship models safer, faster, and more reliably.

### Context: DevOps vs. Traditional Development
Without DevOps, development and operations are manual and siloed:
*   Developers write code -> Build locally -> Test locally -> Hand over to Ops.
*   Ops deploy manually (Server setup, Load Balancer, API Gateway, etc.).
*   **Problem:** Slow, error-prone, takes multiple iterations.

**With DevOps:**
*   **CI/CD:** Automated pipelines build, test, and deploy code whenever changes are made.
*   **IaC:** Infrastructure is automated.
*   **Result:** Faster software delivery.

### The Need for MLOps
Similarly, building and shipping ML models is tedious without MLOps.
**Without MLOps:**
1.  Data Scientists collect/prepare data.
2.  Develop/Evaluate model manually.
3.  ML Engineers build APIs and test locally.
4.  Ops team deploys manually (Containerization, Kubernetes, etc.).
5.  **Problem:** Models need many iterations (10-20 to get right efficiency). Doing this manually 20 times is extremely slow.

**With MLOps:**
*   Automated CI/CD pipelines for models (Retraining, Testing).
*   Infrastructure as Code (IaC) for ML infrastructure.
*   Automated Observability and Monitoring.
*   **Result:** Significantly enhanced Machine Learning Lifecycle.

### Can DevOps and MLOps Coexist?
**Yes.**
*   **DevOps** handles the traditional software application (e.g., Netflix UI, Payment Microservice).
*   **MLOps** handles the machine learning models (e.g., Netflix Recommendation Engine, PayPal Fraud Detection).

---

## Machine Learning Lifecycle Overview

The complete journey from collecting raw data to monitoring the model in production.

1.  **Problem Definition:** Understanding the business problem in detail.
2.  **Data Collection:** Gathering data from various sources (Databases, APIs, Logs, Public datasets like Iris).
3.  **Data Cleaning:** Removing noise, duplicates, and fixing errors in the gathered data.
4.  **Feature Engineering:** Adding new features to existing data to help the model learn better (e.g., creating a "Total Area" feature from length/width).
5.  **Model Selection:** Choosing the right algorithm (Linear Regression, Decision Tree, Neural Networks, etc.).
6.  **Model Training:** Training the algorithm with the dataset to create the mathematical function.
7.  **Model Evaluation:** Testing accuracy (90%? 95%?). If low, retrain (Loop back to previous steps).
8.  **Hyperparameter Tuning:** Fine-tuning model parameters (Advanced optimization).
9.  **Model Deployment:** Deploying the model via APIs or embedded in apps so users can utilize it.
10. **Model Monitoring & Maintenance:** Monitoring performance in production. If accuracy drops (Data Drift), trigger retraining.

---

## Data Scientist vs. ML Engineer vs. MLOps Engineer

### Data Scientist
*   **Focus:** Core PL/Algorithms/Math.
*   **Responsibilities:**
    *   Understand problem definition.
    *   Collect and Clean data.
    *   Feature Engineering.
    *   Select Algorithm/Model.
    *   Train and Evaluate the model.
*   **Output:** A trained model (often local).

### ML Engineer
*   **Focus:** Productionizing the model.
*   **Responsibilities:**
    *   Take the model from Data Scientists.
    *   Optimize for performance, scalability, latency, memory.
    *   Develop APIs for the model.
    *   Integrate model into backend systems/software applications.
    *   (In startups) Sometimes deploy and monitor (Manual).

### MLOps Engineer
*   **Focus:** Automation & Infrastructure (Shipping models faster and safer).
*   **Responsibilities:**
    *   Introduce MLOps practices (CI/CD for ML).
    *   Automate manual activities of Data Scientists (Reproducible training pipelines).
    *   Automate manual activities of ML Engineers/Ops (Continuous Deployment).
    *   Set up Model Registries.
    *   Manage Infrastructure (Infrastructure as Code, GPU instances).
    *   Kubernetes Management & Cost Optimization.
    *   Set up Observability/Alerts for model performance.

**Summary:** MLOps engineers enable Data Scientists and ML Engineers to focus on their core tasks by handling the operational complexity and automation.
