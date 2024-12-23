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
                    withCredentials([aws(credentialsId: 'aws_secret', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh "aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                        sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                        sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                withCredentials([aws(credentialsId: 'aws_secret', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    script {
                        def taskDefinitionResponse = sh(
                            script: """
                            aws ecs register-task-definition \
                                --cli-input-json file://task-definition.json \
                                --region ${REGION}
                            """,
                            returnStdout: true
                        ).trim()

                        echo "Task Definition Response: ${taskDefinitionResponse}"

                        def taskDefinitionArn = sh(
                            script: "echo '${taskDefinitionResponse}' | jq -r '.taskDefinition.taskDefinitionArn'",
                            returnStdout: true
                        ).trim()

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
                try {
                    // Fetch last 5000 lines of logs
                    def allLogs = currentBuild.rawBuild.getLog(100)

                    // Print the first 20 lines in Jenkins console for debugging
                    echo "First 20 lines of logs:\n" + allLogs.take(100).join('\n')

                    // Filter all lines containing 'error' (case-insensitive)
                    def errorLogs = allLogs.findAll { it =~ /(?i)error/ }

                    if (errorLogs && errorLogs.size() > 0) {
                        def errorMessage = errorLogs.join("\n")

                        // Print filtered error logs in Jenkins console for verification
                        echo "Filtered Error Logs:\n${errorMessage}"

                        // Slack allows ~4000 characters per message
                        def maxSlackMessageLength = 3900
                        if (errorMessage.length() > maxSlackMessageLength) {
                            errorMessage = errorMessage.take(maxSlackMessageLength) + "\n... (truncated)"
                        }

                        slackSend channel: "${SLACK_CHANNEL_NAME}",
                                  color: 'danger',
                                  message: "❌ Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} failed! Check logs: ${env.JENKINS_URL}\nError Logs:\n${errorMessage}",
                                  tokenCredentialId: 'slack-tocken'
                    } else {
                        echo "No error logs found in the captured logs."

                        slackSend channel: "${SLACK_CHANNEL_NAME}",
                                  color: 'danger',
                                  message: "❌ Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} failed! No error logs found. Check logs: ${env.JENKINS_URL}",
                                  tokenCredentialId: 'slack-tocken'
                    }
                } catch (Exception e) {
                    echo "Failed to fetch or process logs: ${e.getMessage()}"

                    slackSend channel: "${SLACK_CHANNEL_NAME}",
                              color: 'danger',
                              message: "❌ Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} failed! Error retrieving logs: ${e.getMessage()}. Check logs: ${env.JENKINS_URL}",
                              tokenCredentialId: 'slack-tocken'
                }
            }
        }
    }
}
