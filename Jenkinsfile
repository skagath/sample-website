pipeline {
    agent any

     environment{
       ECR_REPO = "sample-app"
       ECR_REGISTRY = "438465160558.dkr.ecr.us-east-1.amazonaws.com"
       IMAGE_TAG ="latest"
       CLUSTER = "sampleapp"
       REGION = "us-east-1"
       SERVICE  = "svc-sample-app"
       ECS_TASK_DEFINITION = "tf-sampleapp"
   }

    stages {
        stage('Clone Website') {
            steps {
                git credentialsId: 'github_creds', url: 'https://github.com/skagath/sample-website.git'
            }
        }

       stage("Docker Build Image"){
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

       stage("Upload App Image"){
         steps{
            script{
              
                withCredentials([aws(credentialsId: 'aws_secret', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh "aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                }
            }
         }
      }                                                                

      stage('Deploy to ecs'){
        
         steps{
            withCredentials([aws(credentialsId: 'aws_secret', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                script {
                        // Register a new ECS task definition revision with the updated image
                        def taskDefinitionResponse = sh(
                            script: """
                            aws ecs register-task-definition \
                              --family ${ECS_TASK_DEFINITION} \
                              --container-definitions '[{
                                  "name": "your-container-name",
                                  "image": "${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}",
                                  "essential": true,
                                  "memory": 205,
                                  "cpu": 0
                              }]'
                            """,
                            returnStdout: true
                        ).trim()

                        // Extract the new task definition revision
                        def taskDefinitionArn = sh(
                            script: "echo ${taskDefinitionResponse} | jq -r '.taskDefinition.taskDefinitionArn'",
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
}
