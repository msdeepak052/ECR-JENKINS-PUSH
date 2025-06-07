Here's a step-by-step guide to create AWS ECR credentials in Jenkins for your `docker.withRegistry()` authentication:

### **Steps to Create AWS Credentials in Jenkins**

#### **1. Get Your AWS Access Keys**
First, ensure you have an IAM user with ECR access:
1. Go to **AWS IAM Console** → **Users** → Select your user
2. Under **Security credentials** → **Create access key**
3. Copy the **Access Key ID** and **Secret Access Key**

> ⚠️ Save these immediately - the secret key is shown only once!

---

#### **2. Create Jenkins Credentials**
1. **Open Jenkins** → Go to your pipeline project
2. Navigate to:
   ```
   Dashboard → Manage Jenkins → Credentials → System → Global credentials (unrestricted)
   ```
3. Click **Add Credentials**:

   - **Kind**: `AWS Credentials`
   - **Scope**: `Global`  
   - **ID**: `ecr:ap-south-1:aws-credentials` *(must match your Jenkinsfile)*  
   - **Description**: `AWS ECR Push Credentials`  
   - **Access Key ID**: Paste your AWS key  
   - **Secret Access Key**: Paste your AWS secret  
   - Click **OK**

![Jenkins AWS Credentials Screenshot](https://i.imgur.com/JQ4W5vL.png)

---

#### **3. Verify IAM Permissions**
Ensure your IAM user has these permissions (attach policy):
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload"
            ],
            "Resource": "*"
        }
    ]
}
```

---

#### **4. Test in Pipeline**
Update your Jenkinsfile to use the credentials:
```groovy
stage('Push to ECR') {
    steps {
        script {
            docker.withRegistry(
                "https://123456789.dkr.ecr.ap-south-1.amazonaws.com",
                "ecr:ap-south-1:aws-credentials"  // Must match credential ID
            ) {
                env.dockerImage.push()
            }
        }
    }
}
```

---

### **Troubleshooting**
1. **"Invalid Credentials" Error**  
   - Double-check the **ID** matches exactly  
   - Verify AWS keys are active in IAM  

2. **Permission Denied**  
   - Ensure the IAM user has the ECR permissions listed above  

3. **Region Mismatch**  
   Confirm the region in `ecr:ap-south-1:aws-credentials` matches your ECR repo  

---

### **Alternative: Instance Role (For EC2 Jenkins)**
If Jenkins runs on AWS EC2:
1. Attach an IAM role with ECR permissions to the EC2 instance  
2. Remove `docker.withRegistry()` credentials parameter  
3. Jenkins will auto-use the instance role  

```groovy
docker.withRegistry("https://your-ecr-url") {
    dockerImage.push()  // Uses instance role
}
``` 

This eliminates key management!


```Jenkinsfile

pipeline {
    agent any
    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = 'app/tour-travels-webapp'
        APP_NAME = 'tour-travels-webapp'
        VERSION = "1.0.${BUILD_NUMBER}"
    }
    stages {
        stage('Prepare') {
            steps {
                script {
                    // Get ECR URL once and store in env
                    env.ECR_URL = sh(
                        script: "aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} --query 'repositories[0].repositoryUri' --output text",
                        returnStdout: true
                    ).trim()
                    env.IMAGE_URI = "${env.ECR_URL}:${env.VERSION}"
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    // Build ONCE and store the Docker image object
                    env.dockerImage = docker.build("${env.IMAGE_URI}", 
                        "--build-arg APP_NAME=${env.APP_NAME} ."
                    )
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    // Scan the already-built image by tag
                    sh "trivy image --exit-code 1 --ignore-unfixed ${env.IMAGE_URI}"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    // Authenticate and push using the stored image reference
                    docker.withRegistry(
                        "https://${env.ECR_URL.split(':')[0]}", 
                        'ecr:ap-south-1:aws-credentials'
                    ) {
                        env.dockerImage.push()
                        env.dockerImage.push('latest')
                    }
                }
            }
        }
    }
}

```

