pipeline {
    agent any
    
    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = credentials('aws-account-id')
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        EKS_CLUSTER_NAME = 'raghu-eks-cluster'
        DOCKER_BUILDKIT = '1'
        SLACK_CHANNEL = '#deployments'
        SLACK_CREDENTIAL_ID = 'slack-token'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    // Send build start notification
                    slackSend(
                        channel: env.SLACK_CHANNEL,
                        color: '#439FE0',
                        message: """
                            🚀 *Deployment Started*
                            *Job:* ${env.JOB_NAME}
                            *Build:* #${env.BUILD_NUMBER}
                            *Started by:* ${env.BUILD_USER ?: 'Jenkins'}
                            *Branch:* ${env.GIT_BRANCH ?: 'main'}
                        """.stripIndent(),
                        tokenCredentialId: env.SLACK_CREDENTIAL_ID
                    )
                }
                
                checkout scm
                
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    env.BUILD_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                    env.GIT_COMMIT_MSG = sh(
                        script: "git log -1 --pretty=%B",
                        returnStdout: true
                    ).trim()
                }
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    slackSend(
                        channel: env.SLACK_CHANNEL,
                        color: 'good',
                        message: "🔨 Building Docker images...",
                        tokenCredentialId: env.SLACK_CREDENTIAL_ID
                    )
                }
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
                    slackSend(
                        channel: env.SLACK_CHANNEL,
                        color: 'good',
                        message: "📦 Pushing images to ECR...",
                        tokenCredentialId: env.SLACK_CREDENTIAL_ID
                    )
                    
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
                    slackSend(
                        channel: env.SLACK_CHANNEL,
                        color: 'good',
                        message: "☸️ Deploying to EKS cluster: ${EKS_CLUSTER_NAME}...",
                        tokenCredentialId: env.SLACK_CREDENTIAL_ID
                    )
                    
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
                
                def duration = currentBuild.durationString.replace(' and counting', '')
                
                slackSend(
                    channel: env.SLACK_CHANNEL,
                    color: 'good',
                    message: """
                        ✅ *Deployment Successful!*
                        *Job:* ${env.JOB_NAME}
                        *Build:* #${env.BUILD_NUMBER}
                        *Commit:* ${env.GIT_COMMIT_SHORT}
                        *Message:* ${env.GIT_COMMIT_MSG}
                        *Duration:* ${duration}
                        *Application URL:* https://raghu.buzz
                        *ALB URL:* http://${ingressUrl}
                        
                        All services deployed and running! 🎉
                    """.stripIndent(),
                    tokenCredentialId: env.SLACK_CREDENTIAL_ID
                )
                
                echo "Application URL: http://${ingressUrl}"
                echo "Domain: https://raghu.buzz"
            }
        }
        
        failure {
            script {
                def duration = currentBuild.durationString.replace(' and counting', '')
                def failedStage = currentBuild.rawBuild.getAction(org.jenkinsci.plugins.workflow.actions.ErrorAction.class)?.error?.message ?: 'Unknown'
                
                def podStatus = sh(
                    script: "kubectl get pods -n task-management 2>/dev/null || echo 'Unable to fetch pod status'",
                    returnStdout: true
                ).trim()
                
                slackSend(
                    channel: env.SLACK_CHANNEL,
                    color: 'danger',
                    message: """
                        ❌ *Deployment Failed!*
                        *Job:* ${env.JOB_NAME}
                        *Build:* #${env.BUILD_NUMBER}
                        *Commit:* ${env.GIT_COMMIT_SHORT}
                        *Duration:* ${duration}
                        *Failed Stage:* ${env.STAGE_NAME ?: 'Unknown'}
                        
                        *Pod Status:*
                        ```${podStatus}```
                        
                        Check console output: ${env.BUILD_URL}console
                    """.stripIndent(),
                    tokenCredentialId: env.SLACK_CREDENTIAL_ID
                )
                
                sh """
                    echo "Pod status:"
                    kubectl get pods -n task-management 2>/dev/null || echo "Unable to fetch pods"
                    
                    echo "Recent events:"
                    kubectl get events -n task-management --sort-by='.lastTimestamp' 2>/dev/null | tail -20 || echo "Unable to fetch events"
                """
            }
        }
        
        unstable {
            script {
                slackSend(
                    channel: env.SLACK_CHANNEL,
                    color: 'warning',
                    message: """
                        ⚠️ *Deployment Unstable*
                        *Job:* ${env.JOB_NAME}
                        *Build:* #${env.BUILD_NUMBER}
                        
                        Some tests or checks failed. Please review.
                        ${env.BUILD_URL}console
                    """.stripIndent(),
                    tokenCredentialId: env.SLACK_CREDENTIAL_ID
                )
            }
        }
        
        aborted {
            script {
                slackSend(
                    channel: env.SLACK_CHANNEL,
                    color: 'warning',
                    message: """
                        🛑 *Deployment Aborted*
                        *Job:* ${env.JOB_NAME}
                        *Build:* #${env.BUILD_NUMBER}
                        
                        Build was manually stopped or timed out.
                    """.stripIndent(),
                    tokenCredentialId: env.SLACK_CREDENTIAL_ID
                )
            }
        }
        
        always {
            cleanWs()
        }
    }
}
