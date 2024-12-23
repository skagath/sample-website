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
                    try {
                        withCredentials([aws(credentialsId: 'AWS-CREDENDS', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                            // Authenticate Docker to AWS ECR
                            sh "aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                            
                            // Tag the Docker image
                            sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                            
                            // Push the Docker image to ECR
                            sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                        }
                    } catch (Exception e) {
                        sendFailureLogToSlack("Upload App Image")
                        throw e // Re-throw the exception to mark the build as failed
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    try {
                        withCredentials([aws(credentialsId: 'AWS-CREDENDS', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
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
                    } catch (Exception e) {
                        sendFailureLogToSlack("Deploy to ECS")
                        throw e // Re-throw the exception to mark the build as failed
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
                sendFailureLogToSlack("Pipeline")
            }
        }
    }
}

// Helper function to send logs to Slack
def sendFailureLogToSlack(stageName) {
    try {
        // Fetch up to 1000 lines from the console log and extract the last 50 lines
        def fullLog = currentBuild.rawBuild.getLog(1000)
        def logLines = fullLog.takeRight(50).join('\n')

        // Send the Slack notification
        slackSend channel: "${env.SLACK_CHANNEL_NAME}",
                  color: 'danger',
                  message: """❌ Stage *${stageName}* failed in pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER}!
                              Check logs: ${env.JENKINS_URL}console
                              \n*Last 50 Log Lines:*\n```$logLines```""",
                  tokenCredentialId: 'slack-tocken'
    } catch (Exception ex) {
        echo "Error while sending failure logs to Slack: ${ex.message}"
    }
}
