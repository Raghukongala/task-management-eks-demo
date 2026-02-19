# Jenkins Server on AWS EC2 - Complete Setup Guide

Step-by-step guide to launch and configure Jenkins on an EC2 instance in your AWS account.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Account                              │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │                    VPC (Default or Custom)              │    │
│  │                                                         │    │
│  │  ┌──────────────────────────────────────────────┐     │    │
│  │  │         Public Subnet                         │     │    │
│  │  │                                               │     │    │
│  │  │  ┌─────────────────────────────────────┐    │     │    │
│  │  │  │   EC2 Instance (Jenkins Server)     │    │     │    │
│  │  │  │   - t3.medium (2 vCPU, 4GB RAM)     │    │     │    │
│  │  │  │   - Amazon Linux 2023               │    │     │    │
│  │  │  │   - Jenkins + Docker + kubectl      │    │     │    │
│  │  │  │   - Port 8080 (Jenkins UI)          │    │     │    │
│  │  │  │   - IAM Role attached               │    │     │    │
│  │  │  └─────────────────────────────────────┘    │     │    │
│  │  │                    │                         │     │    │
│  │  │         ┌──────────┴──────────┐             │     │    │
│  │  │         │   Security Group    │             │     │    │
│  │  │         │   - SSH (22)        │             │     │    │
│  │  │         │   - HTTP (8080)     │             │     │    │
│  │  │         └─────────────────────┘             │     │    │
│  │  └──────────────────────────────────────────────┘     │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              IAM Role (JenkinsEC2Role)                 │    │
│  │  - ECR Full Access                                     │    │
│  │  - EKS Describe Cluster                                │    │
│  │  - S3 (for artifacts - optional)                       │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              ECR Repositories                          │    │
│  │  - task-management/frontend                            │    │
│  │  - task-management/user-service                        │    │
│  │  - task-management/task-service                        │    │
│  │  - task-management/notification-service                │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              EKS Cluster                               │    │
│  │  - raghu-eks-cluster                                   │    │
│  │  - Managed node groups                                 │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- AWS Account with admin access
- AWS CLI configured on your local machine
- SSH key pair (or create new one)
- Basic knowledge of AWS EC2 and security groups

## Step 1: Create IAM Role for Jenkins EC2

### 1.1 Create IAM Policy for Jenkins

```bash
# Set your AWS region
export AWS_REGION=ap-south-1

# Create policy document
cat > jenkins-ec2-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:DescribeRepositories",
        "ecr:ListImages",
        "ecr:DescribeImages"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::jenkins-artifacts-*",
        "arn:aws:s3:::jenkins-artifacts-*/*"
      ]
    }
  ]
}
EOF

# Create IAM policy
aws iam create-policy \
  --policy-name JenkinsEC2Policy \
  --policy-document file://jenkins-ec2-policy.json \
  --description "Policy for Jenkins EC2 to access ECR and EKS"

# Get your account ID
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account ID: $ACCOUNT_ID"
```

### 1.2 Create IAM Role

```bash
# Create trust policy
cat > jenkins-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create IAM role
aws iam create-role \
  --role-name JenkinsEC2Role \
  --assume-role-policy-document file://jenkins-trust-policy.json \
  --description "IAM role for Jenkins EC2 instance"

# Attach policy to role
aws iam attach-role-policy \
  --role-name JenkinsEC2Role \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/JenkinsEC2Policy

# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name JenkinsEC2InstanceProfile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name JenkinsEC2InstanceProfile \
  --role-name JenkinsEC2Role

echo "IAM Role created: JenkinsEC2Role"
```

## Step 2: Create Security Group

```bash
# Get default VPC ID
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text \
  --region $AWS_REGION)

echo "VPC ID: $VPC_ID"

# Create security group
export SG_ID=$(aws ec2 create-security-group \
  --group-name jenkins-server-sg \
  --description "Security group for Jenkins server" \
  --vpc-id $VPC_ID \
  --region $AWS_REGION \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SG_ID"

# Get your public IP (for SSH access)
export MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "Your IP: $MY_IP"

# Allow SSH from your IP
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr ${MY_IP}/32 \
  --region $AWS_REGION

# Allow Jenkins UI from anywhere (or restrict to your IP)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 8080 \
  --cidr 0.0.0.0/0 \
  --region $AWS_REGION

# Allow HTTPS (optional, for future SSL setup)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0 \
  --region $AWS_REGION

echo "Security group configured!"
```

## Step 3: Create or Use Existing Key Pair

### Option A: Create New Key Pair
```bash
# Create new key pair
aws ec2 create-key-pair \
  --key-name jenkins-key \
  --query 'KeyMaterial' \
  --output text \
  --region $AWS_REGION > jenkins-key.pem

# Set permissions
chmod 400 jenkins-key.pem

export KEY_NAME=jenkins-key
echo "Key pair created: jenkins-key.pem"
```

### Option B: Use Existing Key Pair
```bash
# List existing key pairs
aws ec2 describe-key-pairs --region $AWS_REGION

# Set your key name
export KEY_NAME=your-existing-key-name
```

## Step 4: Create User Data Script

```bash
cat > jenkins-userdata.sh <<'USERDATA'
#!/bin/bash
set -e

# Update system
yum update -y

# Install Java 17 (required for Jenkins)
yum install java-17-amazon-corretto -y

# Add Jenkins repository
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

# Install Jenkins
yum install jenkins -y

# Start Jenkins
systemctl start jenkins
systemctl enable jenkins

# Install Docker
yum install docker -y
systemctl start docker
systemctl enable docker

# Add jenkins user to docker group
usermod -aG docker jenkins

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
yum install unzip -y
unzip awscliv2.zip
./aws/install
rm -rf aws awscliv2.zip

# Install git
yum install git -y

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl /usr/local/bin

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Restart Jenkins to apply group changes
systemctl restart jenkins

# Wait for Jenkins to start
sleep 30

# Get initial admin password
JENKINS_PASSWORD=$(cat /var/lib/jenkins/secrets/initialAdminPassword)

# Create info file
cat > /home/ec2-user/jenkins-info.txt <<EOF
Jenkins Installation Complete!

Jenkins URL: http://$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4):8080
Initial Admin Password: $JENKINS_PASSWORD

Tools Installed:
- Jenkins (latest)
- Docker (latest)
- kubectl (latest)
- AWS CLI v2
- eksctl (latest)
- Helm 3
- Git

Next Steps:
1. Access Jenkins UI at the URL above
2. Use the initial admin password to unlock Jenkins
3. Install suggested plugins
4. Create admin user
5. Configure Jenkins for EKS deployment

IAM Role: JenkinsEC2Role (already attached)
Security Group: jenkins-server-sg
EOF

echo "Setup complete! Check /home/ec2-user/jenkins-info.txt for details"
USERDATA

echo "User data script created!"
```

## Step 5: Launch EC2 Instance

```bash
# Get latest Amazon Linux 2023 AMI
export AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text \
  --region $AWS_REGION)

echo "AMI ID: $AMI_ID"

# Launch EC2 instance
export INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.medium \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=JenkinsEC2InstanceProfile \
  --user-data file://jenkins-userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Jenkins-Server}]' \
  --block-device-mappings 'DeviceName=/dev/xvda,Ebs={VolumeSize=30,VolumeType=gp3}' \
  --region $AWS_REGION \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance ID: $INSTANCE_ID"
echo "Waiting for instance to start..."

# Wait for instance to be running
aws ec2 wait instance-running \
  --instance-ids $INSTANCE_ID \
  --region $AWS_REGION

# Get public IP
export PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text \
  --region $AWS_REGION)

echo ""
echo "=========================================="
echo "Jenkins Server Launched Successfully!"
echo "=========================================="
echo "Instance ID: $INSTANCE_ID"
echo "Public IP: $PUBLIC_IP"
echo "Jenkins URL: http://${PUBLIC_IP}:8080"
echo ""
echo "Wait 3-5 minutes for Jenkins to install..."
echo "=========================================="

# Save details to file
cat > jenkins-server-details.txt <<EOF
Jenkins Server Details
======================
Instance ID: $INSTANCE_ID
Public IP: $PUBLIC_IP
Jenkins URL: http://${PUBLIC_IP}:8080
Region: $AWS_REGION
Key Name: $KEY_NAME
Security Group: $SG_ID
IAM Role: JenkinsEC2Role

SSH Command:
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP}

To get initial admin password:
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP} "sudo cat /var/lib/jenkins/secrets/initialAdminPassword"
EOF

echo "Details saved to jenkins-server-details.txt"
```

## Step 6: Access Jenkins Server

### 6.1 Wait for Installation
```bash
# Wait 3-5 minutes for user data script to complete
echo "Waiting for Jenkins installation (this takes 3-5 minutes)..."
sleep 180

# Check if Jenkins is running
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP} "sudo systemctl status jenkins"
```

### 6.2 Get Initial Admin Password
```bash
# Get initial password
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP} "sudo cat /var/lib/jenkins/secrets/initialAdminPassword"

# Or check the info file
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP} "cat /home/ec2-user/jenkins-info.txt"
```

### 6.3 Access Jenkins UI
1. Open browser: `http://YOUR_PUBLIC_IP:8080`
2. Paste the initial admin password
3. Click "Continue"

## Step 7: Configure Jenkins

### 7.1 Install Plugins
1. Select "Install suggested plugins"
2. Wait for plugins to install
3. Additional plugins to install:
   - Docker Pipeline
   - Kubernetes CLI
   - Slack Notification
   - Git
   - Pipeline

### 7.2 Create Admin User
1. Fill in admin user details:
   - Username: `admin`
   - Password: (your secure password)
   - Full name: Your name
   - Email: your-email@example.com
2. Click "Save and Continue"

### 7.3 Configure Jenkins URL
1. Jenkins URL should be: `http://YOUR_PUBLIC_IP:8080/`
2. Click "Save and Finish"
3. Click "Start using Jenkins"

## Step 8: Configure AWS and Kubernetes

### 8.1 SSH into Jenkins Server
```bash
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP}
```

### 8.2 Switch to Jenkins User
```bash
sudo su - jenkins
```

### 8.3 Configure AWS CLI (Uses IAM Role - No credentials needed!)
```bash
# Test AWS access (should work automatically via IAM role)
aws sts get-caller-identity
aws eks list-clusters --region ap-south-1

# Configure default region
aws configure set default.region ap-south-1
```

### 8.4 Configure kubectl for EKS
```bash
# Update kubeconfig for your EKS cluster
aws eks update-kubeconfig --name raghu-eks-cluster --region ap-south-1

# Test kubectl
kubectl get nodes
kubectl get namespaces
```

### 8.5 Test Docker
```bash
# Test Docker access
docker ps

# Test ECR login
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com
```

## Step 9: Create Jenkins Pipeline Job

### 9.1 Create New Pipeline
1. Jenkins Dashboard → New Item
2. Name: `task-management-eks-deploy`
3. Type: Pipeline
4. Click OK

### 9.2 Configure Pipeline
1. **General**:
   - Description: "Deploy Task Management System to EKS"
   - GitHub project URL: Your repo URL

2. **Build Triggers**:
   - Poll SCM: `H/5 * * * *` (every 5 minutes)

3. **Pipeline**:
   - Definition: "Pipeline script from SCM"
   - SCM: Git
   - Repository URL: Your GitHub repo
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`

4. Click "Save"

### 9.3 Add AWS Account ID Credential
1. Jenkins → Manage Jenkins → Credentials
2. Global → Add Credentials
3. Kind: Secret text
4. Secret: Your AWS Account ID (e.g., 796796207873)
5. ID: `aws-account-id`
6. Click "Create"

## Step 10: Run First Build

### 10.1 Trigger Build
1. Go to your pipeline job
2. Click "Build Now"
3. Watch "Console Output"

### 10.2 Monitor Build
```bash
# SSH to Jenkins server
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP}

# Watch Jenkins logs
sudo tail -f /var/log/jenkins/jenkins.log

# Check Docker images
sudo docker images

# Check kubectl
sudo su - jenkins
kubectl get pods -n task-management
```

## Step 11: Set Up Elastic IP (Optional but Recommended)

```bash
# Allocate Elastic IP
export EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --region $AWS_REGION \
  --query 'AllocationId' \
  --output text)

echo "Elastic IP Allocation ID: $EIP_ALLOC"

# Associate with instance
aws ec2 associate-address \
  --instance-id $INSTANCE_ID \
  --allocation-id $EIP_ALLOC \
  --region $AWS_REGION

# Get new public IP
export NEW_PUBLIC_IP=$(aws ec2 describe-addresses \
  --allocation-ids $EIP_ALLOC \
  --query 'Addresses[0].PublicIp' \
  --output text \
  --region $AWS_REGION)

echo "New Jenkins URL: http://${NEW_PUBLIC_IP}:8080"
```

## Troubleshooting

### Issue: Can't access Jenkins UI

**Check 1: Instance is running**
```bash
aws ec2 describe-instances --instance-ids $INSTANCE_ID --region $AWS_REGION
```

**Check 2: Security group allows port 8080**
```bash
aws ec2 describe-security-groups --group-ids $SG_ID --region $AWS_REGION
```

**Check 3: Jenkins is running**
```bash
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP} "sudo systemctl status jenkins"
```

### Issue: Docker permission denied

```bash
# SSH to server
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP}

# Add jenkins to docker group
sudo usermod -aG docker jenkins

# Restart Jenkins
sudo systemctl restart jenkins
```

### Issue: kubectl not working

```bash
# SSH as jenkins user
sudo su - jenkins

# Update kubeconfig
aws eks update-kubeconfig --name raghu-eks-cluster --region ap-south-1

# Test
kubectl get nodes
```

### Issue: ECR login failed

```bash
# Check IAM role is attached
aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].IamInstanceProfile' \
  --region $AWS_REGION

# Test ECR access
aws ecr describe-repositories --region ap-south-1
```

## Monitoring and Maintenance

### Check Jenkins Status
```bash
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP} "sudo systemctl status jenkins"
```

### View Jenkins Logs
```bash
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP} "sudo journalctl -u jenkins -f"
```

### Restart Jenkins
```bash
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP} "sudo systemctl restart jenkins"
```

### Update Jenkins
```bash
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP}
sudo yum update jenkins -y
sudo systemctl restart jenkins
```

### Backup Jenkins
```bash
# Create backup of Jenkins home
ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP}
sudo tar -czf jenkins-backup-$(date +%Y%m%d).tar.gz /var/lib/jenkins/
```

## Cost Optimization

### Instance Costs (ap-south-1)
- **t3.medium**: ~$0.0416/hour (~$30/month)
- **30GB EBS gp3**: ~$2.40/month
- **Elastic IP**: Free when attached, $0.005/hour when not
- **Total**: ~$32-35/month

### Save Costs
```bash
# Stop instance when not in use
aws ec2 stop-instances --instance-ids $INSTANCE_ID --region $AWS_REGION

# Start instance when needed
aws ec2 start-instances --instance-ids $INSTANCE_ID --region $AWS_REGION

# Get new public IP after start (if not using Elastic IP)
aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text --region $AWS_REGION
```

## Cleanup

### Delete Everything
```bash
# Terminate instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID --region $AWS_REGION

# Release Elastic IP (if allocated)
aws ec2 release-address --allocation-id $EIP_ALLOC --region $AWS_REGION

# Delete security group (wait for instance to terminate first)
aws ec2 delete-security-group --group-id $SG_ID --region $AWS_REGION

# Delete IAM resources
aws iam remove-role-from-instance-profile \
  --instance-profile-name JenkinsEC2InstanceProfile \
  --role-name JenkinsEC2Role

aws iam delete-instance-profile \
  --instance-profile-name JenkinsEC2InstanceProfile

aws iam detach-role-policy \
  --role-name JenkinsEC2Role \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/JenkinsEC2Policy

aws iam delete-role --role-name JenkinsEC2Role

aws iam delete-policy \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/JenkinsEC2Policy

# Delete key pair (if created)
aws ec2 delete-key-pair --key-name jenkins-key --region $AWS_REGION
rm jenkins-key.pem
```

## Quick Reference

### Important Files
- Jenkins home: `/var/lib/jenkins/`
- Jenkins logs: `/var/log/jenkins/jenkins.log`
- Initial password: `/var/lib/jenkins/secrets/initialAdminPassword`
- User data log: `/var/log/cloud-init-output.log`

### Important Commands
```bash
# Jenkins service
sudo systemctl start|stop|restart|status jenkins

# Docker service
sudo systemctl start|stop|restart|status docker

# View user data execution log
sudo cat /var/log/cloud-init-output.log

# Switch to jenkins user
sudo su - jenkins

# Test AWS access
aws sts get-caller-identity

# Test kubectl
kubectl get nodes

# Test Docker
docker ps
```

## Next Steps

1. ✅ Launch EC2 instance with Jenkins
2. ✅ Access Jenkins UI
3. ✅ Install plugins
4. ✅ Create admin user
5. ✅ Configure AWS and kubectl
6. ✅ Create pipeline job
7. ✅ Run first deployment
8. ✅ Set up Slack notifications (see SLACK_INTEGRATION.md)
9. ✅ Set up GitHub webhook for auto-deploy

Your Jenkins server on EC2 is ready to deploy to EKS! 🚀
