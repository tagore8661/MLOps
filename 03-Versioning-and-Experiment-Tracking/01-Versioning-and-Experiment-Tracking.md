# Data Versioning and Data Version Control (DVC)

## Table of Contents
1. [What is Data Versioning - Why Git isn't enough?](#what-is-data-versioning---why-git-isnt-enough)
2. [Introduction to DVC (Data Version Control)](#introduction-to-dvc-data-version-control)
3. [DVC Realtime Hands-on [DVC + AWS S3]](#dvc-realtime-hands-on-dvc--aws-s3)

---

## What is Data Versioning - Why Git isn't enough?

Machine Learning projects rely on two core components: **Scripts (Algorithms)** and **Data**.

### Why Git works for Scripts:
*   **Auditing:** Know who changed what.
*   **Versioning:** Compare versions and roll back if needed.
*   **RBAC:** Control who has access.

### Why Git fails for Data:
1.  **Size Limits:** Data can be huge (GBs or TBs - e.g., weather data or image datasets). Git is not designed for large binary files.
2.  **Performance:** Git becomes extremely slow when multiple people push large files.
3.  **Cost:** Self-hosting Git with large storage is expensive (e.g., EBS volumes on AWS). Git storage isn't as cost-effective as specialized cloud storage.

**Conclusion:** We need a system that offers Git-like versioning but is optimized for large data files. That system is **DVC**.

---

## Introduction to DVC (Data Version Control)

DVC acts as a bridge between your code (in Git) and your data (in cloud storage).

### How it works:
*   **Storage:** DVC stores actual data sets or models in remote cloud storage (AWS S3, Azure Blob, Google Cloud Storage).
*   **Pointers:** Instead of the data itself, Git stores `.dvc` files. These are tiny text files containing a unique **Check/Hash** of the data.
*   **Workflow:** 
    *   `dvc add data.csv`: Starts tracking the file and creates `data.csv.dvc`.
    *   `dvc push`: Uploads the actual data to the cloud (S3).
    *   `git add data.csv.dvc`: Commits the "pointer" to Git.

### Advantages of DVC:
*   **Endless Storage:** Cloud storage (S3) is virtually infinite.
*   **Cost-Effective:** Cloud storage is much cheaper than standard disk storage.
*   **Durability:** 99.999999999% durability (files are never lost).
*   **Collaboration:** Colleagues pull the code from Git, see the `.dvc` pointers, and run `dvc pull` to fetch the exact dataset version.

---

## DVC Realtime Hands-on [DVC + AWS S3]

### A. Environment Setup
Always use a virtual environment to manage dependencies.
```bash
# Clone the repo and switch to it
git clone https://github.com/tagore8661/wine-prediction-mlops
cd wine-prediction-mlops

# Setup virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows use: .venv\Scripts\activate

# Install DVC
pip install dvc dvc-s3
```

### B. Initialize Repositories
```bash
git init
dvc init
# Commits DVC config files to Git
git commit -m "Initialize DVC"
```

### C. Tracking Data with DVC
1.  **Create data:** Assume you have `data/wine_sample.csv`.
2.  **Add to DVC:**
    ```bash
    dvc add data/wine_sample.csv
    ```
    *This creates `data/wine_sample.csv.dvc` and adds the actual CSV to `.gitignore` automatically.*
3.  **Inspect metadata:** Running `cat data/wine_sample.csv.dvc` will show a unique MD5 checksum.

### D. Configuring Remote Storage (AWS S3)
1.  **Create an S3 Bucket** in the AWS Console (e.g., `my-dvc-bucket`).
2.  **Configure AWS CLI:** Run `aws configure` and provide your Access Key and Secret Key.
3.  **Add Remote to DVC:**
    ```bash
    dvc remote add -d wine-remote s3://my-dvc-bucket/data
    git commit .dvc/config -m "Configure S3 remote"
    ```

### E. The DVC Workflow (Push/Pull)
1.  **Push data to S3:**
    ```bash
    dvc push
    ```
2.  **Commit metadata to Git:**
    ```bash
    git add data/wine_sample.csv.dvc
    git commit -m "Add first version of dataset"
    ```
3.  **Updating Data:**
    *   Modify the CSV file.
    *   Run `dvc add data/wine_sample.csv` (updates the `.dvc` hash).
    *   Run `git add data/wine_sample.csv.dvc`
    *   Run `git commit -m "Update dataset to v2"` 
    *   Run `dvc push` (uploads new version to S3).


### F. Collaboration
When a teammate joins:
1.  `git clone <repo-url>`
2.  `dvc pull` (Fetches the specific version of the data matching the hash currently in the `.dvc` file).

**Key Takeaway:** The Git repository is the "Source of Truth" for which version of the data should be used, while S3 holds the actual large files.

---

### Resources
- GitHub Reference: [Wine Prediction MLOps](https://github.com/tagore8661/wine-prediction-mlops)