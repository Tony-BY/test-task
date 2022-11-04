# Report
App repository https://github.com/benc-uk/python-demoapp

### Message about status of deployment and approve interface on Jenkins WebUI
![image](screens/manual_approve.png)
### After approvement
![image](screens/deploy.png)

### Jenkinsfile listing
```groovy
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
            slackSend (color: '#2EB886', message: "✔️ Deployment success: Job: ${env.JOB_NAME} Build: ${env.BUILD_NUMBER}")
        }
        failure {
            slackSend (color: '#A30200', message: "❌ Deployment failed: Job: ${env.JOB_NAME} Build: ${env.BUILD_NUMBER}")
        }
    }
}
```

### Slack Notification
![image](screens/notification.png)

### App in pre-prod 
![image](screens/app_in_pre-prod.png)
### App in prod
![image](screens/app_in_prod.png)
### Job Log

<details>
  <summary>Console Log</summary>

   ```bash
    Started by user admin
    Obtained Jenkinsfile from git https://github.com/Tony-BY/test-task.git
    [Pipeline] Start of Pipeline
    [Pipeline] node
    Running on Jenkins in /var/lib/jenkins/workspace/test-task
    [Pipeline] {
    [Pipeline] stage
    [Pipeline] { (Declarative: Checkout SCM)
    [Pipeline] checkout
    Selected Git installation does not exist. Using Default
    The recommended git tool is: NONE
    using credential github_credentials
    > git rev-parse --resolve-git-dir /var/lib/jenkins/workspace/test-task/.git # timeout=10
    Fetching changes from the remote Git repository
    > git config remote.origin.url https://github.com/Tony-BY/test-task.git # timeout=10
    Fetching upstream changes from https://github.com/Tony-BY/test-task.git
    > git --version # timeout=10
    > git --version # 'git version 2.25.1'
    using GIT_ASKPASS to set credentials github user and password
    > git fetch --tags --force --progress -- https://github.com/Tony-BY/test-task.git +refs/heads/*:refs/remotes/origin/* # timeout=10
    > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
    Checking out Revision 5fe5a939f578ae333a89cc0a4db24b0515b1e107 (refs/remotes/origin/main)
    > git config core.sparsecheckout # timeout=10
    > git checkout -f 5fe5a939f578ae333a89cc0a4db24b0515b1e107 # timeout=10
    Commit message: "Update manifest with container version 28"
    > git rev-list --no-walk 56461b0d77f9bf58c9c90b08555589fbc73345d9 # timeout=10
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] withEnv
    [Pipeline] {
    [Pipeline] withEnv
    [Pipeline] {
    [Pipeline] stage
    [Pipeline] { (Checkout code)
    [Pipeline] echo
    ========executing Checkout code========
    [Pipeline] deleteDir
    [Pipeline] checkout
    Selected Git installation does not exist. Using Default
    The recommended git tool is: NONE
    No credentials specified
    Cloning the remote Git repository
    Cloning repository https://github.com/Tony-BY/test-task.git
    > git init /var/lib/jenkins/workspace/test-task # timeout=10
    Fetching upstream changes from https://github.com/Tony-BY/test-task.git
    > git --version # timeout=10
    > git --version # 'git version 2.25.1'
    > git fetch --tags --force --progress -- https://github.com/Tony-BY/test-task.git +refs/heads/*:refs/remotes/origin/* # timeout=10
    > git config remote.origin.url https://github.com/Tony-BY/test-task.git # timeout=10
    > git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/* # timeout=10
    Avoid second fetch
    > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
    Checking out Revision 5fe5a939f578ae333a89cc0a4db24b0515b1e107 (refs/remotes/origin/main)
    > git config core.sparsecheckout # timeout=10
    > git checkout -f 5fe5a939f578ae333a89cc0a4db24b0515b1e107 # timeout=10
    Commit message: "Update manifest with container version 28"
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] stage
    [Pipeline] { (Validate Dockerfile)
    [Pipeline] echo
    ========executing hadolint========
    [Pipeline] sh
    + docker run --rm -i hadolint/hadolint
    -:3 DL3048 [96mstyle[0m: Invalid label key.
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] stage
    [Pipeline] { (Build image)
    [Pipeline] echo
    ========Build docker image========
    [Pipeline] script
    [Pipeline] {
    [Pipeline] isUnix
    [Pipeline] withEnv
    [Pipeline] {
    [Pipeline] sh
    + docker build -t tonyby/test-task:29 .
    Sending build context to Docker daemon  267.8kB

    Step 1/11 : FROM python:3.9-slim-buster
    ---> a635b96ee6f8
    Step 2/11 : LABEL Name="Python Flask Demo App" Version=1.4.2
    ---> Using cache
    ---> 4fea2911ed31
    Step 3/11 : LABEL org.opencontainers.image.source = "https://github.com/benc-uk/python-demoapp"
    ---> Using cache
    ---> 36fdd7cde2cb
    Step 4/11 : ARG srcDir=src
    ---> Using cache
    ---> 4bd121ad530b
    Step 5/11 : WORKDIR /app
    ---> Using cache
    ---> 0c64d7582922
    Step 6/11 : COPY $srcDir/requirements.txt .
    ---> Using cache
    ---> d739608b5e34
    Step 7/11 : RUN pip install --no-cache-dir -r requirements.txt
    ---> Using cache
    ---> d36b859a7a12
    Step 8/11 : COPY $srcDir/run.py .
    ---> Using cache
    ---> f600c75f83a9
    Step 9/11 : COPY $srcDir/app ./app
    ---> Using cache
    ---> 00add215dec5
    Step 10/11 : EXPOSE 5000
    ---> Using cache
    ---> ac458a7b39cf
    Step 11/11 : CMD ["gunicorn", "-b", "0.0.0.0:5000", "run:app"]
    ---> Using cache
    ---> 7a978d8bf259
    Successfully built 7a978d8bf259
    Successfully tagged tonyby/test-task:29
    [Pipeline] }
    [Pipeline] // withEnv
    [Pipeline] }
    [Pipeline] // script
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] stage
    [Pipeline] { (Test image)
    [Pipeline] echo
    ========Test docker image========
    [Pipeline] sh
    + docker run -d --name testcontainer -p 5000:5000 -i tonyby/test-task:29
    25b17f3bf09be4ba00da782fe045685532e82d5b1798a7e59e75dd2d88310c18
    [Pipeline] sleep
    Sleeping for 5 sec
    [Pipeline] sh
    + curl http://localhost:5000
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed

    0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
    100  3467  100  3467    0     0   241k      0 --:--:-- --:--:-- --:--:--  241k
    <!DOCTYPE html>
    <html>
    <head>
        <title>Python DemoApp</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <link href="/static/css/main.css" rel="stylesheet" />
        <link href="/static/img/favicon.ico" rel="icon" />
        <link
        rel="stylesheet"
        href="https://stackpath.bootstrapcdn.com/bootswatch/4.5.2/united/bootstrap.min.css"
        integrity="sha384-JW3PJkbqVWtBhuV/gsuyVVt3m/ecRJjwXC3gCXlTzZZV+zIEEl6AnryAriT7GWYm"
        crossorigin="anonymous"
        />
    </head>

    <body>
        <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <a class="navbar-brand logotext" href="/">
            <img src="static/img/python.svg" width="37" height="40" />
            &nbsp; Python Demo
        </a>

        <button
            class="navbar-toggler"
            type="button"
            data-toggle="collapse"
            data-target="#navbarNav"
            aria-controls="navbarNav"
            aria-expanded="true"
            aria-label="Toggle navigation"
        >
            <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="navbarNav">
            <ul class="navbar-nav mr-auto">
            <li class="nav-item active">
                <a class="btn btn-success btn-lg" href="/info">🧾 Info</a>
            </li>
            &nbsp;
            <li class="nav-item active">
                <a class="btn btn-success btn-lg" href="/monitor">🚦 Monitor</a>
            </li>
            </ul>
        </div>
        </nav>

        <script
        src="https://code.jquery.com/jquery-3.5.1.min.js"
        integrity="sha256-9/aliU8dGd2tb6OSsuzixeV4y/faTqgFtohetphbbj0="
        crossorigin="anonymous"
        ></script>

        <script
        src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.12.9/umd/popper.min.js"
        integrity="sha384-ApNbgh9B+Y1QKtv3Rn7W3mgPxhU9K/ScQsAP7hUibX39j7fakFPskvXusvfa0b4Q"
        crossorigin="anonymous"
        ></script>
        <script
        src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/js/bootstrap.min.js"
        integrity="sha384-JZR6Spejh4U02d8jOt6vLEHfe/JQGiRRSQQxSfFWpi1MquVdAyjUar5+76PVCmYl"
        crossorigin="anonymous"
        ></script>

        <div class="container body-content">
    <br />
    <div class="jumbotron">
    <h1><img src="/static/img/flask.png" height="80px" /> Python & Flask Demo App</h1>

    <div class="lead">
        This is a simple web application written in Python and using Flask. It has been designed with cloud demos &amp;
        containers in mind. Demonstrating capabilities such as auto scaling, deployment to Azure or Kubernetes, or anytime
        you want something quick and lightweight to run & deploy.
    </div>
    <br />

    <div class="dimmed-box">
        <p>
        <img src="static/img/github-2.svg" class="icon" />
        <a href="https://github.com/benc-uk/python-demoapp" class="btn btn-info btn px-4"> GitHub Project </a>

        &nbsp;&nbsp;&nbsp;
        <img src="static/img/docker-whale.svg" class="icon" />
        <a href="https://github.com/users/benc-uk/packages/container/package/python-demoapp" class="btn btn-info btn">
            Docker Images
        </a>
        </p>
        <hr />
        <p>
        <img src="static/img/python.svg" class="icon" />
        <a class="btn btn-info btn" href="https://azure.microsoft.com/en-gb/develop/python/">
            Get started with Azure &amp; Python
        </a>
        </p>

        <br />
        <p>Microsoft ❤ Open Source</p>
    </div>
    </div>
    </div>
        <span style="float: right">v1.4.2 [Ben Coleman, 2018-2021] &nbsp;&nbsp;&nbsp;</span>
    </body>
    </html>
    [Pipeline] sh
    + docker stop testcontainer
    testcontainer
    [Pipeline] sh
    + docker rm testcontainer
    testcontainer
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] stage
    [Pipeline] { (Push image to DockerHub)
    [Pipeline] script
    [Pipeline] {
    [Pipeline] withEnv
    [Pipeline] {
    [Pipeline] withDockerRegistry
    $ docker login -u tonyby -p ******** https://index.docker.io/v1/
    WARNING! Using --password via the CLI is insecure. Use --password-stdin.
    WARNING! Your password will be stored unencrypted in /var/lib/jenkins/workspace/test-task@tmp/221e6eca-045d-4da0-9012-5b3c2b7030ec/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store

    Login Succeeded
    [Pipeline] {
    [Pipeline] isUnix
    [Pipeline] withEnv
    [Pipeline] {
    [Pipeline] sh
    + docker tag tonyby/test-task:29 tonyby/test-task:29
    [Pipeline] }
    [Pipeline] // withEnv
    [Pipeline] isUnix
    [Pipeline] withEnv
    [Pipeline] {
    [Pipeline] sh
    + docker push tonyby/test-task:29
    The push refers to repository [docker.io/tonyby/test-task]
    09d5cc93197c: Preparing
    cbc437b58b21: Preparing
    b1529ab0e2cd: Preparing
    20b1e48d74fe: Preparing
    fa5bfd82652f: Preparing
    c6b69518a1a5: Preparing
    2bd15eb2618e: Preparing
    d0569174cd96: Preparing
    a8f9e3c1138b: Preparing
    74f8a626d86f: Preparing
    74f8a626d86f: Waiting
    c6b69518a1a5: Waiting
    2bd15eb2618e: Waiting
    d0569174cd96: Waiting
    a8f9e3c1138b: Waiting
    20b1e48d74fe: Layer already exists
    fa5bfd82652f: Layer already exists
    09d5cc93197c: Layer already exists
    cbc437b58b21: Layer already exists
    b1529ab0e2cd: Layer already exists
    c6b69518a1a5: Layer already exists
    d0569174cd96: Layer already exists
    74f8a626d86f: Layer already exists
    2bd15eb2618e: Layer already exists
    a8f9e3c1138b: Layer already exists
    29: digest: sha256:696bbcf38cfbe9f41614a55498367f5766fab8c76b9dec57b5890d41633012f8 size: 2412
    [Pipeline] }
    [Pipeline] // withEnv
    [Pipeline] }
    [Pipeline] // withDockerRegistry
    [Pipeline] }
    [Pipeline] // withEnv
    [Pipeline] }
    [Pipeline] // script
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] stage
    [Pipeline] { (Update manifests)
    [Pipeline] script
    [Pipeline] {
    [Pipeline] catchError
    [Pipeline] {
    [Pipeline] withCredentials
    Masking supported pattern matches of $GIT_PASSWORD
    [Pipeline] {
    [Pipeline] sh
    + git config user.email iantonio.by@gmail.com
    [Pipeline] sh
    + git config user.name Jenkins
    [Pipeline] echo
    ============================== Update pre-prod manifest ==============================
    [Pipeline] sh
    + sed -i s+tonyby/test-task.*+tonyby/test-task:29+g manifests/pre-prod/demoapp.yaml
    [Pipeline] echo
    ============================== Update prod manifest ==============================
    [Pipeline] sh
    + sed -i s+tonyby/test-task.*+tonyby/test-task:29+g manifests/prod/demoapp.yaml
    [Pipeline] sh
    + git add .
    [Pipeline] sh
    + git commit -m Update manifest with container version 29
    [detached HEAD a03266f] Update manifest with container version 29
    2 files changed, 2 insertions(+), 2 deletions(-)
    [Pipeline] sh
    Warning: A secret was passed to "sh" using Groovy String interpolation, which is insecure.
            Affected argument(s) used the following variable(s): [GIT_PASSWORD]
            See https://jenkins.io/redirect/groovy-string-interpolation for details.
    + git push https://tony-by:****@github.com/Tony-BY/test-task.git HEAD:main
    To https://github.com/Tony-BY/test-task.git
    5fe5a93..a03266f  HEAD -> main
    [Pipeline] }
    [Pipeline] // withCredentials
    [Pipeline] }
    [Pipeline] // catchError
    [Pipeline] }
    [Pipeline] // script
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] stage
    [Pipeline] { (Deploy in K8s pre-prod)
    [Pipeline] script
    [Pipeline] {
    [Pipeline] sh
    + kubectl apply -f manifests/pre-prod/. -n pre-prod
    deployment.apps/flaskdemo created
    service/flaskdemo-service created
    ingress.networking.k8s.io/ingress-sa created
    [Pipeline] sleep
    Sleeping for 5 sec
    [Pipeline] sh
    + kubectl get pods -n pre-prod
    NAME                         READY   STATUS    RESTARTS   AGE
    flaskdemo-75b7cc78bc-5xk5h   1/1     Running   0          5s
    [Pipeline] }
    [Pipeline] // script
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] stage
    [Pipeline] { (Deploy in K8s prod)
    [Pipeline] script
    [Pipeline] {
    [Pipeline] catchError
    [Pipeline] {
    [Pipeline] input
    Do you want to deploy application in production environment?
    Proceed or Abort
    Approved by admin
    [Pipeline] sh
    + kubectl apply -f manifests/prod/. -n prod
    deployment.apps/flaskdemo configured
    service/flaskdemo-service unchanged
    ingress.networking.k8s.io/ingress-sa unchanged
    [Pipeline] sleep
    Sleeping for 5 sec
    [Pipeline] sh
    + kubectl get pods -n prod
    NAME                         READY   STATUS    RESTARTS   AGE
    flaskdemo-75b7cc78bc-j8d8g   1/1     Running   0          5s
    [Pipeline] echo
    ================ Deleting app from pre-prod environment ================
    [Pipeline] sh
    + kubectl delete -f manifests/pre-prod/. -n pre-prod
    deployment.apps "flaskdemo" deleted
    service "flaskdemo-service" deleted
    ingress.networking.k8s.io "ingress-sa" deleted
    [Pipeline] }
    [Pipeline] // catchError
    [Pipeline] }
    [Pipeline] // script
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] stage
    [Pipeline] { (Declarative: Post Actions)
    [Pipeline] slackSend
    Slack Send Pipeline step running, values are - baseUrl: <empty>, teamDomain: antonskuratovich, channel: #notifications, color: #2EB886, botUser: false, tokenCredentialId: Slack, notifyCommitters: false, iconEmoji: <empty>, username: <empty>, timestamp: <empty>
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] }
    [Pipeline] // withEnv
    [Pipeline] }
    [Pipeline] // withEnv
    [Pipeline] }
    [Pipeline] // node
    [Pipeline] End of Pipeline
    Finished: SUCCESS
   ```
</details>  

