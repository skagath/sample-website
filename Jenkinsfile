pipeline {
    agent any

     environment{
       registryCredential = 'ecr:us-east-1:aws_secret'
       appRegistry = "sample-app"
       capstoneRegistry = "438465160558.dkr.ecr.us-east-1.amazonaws.com/sample-app"
       cluster = "sampleapp"
       service = "svc-sample-app"
   }

    stages {
        stage('Clone Website') {
            steps {
                git url:'https://github.com/skagath/sample-website.git'
            }
        }

       stage("Docker Build Image"){
           steps{
             script{
                dockerImage = docker.build(appRegistry+ ":$BUILD_NUMBER",".")
             }
           }
      }

       stage('Test Website') {
            steps {
                // Run tests on the website
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
