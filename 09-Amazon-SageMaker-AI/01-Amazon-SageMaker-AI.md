# Amazon SageMaker AI

## Table of Contents
1. [Introduction to Amazon SageMaker AI](#introduction-to-amazon-sagemaker-ai)
2. [Getting Started with Amazon SageMaker AI](#getting-started-with-amazon-sagemaker-ai)
3. [Amazon SageMaker Production Setup - Part 1](#amazon-sagemaker-production-setup---part-1)
4. [Amazon SageMaker Production Setup - Part 2](#amazon-sagemaker-production-setup---part-2)
5. [Create and Save Models to Registry using SageMaker](#create-and-save-models-to-registry-using-sagemaker)
6. [Deploy and Serve Model for Inference using SageMaker](#deploy-and-serve-model-for-inference-using-sagemaker)
7. [Delete Resources [Important]](#delete-resources-important)

---

## Introduction to Amazon SageMaker AI

### Rebranding Note
- Amazon SageMaker has been rebranded as **Amazon SageMaker AI**
- Old documentation may refer to "Amazon SageMaker"
- Latest documentation uses "Amazon SageMaker AI"
- Both refer to the same platform

### What is Amazon SageMaker AI?
- **Unified platform for implementing machine learning lifecycle**
- End-to-end ML platform (not just MLOps)
- Supports entire ML workflow from model creation to deployment and inference

### Key Components
#### SageMaker Studio
- Central hub for ML activities
- Organizations can set up **domains** (typically one domain per team)
- Within domains, multiple applications are available:
  - **Jupyter Notebooks** - For data scientists
  - **Canvas** - For low-code ML
  - **Pipelines** - For CI/CD workflows
  - **Model Serving** - For deployment and inference

### Advantages of SageMaker AI

#### Pros
1. **Single Platform** - Common platform for all team members improves collaboration
2. **Fully Managed** - Infrastructure managed by AWS
   - No need to create infrastructure for training, pipelines, etc.
3. **Reduced Learning Curve** - Data scientists and ML engineers only need to learn one platform

#### Cons
1. **Cost** - Not open source, involves:
   - SageMaker service pricing
   - AWS infrastructure pricing
   - Maintenance costs
2. **Vendor Locking** - Difficult to migrate to open-source alternatives (Kubeflow, MLflow, etc.)
3. **MLOps Complexity** - Implementing RBAC, cost control, and governance can be tedious

---

## Getting Started with Amazon SageMaker AI

### Accessing SageMaker AI
1. Go to AWS Console
2. Search for "SageMaker AI" (rebranded version)
3. Navigate to domain and user profile setup

### Key Concepts

#### Domains
- **Purpose**: Represent teams within an organization
- **Setup**: Created by MLOps engineers
- **Examples**: Payment team, transaction team, UI team

#### User Profiles
- **Purpose**: Individual user accounts within domains
- **Access**: Provide access to SageMaker Studio applications
- **Management**: Configured per user role and requirements

### Domain Creation Options

#### Quick Start (Single User)
- **Best for**: Research scientists, freelancers, individuals
- **Features**: Cost-effective, simple setup
- **Limitations**: Lacks enterprise features

#### Enterprise Setup
- **Best for**: Organizations, teams
- **Features**:
  - Custom IAM role capabilities
  - SageMaker Studio interface
  - VPC networking and security groups
  - MLflow integration
- **Recommended**: For production environments

### Authentication Methods
1. **AWS Identity Center** - For organizations with existing Identity Center setup
2. **IAM Authentication** - Recommended for mid-scale companies, provides granular RBAC

### Studio Applications
- **JupyterLab** - For data scientists
- **RStudio** - For R developers
- **Canvas** - Low-code ML platform
- **Pipelines** - CI/CD workflows
- **Model Registry** - Model versioning and storage
- **MLflow** - Experiment tracking

---

## Amazon SageMaker Production Setup - Part 1

### Real-World Setup Example
**Organization**: example.com
**Models**: 
1. User traffic model (traffic prediction)
2. Cost prediction model (for end users)

### Three-Step Setup Process

#### Step 1: Domain Creation
**Responsibility**: MLOps Engineers
**Method**: Infrastructure as Code (IAC) or AWS CLI
**Considerations**:
- Number of domains based on teams/models
- Create IAM role with SageMaker Full Access
- Assign role to domain for resource access

#### Step 2: User Profile Creation
**Process**:
- Analyze team structure
- Create user profiles per team member
- Configure application access per role:
  - **Data Scientists**: Jupyter notebooks access
  - **ML Engineers**: Model packaging, container creation
  - **MLOps Engineers**: Pipelines, model serving access
- Implement proper access controls

#### Step 3: Attribute-Based Access Control (ABAC)
**Purpose**: Prevent users from accessing other user profiles
**Implementation**:
1. **Tag User Profiles**: Add tags to user profiles (e.g., key: "studiouserid", value: "alice123")
2. **Configure IAM Policies**: Add conditions to IAM user policies
3. **Enforce Access**: Users can only access resources with matching tags

**Example ABAC Policy Structure**:
```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["sagemaker:*"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "sagemaker:ResourceTag/studiouserid": "${aws:PrincipalTag/studiouserid}"
        }
      }
    }
  ]
}
```

---

## Amazon SageMaker Production Setup - Part 2

### Prerequisites
- AWS CLI installed and configured
- Default VPC and subnet information
- Appropriate IAM permissions

### Step-by-Step Implementation

#### 1. Prepare Network Information
```bash
# Get default VPC
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text \
  --region <REGION>

# Get subnets
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<DEFAULT_VPC_ID>" \
  --query "Subnets[].SubnetId" \
  --output text \
  --region <REGION>
```

#### 2. Create IAM Role for Domain
**Trust Policy** (trust.json):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "sagemaker.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Commands**:
```bash
# Create role
aws iam create-role \
  --role-name SageMakerDomainExecutionRole \
  --assume-role-policy-document file://trust.json

# Attach permissions
aws iam attach-role-policy \
  --role-name SageMakerDomainExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
```

#### 3. Create Domain
```bash
aws sagemaker create-domain \
  --domain-name mlops-demo-domain \
  --auth-mode IAM \
  --vpc-id <VPC_ID> \
  --subnet-ids <SUBNET_ID_1> <SUBNET_ID_2> \
  --app-network-access-type VpcOnly \
  --default-user-settings "{
      \"ExecutionRole\": \"<ROLE_ARN>\"
   }" \
  --region <REGION>
```
It will Returns:
`{
     "DomainArn": "<DOMAIN_ARN>",
     "DomainId: "<DOMAIN_ID",
     "Url": "<SAGEMAKER_URL>"
 }`
 
*NOTE:* Creation will take Time 2-3 min, so don't move to the next steps, instead go to AWS Console and Search `Amazon SageMaker` and wiat for the Domain Status is `InService`.

#### 4. Create User Profile with Tags
```bash
aws sagemaker create-user-profile \
  --domain-id <DOMAIN_ID> \
  --user-profile-name alice-profile \
  --tags Key=studiouserid,Value=alice123 \
  --region <REGION>
```
It will Returns:
`{
     "UserProfileArn": "<USER_PROFILE_ARN>"
 }`

#### 5. Create IAM User and Tag
```bash
# Create IAM user
aws iam create-user --user-name alice-iam-user

# Tag IAM user
aws iam tag-user --user-name alice-iam-user --tags Key=studiouserid,Value=alice123
```

#### 6. Create ABAC Policy
**Policy Document** (sagemaker-abac.json):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowConsoleListAndDescribe",
      "Effect": "Allow",
      "Action": [
        "sagemaker:ListDomains",
        "sagemaker:ListUserProfiles",
        "sagemaker:ListApps",
        "sagemaker:DescribeDomain",
        "sagemaker:DescribeUserProfile",
        "sagemaker:ListTags"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowPresignedUrlWhenTagMatches",
      "Effect": "Allow",
      "Action": [
        "sagemaker:CreatePresignedDomainUrl"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "sagemaker:ResourceTag/studiouserid": "${aws:PrincipalTag/studiouserid}"
        }
      }
    }
  ]
}
```

#### 7. Create and Attach Policy
```bash
# Create policy
aws iam create-policy \
  --policy-name SageMaker-Studio-ABAC \
  --policy-document file://sagemaker-abac.json

# Attach to user
aws iam attach-user-policy \
  --user-name alice-iam-user \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/SageMaker-Studio-ABAC
```

#### 8. Generate Pre-signed URL (Optional)
```bash
aws sagemaker create-presigned-domain-url \
  --domain-id <DOMAIN_ID> \
  --user-profile-name alice-profile \
  --session-expiration-duration-in-seconds 3600 \
  --region ap-south-1
```

### Verification Process
1. Log in as the IAM user
2. Navigate to SageMaker AI
3. Verify access to own user profile ✓
4. Verify denied access to other user profiles ✗

---

## Create and Save Models to Registry using SageMaker

### Workflow Overview
1. **Create S3 Bucket** - Model registry
2. **Set up SageMaker Environment** - Infrastructure
3. **Create Domain & User Profile** - Access management
4. **Use Jupyter Notebooks** - Model development
5. **Push Model to Registry** - Storage

### Step-by-Step Implementation

#### 1. Create S3 Bucket (Model Registry)
```bash
aws s3 mb s3://<UNIQUE-BUCKET-NAME>
```

#### 2. Set up SageMaker Environment
```bash
# Create IAM Execution Role for SageMaker
# Create trust policy - trust.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "sagemaker.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}

---

# Create the role
aws iam create-role \
  --role-name SageMakerDemoExecutionRole \
  --assume-role-policy-document file://trust.json

---

# Attach permissions
# i.SageMakerFullAccess
aws iam attach-role-policy \
  --role-name SageMakerDemoExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess

# ii.S3FullAccess
aws iam attach-role-policy \
  --role-name SageMakerDemoExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

---

# Find default VPC + default Subnets
aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" --output text

aws ec2 describe-subnets --filters Name=vpc-id,Values=<VPC_ID> \
  --query "Subnets[*].SubnetId" --output text

```

#### 3. Create Domain & User Profile
```
# Create SageMaker Domain (Studio)
aws sagemaker create-domain \
  --domain-name demo-domain \
  --auth-mode IAM \
  --default-user-settings "ExecutionRole=arn:aws:iam::<ACCOUNT-ID>:role/SageMakerDemoExecutionRole" \
  --vpc-id <VPC_ID> \
  --subnet-ids "<SUBNET_1>" "<SUBNET_2>"

# Create User Profile
aws sagemaker create-user-profile \
  --domain-id <DOMAIN_ID> \
  --user-profile-name demo-user
```

#### 4. Use Jupyter Notebooks

Go to:

`AWS Console → SageMaker → Domains → demo-domain → Launch app → Studio → JupyterLab`

***NOTE:*** We are Not Creating the `IAM User`, Not Creating the `ABAC`. So We Can Directly access the Studio from Admin User.

##### Jupyter Notebook Workflow
1. Launch SageMaker Studio
2. Navigate to Jupyter Lab
3. Select instance type (Quick Start for basic models)
4. Launch instance now

##### Model Creation Example (Iris Dataset)
##### Create iris.py
```python
# Import dependencies
import boto3
import joblib
from sklearn.datasets import load_iris
from sklearn.tree import DecisionTreeClassifier
import os

# Load and prepare data
iris = load_iris()
X, y = iris.data, iris.target

# Create and train model
model = DecisionTreeClassifier()
model.fit(X, y)

# Save model locally
joblib.dump(model, "iris-model.pkl")

print("Model saved as iris-model.pkl")

# Upload to S3 (Model Registry)
s3_client = boto3.client('s3')
bucket = '<UNIQUE-BUCKET-NAME>'
s3_client.upload_file('iris_model.pkl', bucket, 'model-artifacts/iris_model.pkl')

print("Uploaded to S3:", f"s3://{bucket}/model-artifacts/iris-model.pkl")

```

### Key Points for MLOps Engineers
- Focus on model registry storage, not model creation details
- Ensure data scientists push models to S3 registry
- Help with S3 upload code if needed
- Model registry enables deployment and serving workflows

---

## Deploy and Serve Model for Inference using SageMaker

### SageMaker Deployment Requirements
- **Format**: `.tar.gz` (not `.pkl`)
- **Inference Script**: `inference.py` required
- **Container**: Pre-configured container with dependencies

### Prerequisites Setup

#### 1. Create inference.py
```python
import joblib
import os
import json

def model_fn(model_dir):
    """Load model from model directory"""
    model = joblib.load(os.path.join(model_dir, "iris-model.pkl"))
    return model


def input_fn(request_body, request_content_type):
    """Process input data"""
    data = json.loads(request_body)
    return data["instances"]


def predict_fn(input_data, model):
    """Make predictions"""
    return model.predict(input_data)


def output_fn(prediction, content_type):
    """Format output"""
    return json.dumps({"predictions": prediction.tolist()})
```

#### 2. Create .tar.gz File
```python
import tarfile

# Create tar.gz file
with tarfile.open("model.tar.gz", "w:gz") as tar:
    tar.add("iris-model.pkl")
    tar.add("inference.py")

print("model.tar.gz created")
```

#### 3. Upload model.tar.gz to S3
Add new Cell in `iris.py` and Run this..
```python
s3.upload_file(
    "model.tar.gz",
    bucket,
    "model-artifacts/model.tar.gz"
)

print("Packaged model uploaded to S3")
```

### Deployment Methods

#### Method 1: Python Script
##### Create deploy.py
```python
import sagemaker
from sagemaker.sklearn.model import SKLearnModel

# Create model
model = SKLearnModel(
    model_data="s3://{BUCKET_NAME}/model-artifacts/model.tar.gz",
    role='<sagemaker-execution-role-arn>',
    entry_point='inference.py',
    framework_version='1.2' #Use Version matching your local environment
)


# Deploy model
predictor = model.deploy(
    initial_instance_count=1,
    instance_type='ml.t2.medium'
)
```
Once if you run this Kernel went to the Busy State b/c in Deployments → Endpoints → #where we see that Endpoint is Creating and Within Endpoint we see a Model is Run

Once if Status is Available, by using the the `URL` or `Test inference` we can Access the Model..

#### Method 2: AWS Console
1. Navigate to SageMaker → Models → Deployable Models
2. Click "Create Model"
3. Configure:
   - **Name**: demo-sage-model
   - **Container type**: Pre-built container
   - **Container Framework**: Scikit-learn #Based on Framework
   - **Framework Version**: Latest(1.2)
   - **Hardware type**: cpu
   - **S3 URI**: s3://{BUCKET_NAME}/model-artifacts/model.tar.gz
   - **IAM Role**: <sagemaker-execution-role-arn>
4. Click "Create Deployable Model"
5. Click "Deploy" → Select instance type, initial instance count, Maximum instance count → Deploy

### Model Serving and Inference

#### Testing via Console
1. Navigate to Endpoints
2. Select deployed endpoint
3. Use "Test Inference"
4. Provide JSON input:
```json
{ 
  "instances": [[5.1, 3.5, 1.4, 0.2]]
}
```

#### Architecture Flow
```
End User → API Gateway → SageMaker Endpoint → EC2 Instance → Container → Model
```

### Security Enhancements (Production)
- **API Gateway** - For external access
- **Load Balancer** - For high availability
- **Lambda** - For request preprocessing (optional)

---

## Delete Resources [Important]

**Always delete SageMaker resources to avoid high costs!**

### Deletion Order (Important)
1. **Stop/Delete Spaces** (Jupyter Lab instances)
2. **Delete User Profiles**
3. **Delete Domain**

### Step-by-Step Cleanup

#### 1. Delete Spaces/Applications
- Navigate to SageMaker → Domain → Resources (Stop running instances)
- Domain → Space Management (Delete all spaces)

#### 2. Delete User Profiles
- Navigate to User Profiles section
- Delete all user profiles individually

#### 3. Delete Domain
- Only available after spaces and profiles are deleted
- Navigate to Domain → Delete Domain
- Confirm deletion

### CLI Alternative
```bash
# Delete user profile
aws sagemaker delete-user-profile --domain-id <domain-id> --user-profile-name <profile-name>

# Delete domain
aws sagemaker delete-domain --domain-id <domain-id>
```

---

## Quick Reference Commands

### Domain Management
```bash
# Create domain
aws sagemaker create-domain --domain-name <name> --auth-mode IAM --vpc-id <vpc-id> --subnet-ids <subnet-ids> --domain-execution-role-arn <role-arn>

# List domains
aws sagemaker list-domains

# Delete domain
aws sagemaker delete-domain --domain-id <domain-id>
```

### User Profile Management
```bash
# Create user profile
aws sagemaker create-user-profile --domain-id <domain-id> --user-profile-name <name> --tags Key=<key>,Value=<value>

# List user profiles
aws sagemaker list-user-profiles --domain-id <domain-id>

# Delete user profile
aws sagemaker delete-user-profile --domain-id <domain-id> --user-profile-name <name>
```

### Model Deployment
```bash
# Create model
aws sagemaker create-model --model-name <name> --primary-container ContainerImage=<image>,ModelDataUrl=<s3-url>,Environment=<env-vars> --execution-role-arn <role-arn>

# Create endpoint config
aws sagemaker create-endpoint-config --endpoint-config-name <name> --production-variants VariantName=<variant>,ModelName=<model-name>,InstanceType=<instance-type>,InitialInstanceCount=<count>

# Create endpoint
aws sagemaker create-endpoint --endpoint-name <name> --endpoint-config-name <config-name>

# Delete endpoint
aws sagemaker delete-endpoint --endpoint-name <name>
```
