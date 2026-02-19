# Jenkins CI/CD Setup for EKS Deployment

Complete guide to set up Jenkins pipeline for deploying the Task Management System to AWS EKS.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Jenkins Pipeline                             │
│                                                                      │
│  1. Checkout Code → 2. Build Images → 3. Push to ECR               │
│  4. Update Kubeconfig → 5. Deploy to EKS → 6. Verify               │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          AWS Cloud                                   │
│  ┌────────────────┐    ┌──────────────┐    ┌──────────────────┐   │
│  │   Amazon ECR   │    │  Amazon EKS  │    │  Application     │   │
│  │  (Container    │───▶│  (Kubernetes │───▶│  Load Balancer   │   │
│  │   Registry)    │    │   Cluster)   │    │     (ALB)        │   │
│  └────────────────┘    └──────────────┘    └──────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

### 1. Jenkins Server Requirements
- Jenkins 2.x or higher
- Minimum 4GB RAM, 2 CPU cores
- Docker installed on Jenkins server
- kubectl installed on Jenkins server
- AWS CLI installed on Jenkins server

### 2. Required Jenkins Plugins
Install these plugins from Jenkins → Manage Jenkins → Plugin Manager:

```
- Docker Pipeline
- Kubernetes CLI
- AWS Credentials
- Pipeline
- Git
- Credentials Binding
```

### 3. AWS Resources
- EKS cluster created
- ECR repositories created
- IAM user/role with permissions for:
  - ECR (push/pull images)
  - EKS (describe cluster, update kubeconfig)
  - Route53 (optional, for DNS)

## Step-by-Step Setup

### Step 1: Install Jenkins

#### Option A: Docker (Quick Setup)
```bash
# Create Jenkins volume
docker volume create jenkins-data

# Run Jenkins container
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins-data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

#### Option B: EC2 Instance
```bash
# Update system
sudo yum update -y

# Install Java
sudo yum install java-11-amazon-corretto -y

# Add Jenkins repo
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key

# Install Jenkins
sudo yum install jenkins -y

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Get initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Access Jenkins at: `http://your-server-ip:8080`

### Step 2: Install Required Tools on Jenkins Server

```bash
# Install Docker
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker jenkins

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Restart Jenkins to apply group changes
sudo systemctl restart jenkins
```

### Step 3: Configure AWS Credentials in Jenkins

#### 3.1 Create IAM User for Jenkins
```bash
# Create IAM user
aws iam create-user --user-name jenkins-deployer

# Attach policies
aws iam attach-user-policy --user-name jenkins-deployer \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser

aws iam attach-user-policy --user-name jenkins-deployer \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# Create access key
aws iam create-access-key --user-name jenkins-deployer
```

#### 3.2 Add Credentials to Jenkins
1. Go to Jenkins → Manage Jenkins → Credentials
2. Click "Global" → "Add Credentials"
3. Add AWS credentials:
   - **Kind**: AWS Credentials
   - **ID**: `aws-credentials`
   - **Access Key ID**: Your AWS access key
   - **Secret Access Key**: Your AWS secret key
   - **Description**: AWS Credentials for EKS Deployment

4. Add AWS Account ID:
   - **Kind**: Secret text
   - **Secret**: Your AWS Account ID (e.g., 796796207873)
   - **ID**: `aws-account-id`
   - **Description**: AWS Account ID

### Step 4: Create EKS Cluster (if not exists)

```bash
export AWS_REGION=ap-south-1
export CLUSTER_NAME=raghu-eks-cluster
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create cluster
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed

# Install AWS Load Balancer Controller
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --region $AWS_REGION --approve

curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Step 5: Create ECR Repositories

```bash
export AWS_REGION=ap-south-1

# Create repositories
aws ecr create-repository --repository-name task-management/frontend --region $AWS_REGION
aws ecr create-repository --repository-name task-management/user-service --region $AWS_REGION
aws ecr create-repository --repository-name task-management/task-service --region $AWS_REGION
aws ecr create-repository --repository-name task-management/notification-service --region $AWS_REGION
```

### Step 6: Configure Jenkins Pipeline

#### 6.1 Create New Pipeline Job
1. Jenkins Dashboard → New Item
2. Enter name: `task-management-eks-deploy`
3. Select "Pipeline"
4. Click OK

#### 6.2 Configure Pipeline
1. **General**:
   - Description: "Deploy Task Management System to EKS"
   - Check "GitHub project" (optional)
   - Project URL: Your GitHub repo URL

2. **Build Triggers**:
   - Check "GitHub hook trigger for GITScm polling" (for auto-deploy on push)
   - Or check "Poll SCM" with schedule: `H/5 * * * *` (every 5 minutes)

3. **Pipeline**:
   - Definition: "Pipeline script from SCM"
   - SCM: Git
   - Repository URL: Your GitHub repo URL
   - Credentials: Add GitHub credentials if private repo
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`

4. Click "Save"

### Step 7: Update Jenkinsfile Variables

Edit the `Jenkinsfile` and update these values:

```groovy
environment {
    AWS_REGION = 'ap-south-1'              // Your AWS region
    EKS_CLUSTER_NAME = 'raghu-eks-cluster' // Your EKS cluster name
}
```

### Step 8: Configure AWS CLI on Jenkins Server

```bash
# Switch to jenkins user
sudo su - jenkins

# Configure AWS CLI
aws configure
# Enter:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region: ap-south-1
# - Default output format: json

# Test AWS access
aws sts get-caller-identity
aws eks list-clusters --region ap-south-1

# Update kubeconfig
aws eks update-kubeconfig --name raghu-eks-cluster --region ap-south-1

# Test kubectl
kubectl get nodes
```

### Step 9: Run Your First Build

1. Go to your pipeline job
2. Click "Build Now"
3. Monitor the build in "Console Output"

The pipeline will:
1. ✅ Checkout code from Git
2. ✅ Build 4 Docker images in parallel
3. ✅ Push images to ECR
4. ✅ Update kubeconfig for EKS
5. ✅ Deploy to Kubernetes
6. ✅ Verify deployment

### Step 10: Set Up GitHub Webhook (Optional)

For automatic deployments on code push:

1. Go to your GitHub repo → Settings → Webhooks
2. Click "Add webhook"
3. Payload URL: `http://your-jenkins-url:8080/github-webhook/`
4. Content type: `application/json`
5. Select "Just the push event"
6. Click "Add webhook"

Now every push to main branch will trigger a deployment!

## Pipeline Stages Explained

### 1. Checkout
- Clones the Git repository
- Gets the commit hash for image tagging

### 2. Build Docker Images
- Builds all 4 services in parallel
- Tags with build number and commit hash
- Also tags as "latest"

### 3. Push to ECR
- Authenticates with ECR
- Pushes all images with both tags

### 4. Update Kubeconfig
- Configures kubectl to connect to EKS cluster

### 5. Deploy to EKS
- Updates image tags in k8s-deployment.yaml
- Applies the manifest
- Waits for rollout to complete

### 6. Verify Deployment
- Checks pod status
- Shows services and ingress

## Troubleshooting

### Issue: Docker permission denied
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Issue: kubectl not found
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Issue: AWS credentials not working
```bash
# Test as jenkins user
sudo su - jenkins
aws sts get-caller-identity
aws eks list-clusters --region ap-south-1
```

### Issue: ECR login failed
```bash
# Ensure IAM user has ECR permissions
aws ecr get-login-password --region ap-south-1
```

### Issue: kubectl can't connect to cluster
```bash
# Update kubeconfig
aws eks update-kubeconfig --name raghu-eks-cluster --region ap-south-1

# Test connection
kubectl get nodes
```

### View Jenkins Logs
```bash
# Docker installation
docker logs jenkins

# System installation
sudo journalctl -u jenkins -f
```

## Advanced Configuration

### Multi-Environment Deployment

Create separate pipelines for dev/staging/prod:

```groovy
parameters {
    choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Deployment environment')
}

environment {
    EKS_CLUSTER_NAME = "${params.ENVIRONMENT}-eks-cluster"
    NAMESPACE = "task-management-${params.ENVIRONMENT}"
}
```

### Rollback Strategy

Add a rollback stage:

```groovy
stage('Rollback') {
    when {
        expression { currentBuild.result == 'FAILURE' }
    }
    steps {
        script {
            sh """
                kubectl rollout undo deployment/frontend -n task-management
                kubectl rollout undo deployment/user-service -n task-management
                kubectl rollout undo deployment/task-service -n task-management
                kubectl rollout undo deployment/notification-service -n task-management
            """
        }
    }
}
```

### Slack Notifications

Add Slack notifications:

```groovy
post {
    success {
        slackSend(color: 'good', message: "Deployment successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
    }
    failure {
        slackSend(color: 'danger', message: "Deployment failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
    }
}
```

## Security Best Practices

1. **Use Jenkins Credentials**: Never hardcode secrets
2. **Limit IAM Permissions**: Use least privilege principle
3. **Enable RBAC**: Configure Kubernetes RBAC
4. **Scan Images**: Add security scanning stage
5. **Use Private Repos**: Keep code in private repositories
6. **Enable HTTPS**: Use SSL for Jenkins UI
7. **Regular Updates**: Keep Jenkins and plugins updated

## Monitoring

### View Deployment Status
```bash
# Check pods
kubectl get pods -n task-management

# Check services
kubectl get svc -n task-management

# Check ingress
kubectl get ingress -n task-management

# View logs
kubectl logs -f deployment/user-service -n task-management
```

### Jenkins Build History
- View all builds in Jenkins UI
- Check console output for each build
- Monitor build trends and success rate

## Cost Optimization

- Use spot instances for Jenkins server
- Stop Jenkins when not in use
- Use smaller node types for dev/staging
- Clean up old Docker images regularly

## Quick Commands

```bash
# Trigger build from CLI
curl -X POST http://jenkins-url:8080/job/task-management-eks-deploy/build \
  --user username:api-token

# Check cluster status
kubectl get all -n task-management

# View recent deployments
kubectl rollout history deployment/user-service -n task-management

# Scale deployment
kubectl scale deployment user-service --replicas=3 -n task-management
```

## Next Steps

1. ✅ Set up Jenkins server
2. ✅ Configure AWS credentials
3. ✅ Create EKS cluster
4. ✅ Create ECR repositories
5. ✅ Configure pipeline
6. ✅ Run first deployment
7. ✅ Set up GitHub webhook
8. ✅ Monitor and maintain

Your Jenkins CI/CD pipeline is now ready to deploy to EKS automatically!
