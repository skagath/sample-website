pipeline {
    agent any

    environment {
        ECR_REPO = "sample-app"
        ECR_REGISTRY = "438465160558.dkr.ecr.us-east-1.amazonaws.com"
        IMAGE_TAG = "latest"
        CLUSTER = "sampleapp"
        REGION = "us-east-1"
        SERVICE = "svc-sample-app"
        ECS_TASK_DEFINITION = "tf-sampleapp"
        SLACK_CHANNEL_NAME = "xyz"
    }

    stages {
        stage('Clone Website') {
            steps {
                git credentialsId: 'github_creds', url: 'https://github.com/skagath/sample-website.git'
            }
        }

        stage("Docker Build Image") {
            steps {
                script {
                    sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Test Website') {
            steps {
                sh 'echo "Running tests on the website"'
            }
        }

        stage("Upload App Image") {
            steps {
                script {
                    withCredentials([aws(credentialsId: 'AWS-CREDENDS', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        // Authenticate Docker to AWS ECR
                        sh "aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                        
                        // Tag the Docker image
                        sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                        
                        // Push the Docker image to ECR
                        sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                withCredentials([aws(credentialsId: 'AWS-CREDENDS', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    script {
                        // Register a new ECS task definition revision with the updated image
                        def taskDefinitionResponse = sh(
                            script: """
                            aws ecs register-task-definition \
                                --cli-input-json file://task-definition.json \
                                --region ${REGION}
                            """,
                            returnStdout: true
                        ).trim()

                        // Debug: Print the full response (optional)
                        echo "Task Definition Response: ${taskDefinitionResponse}"

                        // Extract the new task definition ARN using jq
                        def taskDefinitionArn = sh(
                            script: "echo '${taskDefinitionResponse}' | jq -r '.taskDefinition.taskDefinitionArn'",
                            returnStdout: true
                        ).trim()

                        // Update ECS service to use the new task definition revision
                        sh """
                        aws ecs update-service \
                            --cluster ${CLUSTER} \
                            --service ${SERVICE} \
                            --task-definition ${taskDefinitionArn} \
                            --force-new-deployment \
                            --region ${REGION}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed."
        }
        success {
            script {
                slackSend channel: "${SLACK_CHANNEL_NAME}",
                          color: 'good',
                          message: "✅ Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} succeeded!",
                          tokenCredentialId: 'slack-tocken'
            }
        }
        failure {
            script {
                // Fetch the entire console output
                def fullConsoleLog = currentBuild.rawBuild.getLog(10000).join('\n') // Fetches logs up to 10,000 lines

                // Clean up log content to avoid formatting issues in Slack
                def cleanLogs = fullConsoleLog.replaceAll('```', '')

                // Send failure message with logs to Slack
                slackSend channel: "${SLACK_CHANNEL_NAME}",
                          color: 'danger',
                          message: """❌ Pipeline *${env.JOB_NAME}* #${env.BUILD_NUMBER} failed!
Check the logs: ${env.JENKINS_URL}console

*Full Logs:*
```text
${cleanLogs}```""",
                          tokenCredentialId: 'slack-tocken'
            }
        }
    }
}
