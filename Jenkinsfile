pipeline {
    agent any
    
    environment {
        AWS_REGION = 'ap-south-1'
        EKS_CLUSTER_NAME = 'raghuwitheks'
        DOCKER_BUILDKIT = '1'
        SLACK_CHANNEL = '#deployments'
        SLACK_CREDENTIAL_ID = 'slack-token'
    }
    
    stages {
        stage('Setup') {
            steps {
                script {
                    // Get AWS Account ID dynamically
                    env.AWS_ACCOUNT_ID = sh(
                        script: "aws sts get-caller-identity --query Account --output text",
                        returnStdout: true
                    ).trim()
                    env.ECR_REGISTRY = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                    
                    // Get git info if available
                    try {
                        env.GIT_COMMIT_SHORT = sh(
                            script: "git rev-parse --short HEAD 2>/dev/null || echo 'latest'",
                            returnStdout: true
                        ).trim()
                    } catch (Exception e) {
                        env.GIT_COMMIT_SHORT = 'latest'
                    }
                    
                    env.BUILD_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                    
                    echo "AWS Account ID: ${env.AWS_ACCOUNT_ID}"
                    echo "ECR Registry: ${env.ECR_REGISTRY}"
                    echo "Build Tag: ${env.BUILD_TAG}"
                }
            }
        }
        
        stage('Build Docker Images') {
            steps {
                echo "🔨 Building Docker images..."
            }
        }
        
        stage('Build All Services') {
            parallel {
                stage('Build Frontend') {
                    steps {
                        script {
                            sh """
                                docker build -t task-management/frontend:${BUILD_TAG} ./frontend
                                docker tag task-management/frontend:${BUILD_TAG} task-management/frontend:latest
                            """
                        }
                    }
                }
                stage('Build User Service') {
                    steps {
                        script {
                            sh """
                                docker build -t task-management/user-service:${BUILD_TAG} ./user-service
                                docker tag task-management/user-service:${BUILD_TAG} task-management/user-service:latest
                            """
                        }
                    }
                }
                stage('Build Task Service') {
                    steps {
                        script {
                            sh """
                                docker build -t task-management/task-service:${BUILD_TAG} ./task-service
                                docker tag task-management/task-service:${BUILD_TAG} task-management/task-service:latest
                            """
                        }
                    }
                }
                stage('Build Notification Service') {
                    steps {
                        script {
                            sh """
                                docker build -t task-management/notification-service:${BUILD_TAG} ./notification-service
                                docker tag task-management/notification-service:${BUILD_TAG} task-management/notification-service:latest
                            """
                        }
                    }
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    echo "📦 Pushing images to ECR..."
                    
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        
                        docker tag task-management/frontend:${BUILD_TAG} ${ECR_REGISTRY}/task-management/frontend:${BUILD_TAG}
                        docker tag task-management/frontend:latest ${ECR_REGISTRY}/task-management/frontend:latest
                        docker push ${ECR_REGISTRY}/task-management/frontend:${BUILD_TAG}
                        docker push ${ECR_REGISTRY}/task-management/frontend:latest
                        
                        docker tag task-management/user-service:${BUILD_TAG} ${ECR_REGISTRY}/task-management/user-service:${BUILD_TAG}
                        docker tag task-management/user-service:latest ${ECR_REGISTRY}/task-management/user-service:latest
                        docker push ${ECR_REGISTRY}/task-management/user-service:${BUILD_TAG}
                        docker push ${ECR_REGISTRY}/task-management/user-service:latest
                        
                        docker tag task-management/task-service:${BUILD_TAG} ${ECR_REGISTRY}/task-management/task-service:${BUILD_TAG}
                        docker tag task-management/task-service:latest ${ECR_REGISTRY}/task-management/task-service:latest
                        docker push ${ECR_REGISTRY}/task-management/task-service:${BUILD_TAG}
                        docker push ${ECR_REGISTRY}/task-management/task-service:latest
                        
                        docker tag task-management/notification-service:${BUILD_TAG} ${ECR_REGISTRY}/task-management/notification-service:${BUILD_TAG}
                        docker tag task-management/notification-service:latest ${ECR_REGISTRY}/task-management/notification-service:latest
                        docker push ${ECR_REGISTRY}/task-management/notification-service:${BUILD_TAG}
                        docker push ${ECR_REGISTRY}/task-management/notification-service:latest
                    """
                }
            }
        }
        
        stage('Update Kubeconfig') {
            steps {
                script {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                    """
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    echo "☸️ Deploying to EKS cluster: ${EKS_CLUSTER_NAME}..."
                    
                    sh """
                        # Update image tags in deployment
                        sed -i "s|image: .*task-management/frontend.*|image: ${ECR_REGISTRY}/task-management/frontend:${BUILD_TAG}|g" k8s-deployment.yaml
                        sed -i "s|image: .*task-management/user-service.*|image: ${ECR_REGISTRY}/task-management/user-service:${BUILD_TAG}|g" k8s-deployment.yaml
                        sed -i "s|image: .*task-management/task-service.*|image: ${ECR_REGISTRY}/task-management/task-service:${BUILD_TAG}|g" k8s-deployment.yaml
                        sed -i "s|image: .*task-management/notification-service.*|image: ${ECR_REGISTRY}/task-management/notification-service:${BUILD_TAG}|g" k8s-deployment.yaml
                        
                        # Apply deployment
                        kubectl apply -f k8s-deployment.yaml
                        
                        # Wait for rollout
                        kubectl rollout status deployment/frontend -n task-management --timeout=5m
                        kubectl rollout status deployment/user-service -n task-management --timeout=5m
                        kubectl rollout status deployment/task-service -n task-management --timeout=5m
                        kubectl rollout status deployment/notification-service -n task-management --timeout=5m
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    sh """
                        echo "Checking pod status..."
                        kubectl get pods -n task-management
                        
                        echo "Checking services..."
                        kubectl get svc -n task-management
                        
                        echo "Checking ingress..."
                        kubectl get ingress -n task-management
                    """
                }
            }
        }
    }
    
    post {
        success {
            script {
                def ingressUrl = sh(
                    script: "kubectl get ingress task-management-ingress -n task-management -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo 'N/A'",
                    returnStdout: true
                ).trim()
                
                echo """
                    ✅ Deployment Successful!
                    Job: ${env.JOB_NAME}
                    Build: #${env.BUILD_NUMBER}
                    Commit: ${env.GIT_COMMIT_SHORT}
                    Application URL: https://raghu.buzz
                    ALB URL: http://${ingressUrl}
                """
            }
        }
        
        failure {
            script {
                echo """
                    ❌ Deployment Failed!
                    Job: ${env.JOB_NAME}
                    Build: #${env.BUILD_NUMBER}
                    Check console output for details
                """
                
                sh """
                    echo "Pod status:"
                    kubectl get pods -n task-management 2>/dev/null || echo "Unable to fetch pods"
                    
                    echo "Recent events:"
                    kubectl get events -n task-management --sort-by='.lastTimestamp' 2>/dev/null | tail -20 || echo "Unable to fetch events"
                """
            }
        }
    }
}
