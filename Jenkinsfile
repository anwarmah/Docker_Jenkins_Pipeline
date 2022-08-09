pipeline {
    agent any
    environment {
    dockerImage = ''
    PATH = "$PATH:/usr/local/bin"
    stages {
            stage('Cloning our Git') {
                when {
                    branch 'master'
                }
                steps {
                git 'https://github.com/anwarmah/Docker_Jenkins_Pipeline.git'
                }
            }

            stage('Building Docker Image') {
                when {
                    branch 'master'
                }
                steps {
                    script {
                        dockerImage = docker.build("backendapi")
                    }
                }
            }

            stage('Deploying Docker Image to ECR') {
                when {
                    branch 'master'
                }
                steps {
                    script {
                        docker.withRegistry('https://720766170633.dkr.ecr.us-east-2.amazonaws.com', 'ecr:us-east-2:aws-credentials') {
                        dockerImage.push("${env.BUILD_NUMBER}")
                        dockerImage.push("latest")
                        }
                    }
                }
            }

            stage('Deploy') {
                when {
                    branch 'master'
                }
                steps{
                  sh "kubectl apply -f deployment.yml"
                }
            }
        }
    }
}
