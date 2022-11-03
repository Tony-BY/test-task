pipeline {
    environment {
        registry = "tonyby/test-task"
        registryCredential = 'dockerhub'
    }
    agent any

    stages {
        stage("Checkout code") {
            steps {
                echo "========executing Checkout code========"
                deleteDir()
                checkout([$class: 'GitSCM',
                          branches: [[name: '*/main']],
                          userRemoteConfigs: [[url: 'https://github.com/Tony-BY/test-task.git']]])
            }
        }
        stage("Validate Dockerfile") {
            steps {
                echo "========executing hadolint========"
                sh "docker run --rm -i hadolint/hadolint < Dockerfile"
            }
        }
        stage("Build image") {
            steps {
                echo "========Build docker image========"
                script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"                    
                }
            }
        }
        stage("Test image") {
            steps {
                echo "========Test docker image========"

            }
        }
        stage("Push image to DockerHub") {
            steps {
                script {
                    docker.withRegistry( '', registryCredential ) {
                        dockerImage.push()
                    }
                }
            }
        }
    }
}