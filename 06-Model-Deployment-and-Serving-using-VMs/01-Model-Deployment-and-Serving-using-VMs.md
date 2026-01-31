# Model Deployment and Serving using Virtual Machines (VMs)

## Table of Contents
1. [Architecture & Problem Analysis](#architecture--problem-analysis)
2. [Implementing WSGI (Web Server Gateway Interface)](#implementing-wsgi-web-server-gateway-interface)
3. [The Userdata Script (Automated Configuration)](#the-userdata-script-automated-configuration)
4. [End-to-End Production Implementation](#end-to-end-production-implementation)
5. [Cleanup Resources](#cleanup-resources)

---

## Architecture & Problem Analysis

### Why avoid basic "python app.py" in Production?
The simple deployment used in development has three major flaws:
1.  **Development Server Limitations:** Flask/FastAPI development servers are not designed for concurrency. They struggle with multiple simultaneous users (a roadblock for apps like Netflix).
2.  **Foreground Processes:** Running a script in a terminal makes it a foreground process. If the terminal closes or the VM restarts, the API stops.
3.  **Static Scaling:** A single VM has fixed resources (e.g., 16GB RAM). It cannot handle dynamic traffic spikes (e.g., more users at 8 PM vs. 3 AM) without wasting money or crashing.

### The Production Solution (Architecture)
*   **VPC & Networking:** Custom subnets, internet gateways, and route tables for security.
*   **WSGI (Web Server Gateway Interface):** Using **Gunicorn** to handle parallelism and production-grade concurrency.
*   **Auto Scaling Group (ASG):** Automatically adds or removes VMs based on CPU/RAM usage (e.g., scale up if CPU > 80%).
*   **Load Balancer (ALB):** Distributes incoming traffic across multiple VMs created by the ASG (e.g., 1000 requests to VM1, 1000 to VM2).
*   **Reverse Proxy (Nginx):** Front-faces the API to handle SSL, rate limiting, and security (e.g., protecting against denial-of-service attacks).

### User Flow
```text
Internet (Client)
     |
Internet Gateway (IGW) - Attached to VPC
     |
Application Load Balancer (ALB) - Internet-facing (Listener: HTTP 80)
     |
Target Group (HTTP: 80, Health-check: /predict)
     |
Auto Scaling Group (ASG)
     |
EC2 Instance (in a Public Subnet) -> 
Nginx (listen: 80) - proxy_pass ->
Gunicorn (127.0.0.1:6000) -> WSGI App (/predict)
```

---

## Implementing WSGI (Web Server Gateway Interface)

WSGI allows your Python app to handle multiple concurrent requests efficiently.

### Create a EC2 Instance

### Using SSH Login to the Instance
```bash
ssh -i <.pem file Path> ubuntu@<PublicIP>
```

### Clone the repo and switch to it
```bash
git clone https://github.com/tagore8661/intent-classifier-mlops
cd intent-classifier-mlops
```

### Setup Virtual Environment
```bash
python3 -m venv .venv
source .venv/bin/activate
```

### Step 1: Create `wsgi.py`
This file acts as the entry point for the production server.
```python
from app import app
application = app
```

### Step 2: Update `requirements.txt`
Add **Gunicorn** to your dependencies:
```text
gunicorn
Flask
scikit-learn
joblib
pytest
```

### Step 3: Install Dependencies
```bash
python3 -m pip install -r requirements.txt
```

### Step 4: Before Execute the API we need the Model
```bash
python3 model/train.py
# When we run this it Creates the artifacts folder
```

### Step 5: Run with Gunicorn
Instead of `python app.py`, use:
```bash
gunicorn --workers 3 --bind 127.0.0.1:6000 wsgi:app
```
*   `--workers 3`: Enables parallelism (3 workers handling requests).
*   `--bind 127.0.0.1:6000`: Exposes the app on port 6000.

### Testing
Open a duplicate tab and then run the curl command:

```bash
# Test a Greeting
curl -X POST http://localhost:6000/predict \
     -H "Content-Type: application/json" \
     -d '{"text": "Hi, How are you?"}'
```

*Now this is not served by the Developer Server but this is served by the WSGI*

> **Note:** If you press `Ctrl + C`, the server will stop immediately, and subsequent requests will fail. This happens because Gunicorn is currently running as a foreground process. For a robust production setup, you should run it as a background task or manage it as a system service (e.g., using `systemd`) so it remains active even if the terminal session closes.

---

## The Userdata Script (Automated Configuration)

The **Userdata Script** is executed by the Auto Scaling Group every time a new VM is launched. It ensures the VM is "ready-to-serve" without manual intervention.

### `user_data.sh` content:
```bash
#!/bin/bash
set -e

# 1. Setup Directory
export APP_DIR="/opt/intent-app"
mkdir -p $APP_DIR
cd $APP_DIR

# 2. Update System & Install Dependencies
apt-get update -y
apt-get install -y git python3 python3-venv python3-pip nginx

# 3. Clone Code
git clone -b virtual-machines https://github.com/tagore8661/intent-classifier-mlops.git .

# 4. Setup Virtual Env & Install Python Packages
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

# 5. Train Model (Generate .pkl)
python3 model/train.py

# 6. Configure Gunicorn as a Systemd Service
cat > /etc/systemd/system/gunicorn.service <<'EOF'
[Unit]
Description=Gunicorn instance to serve Intent Classifier
After=network.target

[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/opt/intent-app
Environment="PATH=/usr/bin:/bin:/opt/intent-app/.venv/bin"
ExecStart=/opt/intent-app/.venv/bin/gunicorn --workers 3 --bind 127.0.0.1:6000 wsgi:app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 7. Configure Nginx as Reverse Proxy
cat > /etc/nginx/sites-available/default <<'EOF'
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:6000/predict;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_read_timeout 120s;
    }
}
EOF

# 8. Enable Nginx site
ln -sf /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default

# 9. Start Services
systemctl daemon-reload
systemctl enable gunicorn
systemctl restart gunicorn
systemctl restart nginx
```

---

## End-to-End Production Implementation

Executing these steps via **AWS Cloud Shell** ensures a consistent environment.

### Step 1: Networking Setup
1.  **Create VPC:** `aws ec2 create-vpc --cidr-block 10.1.0.0/16`
2.  **Create Subnets:** Create two subnets in different Availability Zones (for ALB requirements).
3.  **Internet Gateway:** Create and attach to VPC.
4.  **Route Table:** Create a route to `0.0.0.0/0` via the Internet Gateway and associate with subnets.

### Step 2: Security & Launch Template
1.  **Security Group:** Allow Port 80 (HTTP) and Port 22 (SSH).
2.  **Launch Template:** Define the AMI ID, Instance Type (t3.micro), and the **Base64 encoded** Userdata script.
    *   *Pro-tip:* Base64 encoding is used to avoid indentation/formatting issues when placing a shell script within a JSON or YAML template.

### Step 3: Load Balancer & Auto Scaling
1.  **Target Group:** Create a target group for Port 80.
2.  **Application Load Balancer (ALB):** Create the ALB and associate it with the public subnets.
3.  **Listener:** Add a listener to the ALB to forward traffic to the Target Group.
4.  **Auto Scaling Group (ASG):** Create the ASG using the Launch Template and attach it to the Target Group.

**Verification:** Send a request to the ALB DNS name:
```bash
curl -X POST http://<ALB-DNS-NAME>/predict \
     -H "Content-Type: application/json" \
     -d '{"text": "Hello, how are you?"}'
```

---

## Cleanup Resources

To avoid unexpected costs, delete resources in the reverse order of creation:
1.  **Auto Scaling Group (ASG):**
    *   Edit and disable target group integration.
    *   Scale down capacity (min/max/desired) to 0.
    *   Delete the ASG.
2.  **Load Balancer (ALB):** Delete the ALB and its listeners.
3.  **Target Group:** Delete the target group.
4.  **Launch Template:** Delete the template.
5.  **Security Group:** Delete the inbound rules/group.
6.  **Networking:** Detach Internet Gateway -> Delete Internet Gateway -> Delete Subnets -> Delete VPC.

---

### Resource Link
- **GitHub Reference:** [Intent Classifier MLOps](https://github.com/tagore8661/intent-classifier-mlops)
- **Branch:** [Virtual Machines Branch](https://github.com/tagore8661/intent-classifier-mlops/tree/virtual-machines)
