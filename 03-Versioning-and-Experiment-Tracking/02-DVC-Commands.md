# DVC (Data Version Control) - Commands

## Definition
**Data Version Control (DVC)** is an open-source version control system specifically designed for Machine Learning projects. It allows you to version your datasets and models just like you version your source code with Git. 

Instead of storing large files in Git (which causes performance issues), DVC stores them in **remote cloud storage** (S3, GCS, Azure, etc.) and keeps tiny pointer files (`.dvc`) in Git to track which version of the data belongs to which version of the code.

---

## Table of Contents
1. [Initialization](#initialization)
2. [Data Tracking](#data-tracking)
3. [Remote Storage Configuration](#remote-storage-configuration)
4. [Data Synchronization](#data-synchronization)
5. [Collaboration & Retrieval](#collaboration--retrieval)
6. [Pipeline & Reproducibility](#pipeline--reproducibility)
7. [Experiments (Tracking)](#experiments-tracking)
8. [Inspection & Utilities](#inspection--utilities)

---

## Initialization
Setting up DVC in your project.

| Command | Description | Example |
| :--- | :--- | :--- |
| `dvc init` | Initializes DVC in the current directory (must be a Git repo). | `dvc init` |
| `dvc init --subdir` | Initializes DVC in a specific subdirectory. | `dvc init --subdir ml_project` |
| `dvc destroy` | Removes all DVC configuration and data tracking. | `dvc destroy` |

## Data Tracking
Managing datasets and models.

| Command | Description | Example |
| :--- | :--- | :--- |
| `dvc add <path>` | Starts tracking a file or directory. Creates a `.dvc` file. | `dvc add data/raw_data.csv` |
| `dvc remove <file>.dvc` | Stops tracking the data (does not delete the actual data). | `dvc remove data/raw_data.csv.dvc` |
| `dvc checkout` | Restores data files to the version specified in `.dvc` files. | `dvc checkout` |
| `dvc commit` | Updates the data in the cache to match the workspace. | `dvc commit` |
| `dvc move <src> <dst>` | Renames/Moves a DVC-tracked file and its `.dvc` file. | `dvc move data.csv data/v1.csv` |

## Remote Storage Configuration
Connecting DVC to the cloud.

| Command | Description | Example |
| :--- | :--- | :--- |
| `dvc remote add -d <name> <url>` | Adds a remote and sets it as default (`-d`). | `dvc remote add -d storage s3://my-bucket/data` |
| `dvc remote list` | Lists all configured remote storages. | `dvc remote list` |
| `dvc remote remove <name>` | Removes a configured remote. | `dvc remote remove storage` |
| `dvc remote modify <name> <param> <val>` | Modifies remote parameters (like credentials/endpoints). | `dvc remote modify storage endpointurl http://minio:9000` |

## Data Synchronization
Moving data between local and remote.

| Command | Description | Example |
| :--- | :--- | :--- |
| `dvc push` | Uploads tracked data from local cache to remote storage. | `dvc push` |
| `dvc pull` | Downloads tracked data from remote storage to workspace. | `dvc pull` |
| `dvc fetch` | Downloads data from remote to cache only. | `dvc fetch` |
| `dvc status` | Shows differences between local data and remote/cache. | `dvc status` |
| `dvc install` | Installs Git hooks to automate DVC actions on Git commands. | `dvc install` |

## Collaboration & Retrieval
Working with others and external projects.

| Command | Description | Example |
| :--- | :--- | :--- |
| `dvc get <url> <path>` | Downloads a file/folder from a repo WITHOUT tracking it. | `dvc get https://github.com/example/repo data.csv` |
| `dvc import <url> <path>` | Downloads data from a repo and TRACKS it in your project. | `dvc import https://github.com/example/repo data.csv` |
| `dvc pull -r <remote>` | Pulls data from a specific named remote. | `dvc pull -r s3_remote` |
| `dvc import-url <url> <path>` | Tracks data from a direct URL (S3, HTTP, etc.) | `dvc import-url s3://bucket/data.zip data/` |

## Pipeline & Reproducibility
Managing the ML workflow (Stages).

| Command | Description | Example |
| :--- | :--- | :--- |
| `dvc run -n <name> -d <deps> -o <outs> <cmd>` | Defines a pipeline stage (deps, output, and command). | `dvc run -n train -d train.py -d data.csv -o model.pkl python train.py` |
| `dvc repro` | Runs the pipeline stages whose dependencies changed. | `dvc repro` |
| `dvc dag` | Visualizes the pipeline as a graph in the terminal. | `dvc dag` |
| `dvc metrics show` | Displays tracking metrics (from JSON/CSV/etc). | `dvc metrics show` |
| `dvc plots show` | Generates HTML visualization of logs/plots. | `dvc plots show` |

## Experiments (Tracking)
Newer features for tracking ML experiments.

| Command | Description | Example |
| :--- | :--- | :--- |
| `dvc exp run` | Runs an experiment and tracks params/metrics. | `dvc exp run` |
| `dvc exp show` | Lists all experiments and their results in a table. | `dvc exp show` |
| `dvc exp diff` | Compares metrics/params between experiments. | `dvc exp diff` |

## Inspection & Utilities
Checking the state of things.

| Command | Description | Example |
| :--- | :--- | :--- |
| `dvc list <url>` | Lists files/folders in a DVC-tracked repository. | `dvc list https://github.com/example/repo` |
| `dvc diff` | Shows changes in data files between Git/DVC states. | `dvc diff HEAD~1` |
| `dvc gc` | Garbage Collection: Deletes unused data (local/remote). | `dvc gc -w -c` |
| `dvc config` | Get or set DVC configuration options. | `dvc config core.autostage true` |
| `dvc root` | Returns the absolute path to the project root. | `dvc root` |

---

## Quick Workflow Summary
1. `git init` & `dvc init`
2. `dvc remote add -d myremote s3://my-bucket/data`
3. `dvc add data/raw_data.csv`
4. `git add data/raw_data.csv.dvc .gitignore`
5. `git commit -m "Add raw data tracking"`
6. `dvc push`
7. (Colleague) `git pull` & `dvc pull`