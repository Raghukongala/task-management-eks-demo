# Slack Integration for Jenkins Pipeline

Complete guide to set up Slack notifications for your Jenkins EKS deployment pipeline.

## What You'll Get

Your Jenkins pipeline will send Slack notifications for:
- 🚀 **Build Started** - When deployment begins
- 🔨 **Building Images** - Docker image build progress
- 📦 **Pushing to ECR** - Image push status
- ☸️ **Deploying to EKS** - Kubernetes deployment progress
- ✅ **Success** - Deployment completed with URLs
- ❌ **Failure** - Deployment failed with error details
- ⚠️ **Unstable** - Tests or checks failed
- 🛑 **Aborted** - Build was stopped

## Prerequisites

- Slack workspace (free or paid)
- Admin access to create Slack apps
- Jenkins with Slack Notification plugin

## Step 1: Create Slack App

### 1.1 Go to Slack API
Visit: https://api.slack.com/apps

### 1.2 Create New App
1. Click "Create New App"
2. Choose "From scratch"
3. App Name: `Jenkins CI/CD`
4. Pick your workspace
5. Click "Create App"

### 1.3 Configure Bot Token Scopes
1. In the left sidebar, click "OAuth & Permissions"
2. Scroll to "Scopes" → "Bot Token Scopes"
3. Add these scopes:
   - `chat:write` - Send messages
   - `chat:write.public` - Send messages to public channels
   - `files:write` - Upload files (optional)
   - `incoming-webhook` - Post messages to specific channels

### 1.4 Install App to Workspace
1. Scroll to top of "OAuth & Permissions" page
2. Click "Install to Workspace"
3. Review permissions
4. Click "Allow"
5. Copy the "Bot User OAuth Token" (starts with `xoxb-`)
   - **IMPORTANT**: Save this token securely!

## Step 2: Create Slack Channel

### 2.1 Create Channel for Notifications
1. In Slack, click "+" next to "Channels"
2. Create a new channel:
   - Name: `deployments` (or your preferred name)
   - Description: "Jenkins CI/CD deployment notifications"
   - Make it Public or Private

### 2.2 Invite Jenkins Bot
1. Go to your `#deployments` channel
2. Type: `/invite @Jenkins CI/CD`
3. Or click channel name → Integrations → Add apps → Select "Jenkins CI/CD"

## Step 3: Install Jenkins Slack Plugin

### 3.1 Install Plugin
1. Jenkins → Manage Jenkins → Plugin Manager
2. Go to "Available plugins" tab
3. Search for "Slack Notification"
4. Check the box next to "Slack Notification Plugin"
5. Click "Install without restart"
6. Wait for installation to complete

### 3.2 Verify Installation
1. Go to Manage Jenkins → System Configuration
2. You should see "Slack" section

## Step 4: Configure Slack in Jenkins

### 4.1 Add Slack Token to Jenkins Credentials
1. Jenkins → Manage Jenkins → Credentials
2. Click "System" → "Global credentials"
3. Click "Add Credentials"
4. Configure:
   - **Kind**: Secret text
   - **Scope**: Global
   - **Secret**: Paste your Slack Bot Token (xoxb-...)
   - **ID**: `slack-token`
   - **Description**: Slack Bot Token for Notifications
5. Click "Create"

### 4.2 Configure Slack Settings
1. Jenkins → Manage Jenkins → System
2. Scroll to "Slack" section
3. Configure:
   - **Workspace**: Your workspace name (e.g., `mycompany`)
   - **Credential**: Select "Slack Bot Token for Notifications"
   - **Default channel**: `#deployments`
   - **Bot user**: Check this box
4. Click "Test Connection"
   - You should see "Success" message
   - Check your Slack channel for test message
5. Click "Save"

## Step 5: Update Jenkinsfile

The Jenkinsfile has already been updated with Slack notifications. Verify these settings:

```groovy
environment {
    SLACK_CHANNEL = '#deployments'      // Your channel name
    SLACK_CREDENTIAL_ID = 'slack-token' // Your credential ID
}
```

If you used different names, update them in the Jenkinsfile.

## Step 6: Test Slack Notifications

### 6.1 Run a Test Build
1. Go to your Jenkins pipeline job
2. Click "Build Now"
3. Watch your Slack channel for notifications

### 6.2 Expected Notifications

**Build Start:**
```
🚀 Deployment Started
Job: task-management-eks-deploy
Build: #42
Started by: admin
Branch: main
```

**Build Progress:**
```
🔨 Building Docker images...
📦 Pushing images to ECR...
☸️ Deploying to EKS cluster: raghu-eks-cluster...
```

**Success:**
```
✅ Deployment Successful!
Job: task-management-eks-deploy
Build: #42
Commit: a1b2c3d
Message: Add new feature
Duration: 5 min 23 sec
Application URL: https://raghu.buzz
ALB URL: http://task-management-alb-xxx.elb.amazonaws.com

All services deployed and running! 🎉
```

**Failure:**
```
❌ Deployment Failed!
Job: task-management-eks-deploy
Build: #42
Commit: a1b2c3d
Duration: 2 min 15 sec
Failed Stage: Deploy to EKS

Pod Status:
NAME                                    READY   STATUS    RESTARTS
frontend-xxx                            0/1     Error     0
user-service-xxx                        1/1     Running   0

Check console output: http://jenkins-url/job/task-management-eks-deploy/42/console
```

## Step 7: Customize Notifications

### Change Channel
Edit Jenkinsfile:
```groovy
environment {
    SLACK_CHANNEL = '#your-channel-name'
}
```

### Add More Stages
Add notifications to any stage:
```groovy
stage('Your Stage') {
    steps {
        script {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: "Your custom message",
                tokenCredentialId: env.SLACK_CREDENTIAL_ID
            )
        }
    }
}
```

### Color Codes
- `good` - Green (success)
- `warning` - Yellow (warning)
- `danger` - Red (error)
- `#439FE0` - Blue (info)
- Any hex color code

### Add Attachments
For rich formatting:
```groovy
slackSend(
    channel: env.SLACK_CHANNEL,
    attachments: [
        [
            color: 'good',
            title: 'Deployment Details',
            text: 'All services deployed successfully',
            fields: [
                [title: 'Environment', value: 'Production', short: true],
                [title: 'Version', value: '1.2.3', short: true]
            ]
        ]
    ],
    tokenCredentialId: env.SLACK_CREDENTIAL_ID
)
```

## Step 8: Advanced Configuration

### Multiple Channels
Send to different channels based on result:
```groovy
post {
    success {
        slackSend(channel: '#deployments', color: 'good', message: 'Success!')
    }
    failure {
        slackSend(channel: '#alerts', color: 'danger', message: 'Failed!')
    }
}
```

### Mention Users
Notify specific users on failure:
```groovy
slackSend(
    channel: env.SLACK_CHANNEL,
    color: 'danger',
    message: "<@U12345678> Deployment failed! Please check.",
    tokenCredentialId: env.SLACK_CREDENTIAL_ID
)
```

To get user ID:
1. Click on user in Slack
2. Click "View full profile"
3. Click "More" → "Copy member ID"

### Thread Replies
Reply to original message:
```groovy
def slackResponse = slackSend(
    channel: env.SLACK_CHANNEL,
    message: 'Build started'
)

// Later, reply to the thread
slackSend(
    channel: env.SLACK_CHANNEL,
    message: 'Build completed',
    replyBroadcast: false,
    timestamp: slackResponse.ts
)
```

### Add Build Artifacts
Upload files to Slack:
```groovy
slackUploadFile(
    channel: env.SLACK_CHANNEL,
    filePath: 'test-results.xml',
    initialComment: 'Test results attached'
)
```

## Troubleshooting

### Issue: "Slack notification failed"

**Check 1: Verify Token**
```bash
# Test token with curl
curl -X POST https://slack.com/api/auth.test \
  -H "Authorization: Bearer xoxb-your-token-here"
```

**Check 2: Verify Bot is in Channel**
- Go to Slack channel
- Type `/invite @Jenkins CI/CD`

**Check 3: Check Scopes**
- Go to https://api.slack.com/apps
- Select your app
- OAuth & Permissions → Scopes
- Ensure `chat:write` is added

### Issue: "Channel not found"

**Solution:**
- Channel name must include `#` (e.g., `#deployments`)
- For private channels, bot must be invited first
- Check spelling of channel name

### Issue: "No Slack notifications at all"

**Check Jenkins Logs:**
```bash
# Docker installation
docker exec jenkins tail -f /var/jenkins_home/logs/jenkins.log

# System installation
sudo tail -f /var/log/jenkins/jenkins.log
```

**Verify Plugin:**
1. Manage Jenkins → Plugin Manager
2. Installed plugins → Search "Slack"
3. Should show "Slack Notification Plugin"

### Issue: "Test Connection Failed"

**Solutions:**
1. Verify workspace name is correct
2. Check token starts with `xoxb-`
3. Ensure "Bot user" checkbox is checked
4. Try reinstalling the app to workspace

## Security Best Practices

1. **Never commit tokens to Git**
   - Always use Jenkins credentials
   - Add tokens to `.gitignore`

2. **Use least privilege**
   - Only grant necessary Slack scopes
   - Limit bot to specific channels

3. **Rotate tokens regularly**
   - Generate new tokens every 90 days
   - Update Jenkins credentials

4. **Monitor bot activity**
   - Review Slack app logs
   - Check for unauthorized usage

## Example Notification Messages

### Deployment Started
```
🚀 Deployment Started
Job: task-management-eks-deploy
Build: #42
Started by: John Doe
Branch: main
Commit: a1b2c3d - Add user authentication
```

### Deployment Success
```
✅ Deployment Successful!
Job: task-management-eks-deploy
Build: #42
Commit: a1b2c3d
Message: Add user authentication
Duration: 5 min 23 sec
Application URL: https://raghu.buzz
ALB URL: http://task-management-alb-xxx.elb.amazonaws.com

All services deployed and running! 🎉

Services:
• Frontend: 1/1 Running
• User Service: 1/1 Running
• Task Service: 1/1 Running
• Notification Service: 1/1 Running
```

### Deployment Failed
```
❌ Deployment Failed!
Job: task-management-eks-deploy
Build: #42
Commit: a1b2c3d
Duration: 2 min 15 sec
Failed Stage: Deploy to EKS

Pod Status:
NAME                                    READY   STATUS         RESTARTS
frontend-xxx                            1/1     Running        0
user-service-xxx                        0/1     ImagePullErr   0
task-service-xxx                        1/1     Running        0
notification-service-xxx                1/1     Running        0

Error: Failed to pull image from ECR

Check console output: http://jenkins-url/job/task-management-eks-deploy/42/console
```

## Quick Reference

### Slack API Documentation
- https://api.slack.com/messaging/sending
- https://api.slack.com/reference/messaging/attachments

### Jenkins Slack Plugin
- https://plugins.jenkins.io/slack/

### Color Codes
- Success: `good` or `#36a64f`
- Warning: `warning` or `#ff9900`
- Error: `danger` or `#ff0000`
- Info: `#439FE0`

### Emoji Reference
- 🚀 `:rocket:` - Build started
- ✅ `:white_check_mark:` - Success
- ❌ `:x:` - Failure
- ⚠️ `:warning:` - Warning
- 🔨 `:hammer:` - Building
- 📦 `:package:` - Packaging
- ☸️ `:wheel_of_dharma:` - Kubernetes
- 🎉 `:tada:` - Celebration

## Next Steps

1. ✅ Create Slack app
2. ✅ Install Jenkins plugin
3. ✅ Configure credentials
4. ✅ Test notifications
5. ✅ Customize messages
6. ✅ Set up multiple channels (optional)
7. ✅ Add user mentions (optional)

Your Jenkins pipeline now sends beautiful Slack notifications! 🎉
