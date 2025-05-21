# 🚀 SDA Capstone Project – Django Blog App on AWS

## 🌍 Project Overview

This project demonstrates the deployment of a scalable Django-based blog application on the AWS cloud using production-level architecture. It utilizes:

- ✅ EC2 (Auto Scaling Group)
- ✅ Application Load Balancer (ALB)
- ✅ Amazon RDS (MySQL)
- ✅ Amazon S3 (for media storage)
- ✅ AWS VPC with public/private subnets
- ✅ IAM, NAT Gateway, Route Tables, SSM Parameters
- ✅ Infrastructure as Code via Terraform
- ✅ Monitoring with Prometheus & Grafana

---

## 🛠️ Step-by-Step Deployment Guide

### 🧱 Step 1: VPC and Networking Setup

🔧 Created a custom VPC `sda-capstone-vpc` with CIDR block `10.90.0.0/16`  
🔗 Enabled **DNS hostnames** to allow communication between instances  
🌐 Created 2 Public & 2 Private Subnets across `eu-north-1a` and `eu-north-1b`

#### Subnets:

- Public: `10.90.10.0/24`, `10.90.20.0/24`
- Private: `10.90.11.0/24`, `10.90.21.0/24`

✅ Auto-assign public IP enabled for public subnets  
✅ Internet Gateway created & attached  
✅ Route Tables created:
- Public RT connected to IGW (0.0.0.0/0)
- Private RT created separately for private subnets

---

### 🔐 Step 2: Security Groups

- **ALB SG** – Allows HTTP (80) & HTTPS (443) from anywhere  
- **EC2 SG** – Allows traffic only from ALB SG on 80/443 + SSH from anywhere  
- **RDS SG** – Accepts MySQL (3306) traffic only from EC2 SG  

✅ All rules scoped within `sda-capstone-vpc`  
✅ Security hardened by linking SGs instead of IPs

---

### 🗂 Step 3: GitHub Repository Setup

🔐 Created **private GitHub repo**: `capstone`  
🔑 Generated personal access token with `repo` scope  
📥 Cloned repo locally and pushed project files  
🧠 Token stored securely in AWS SSM for later use in EC2 startup script

Commands:

git clone https://<TOKEN>@github.com/<username>/capstone.git
cd capstone
git add .
git commit -m "initial commit"
git push

🛡 Step 4: SSM Parameter Store
Stored sensitive credentials securely in AWS SSM:

/myname/capstone/username → admin

/myname/capstone/password → Clarusway1234

/myname/capstone/token → <GitHub Token>

➡️ These values will be retrieved in settings.py using boto3

💾 Step 5: RDS – MySQL Database
Created a DB Subnet Group with both private subnets

Launched RDS MySQL 8.0 instance named sda-capstone-rds

Settings:

Storage: 20GB (auto-scaling up to 40GB)

Public Access: ❌

SG: sda-capstone-rds-sg

Initial DB: clarusway

DB credentials pulled from SSM inside Django settings

---

### 🪣 Step 6: Create S3 Bucket

🧾 Created an S3 bucket named: `sdacapstone-<yourname>-blog`  
📦 Purpose: store images and videos uploaded by users on the blog

- Region: Stockholm
- ACL: Bucket Owner Preferred
- ❗ Unchecked "Block all public access"
- Objects will be stored via Django’s S3 storage backend

➡️ Linked later in `settings.py` via:
--python
AWS_STORAGE_BUCKET_NAME = 'sdacapstone-<yourname>-blog'
AWS_S3_REGION_NAME = 'eu-north-1'

🌐 Step 7: Create NAT Gateway
💡 Why? EC2s in private subnets need internet access for updates & GitHub clone

Steps:

Allocated new Elastic IP

Created NAT Gateway in sda-capstone-public-subnet-1a

Updated Private Route Table with:

Destination: 0.0.0.0/0
Target: NAT Gateway
✅ Now private EC2 instances can access the internet securely 🚀

⚙️ Step 8: Update settings.py Configuration
To secure sensitive data, we dynamically pull credentials from SSM using boto3:

def get_ssm_parameters():
    ssm = boto3.client('ssm', region_name='eu-north-1')
    username = ssm.get_parameter(Name="/<yourname>/capstone/username", WithDecryption=False)['Parameter']['Value']
    password = ssm.get_parameter(Name="/<yourname>/capstone/password", WithDecryption=False)['Parameter']['Value']
    return username, password

✅ Also updated S3 settings, database host (from RDS), and secret key
✅ Pushed the changes to GitHub

🧪 Step 9: Test UserData Script on EC2
🎯 Created a temporary EC2 instance (Ubuntu 22.04) in public subnet
🔐 Attached IAM Role: sda-capstone-ec2-ssm-s3-full-access

✅ Ran and verified UserData script:
TOKEN=$(aws ssm get-parameter --name /<yourname>/capstone/token --with-decryption --query 'Parameter.Value' --output text)
git clone https://$TOKEN@github.com/<your-repo-name>/capstone.git
cd capstone/src
python3 manage.py migrate
python3 manage.py runserver 0.0.0.0:80

📸 Verified app at http://<Public-DNS>
✅ If success, test server terminated to avoid extra billing

🛡️ Step 10: Launch Admin Node (Monitoring + JumpBox)
🖥️ Created an EC2 instance: SDA-Admin-Node

OS: Amazon Linux 2023

Subnet: public

IAM Role: same as above

Installed:

Terraform

Ansible

Prometheus

Grafana

Ports Open: 22, 3000, 9090, 9100

✅ hostnamectl set-hostname SDA-Admin-Node

🧱 Step 11: Build Full Infrastructure with Terraform
🎯 Created Terraform configuration with the following modules:

main.tf, userdata.sh, variables.tf, sec-gr.tf, iam.tf, data.tf, outputs.tf

ALB, Target Group, Launch Template, ASG, DB Subnet Group, RDS, S3
Ran:
terraform init
terraform apply

📸 Verified full stack deployed on AWS
🌐 App accessible via ALB DNS
📝 Created blog posts successfully

⚙️ Step 12: Configure Ansible Dynamic Inventory
📦 On SDA-Admin-Node:

Created inventory_aws_ec2.yml

Added AWS region & EC2 filters by tags (Project: Capstone)

Configured ansible.cfg with PEM key

Test:
ansible-inventory --graph
✅ Able to discover EC2s dynamically using tags

🔧 Step 13: Deploy Prometheus Node Exporter via Ansible
🎯 Created Ansible playbook: prometheus_monitoring_setup.yaml
🛠 Installed Node Exporter on all EC2s automatically

ansible-playbook prometheus_monitoring_setup.yaml
✅ Service enabled and running on port 9100

📡 Step 14: Prometheus Service Discovery
Edited prometheus.yml to auto-discover EC2s using ec2_sd_configs:

- job_name: 'ec2-node-exporters'
  ec2_sd_configs:
    - region: eu-north-1
      port: 9100
      filters:
        - name: tag:App
          values: ["BlogPage"]
Ran Prometheus:
./prometheus --config.file=prometheus.yml

📸 Visited http://<Admin-IP>:9090/targets → EC2s detected! ✅

📊 Step 15: Grafana Integration & Dashboard
🎯 Opened http://<Admin-IP>:3000
🔐 Logged in: admin / admin

Steps:

Connected Prometheus as data source

Imported Dashboard ID 1860 (Node Exporter Full)

Saw real-time metrics: CPU, RAM, disk, uptime, network...

✅ Monitoring complete! 🚀

🧼 Clean-up Instructions
To destroy infrastructure:

cd sda-capstone-infra-tf
terraform destroy --auto-approve
Manually delete:

RDS

S3 Buckets (empty first)

Subnet Groups

IAM Roles

IGW & NAT Gateway

SSM Parameters

VPC

👑 Final Notes

🎓 This project was built as a capstone for Clarusway DevOps Bootcamp

🧑‍💻 Team 1
📆 Year: 2025


✅ All steps documented, executed, tested, and monitored successfully
💪 Proudly completed with passion, precision, and practical proof!
