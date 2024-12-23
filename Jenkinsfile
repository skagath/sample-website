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

        stage('Docker Build Image') {
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

        stage('Upload App Image') {
            steps {
                script {
                    withCredentials([aws(credentialsId: 'aws_secret', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                        aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                        """
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

                        echo "Using Task Definition ARN: ${taskDefinitionArn}"

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
                    def last50Logs = currentBuild.rawBuild.getLog(50).join('\n')
                    echo "Last 50 Lines of Logs:\n${last50Logs}"

                    def maxSlackMessageLength = 3900
                    if (last50Logs.length() > maxSlackMessageLength) {
                        last50Logs = last50Logs.take(maxSlackMessageLength) + "\n... (truncated)"
                    }

                    slackSend channel: "${SLACK_CHANNEL_NAME}",
                              color: 'danger',
                              message: "❌ Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} failed! Check logs: ${env.JENKINS_URL}\nLast 50 Lines of Logs:\n${last50Logs}",
                              tokenCredentialId: 'slack-tocken'
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
