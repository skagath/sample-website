pipeline {
    agent any

     environment{
       ECR_REPO = "sample-app"
       ECR_REGISTRY = "438465160558.dkr.ecr.us-east-1.amazonaws.com"
       IMAGE_TAG ="latest"
       CLUSTER = "sampleapp"
       REGION = "us-east-1"
       SERVICE  = "svc-sample-app"
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
                    docker.build("${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}",".")
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
                docker.withRegistry(capstoneRegistry, registryCredential){
                    dockerImage.push("$BUILD_NUMBER")
                    dockerImage.push('latest')
                }
            }
         }
      }

      stage('Deploy to ecs'){
         when {
                branch "master"
         }
         steps{
            withAWS(credentials: 'aws_secret', region: 'us-east-1'){
                sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
            }
         }
      }

    }
}
