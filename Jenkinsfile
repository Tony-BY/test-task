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
                sh "docker run -d --name testcontainer -p 5000:5000 -i $registry:$BUILD_NUMBER"
                sleep 5
                sh "curl http://localhost:5000"
                sh "docker stop testcontainer"
                sh "docker rm testcontainer"

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
        stage("Update manifests") {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        withCredentials([usernamePassword(credentialsId: 'github_credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                            sh '''
                            git config user.email "iantonio.by@gmail.com"
                            git config user.name "Jenkins"
                            cat manifests/pre-prod/demoapp.yaml
                            echo '============================== Update pre-prod manifest =============================='
                            sed -i 's+tonyby/test-task.*+tonyby/test-task:${BUILD_NUMBER}+g' manifests/pre-prod/demoapp.yaml
                            cat manifests/pre-prod/demoapp.yaml
                            echo '============================== Update prod manifest =============================='
                            cat manifests/prod/demoapp.yaml
                            sed -i 's+tonyby/test-task.*+tonyby/test-task:${BUILD_NUMBER}+g' manifests/prod/demoapp.yaml
                            cat manifests/prod/demoapp.yaml
                            git add .
                            git commit -m "Update manifest with container version ${BUILD_NUMBER}"
                            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Tony-BY/test-task.git HEAD:main
                            '''
                        }
                    }
                }
            }

        }
    }
}