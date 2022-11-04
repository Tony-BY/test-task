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
                            sh "git config user.email iantonio.by@gmail.com"
                            sh "git config user.name Jenkins"
                            // sh "cat manifests/pre-prod/demoapp.yaml"
                            echo "============================== Update pre-prod manifest =============================="
                            sh "sed -i 's+tonyby/test-task.*+tonyby/test-task:${BUILD_NUMBER}+g' manifests/pre-prod/demoapp.yaml"
                            // sh "cat manifests/pre-prod/demoapp.yaml"
                            // sh "cat manifests/prod/demoapp.yaml"
                            echo "============================== Update prod manifest =============================="
                            sh "sed -i 's+tonyby/test-task.*+tonyby/test-task:${BUILD_NUMBER}+g' manifests/prod/demoapp.yaml"
                            // sh "cat manifests/prod/demoapp.yaml"
                            sh "git add ."
                            sh "git commit -m 'Update manifest with container version ${BUILD_NUMBER}'"
                            sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Tony-BY/test-task.git HEAD:main"                            
                        }
                    }
                }
            }

        }
        stage("Deploy in K8s pre-prod") {
            steps {
                script {
                    sh "kubectl apply -f manifests/pre-prod/. -n pre-prod"
                    sleep 5
                    sh "kubectl get pods -n pre-prod"
                }
            }
        }
        stage("Deploy in K8s prod") {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        def prod = true
                        try {
                            input("Do you want to deploy application in production environment?")
                        }
                        catch(err) {
                            prod = false
                        }
                        try {
                            if(prod) {
                                sh "kubectl apply -f manifests/prod/. -n prod"
                                sleep 5
                                sh "kubectl get pods -n prod"
                                echo "================ Deleting app from pre-prod environment ================"
                                sh "kubectl delete -f manifests/pre-prod/. -n pre-prod"
                            }
                        }
                        catch(Exception err) {
                            error "Deployment failed!"
                        }
                    }
                }
            }
        }
    }
    post {
        success {
            slackSend (color: '#00FF00', message: "✔️ Deployment success: Job: '${env.JOB_NAME} Build: ${env.BUILD_NUMBER}'")
        }
        failure {
            slackSend (color: '#FF0000', message: "❌ Deployment failed: Job: '${env.JOB_NAME} Build: ${env.BUILD_NUMBER}'")
        }
    }
}