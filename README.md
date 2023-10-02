# Project 15: CICD on Kubernetes Cluster

This project builds a CICD pipeline of a web application that is deployed on a kubernetes cluster. Once code is pushed to Github, Jenkins pipeline is triggered via webhook to first run code tests on sonarqube server, if quality gates are passed maven builds the artifact, the artifact is in turn containerised and stored to Dockerhub. Next Helm packages and deploys the setup on a kubernetes cluster.

## Architecture
![](architecture)

## Prereqs
 - Project 5: Continues integration of a Java webapp
 - Project 14: Web app on kubernetes cluster
 - Dockerhub, Github and Slack accounts
 

## Steps

1. Jenkins, Sonar and Docker Integration
    The Jenkins server used in Projects 5 and 14 will be used here. Make sure the servers are up and running: Jenkins server, Sonarserver and kops. 
    - Make sure Jenkins security group allows all traffic from sonar security group and vice versa
    - Get Dockerhub credentials and set them in Jenkins: manage Jenkins -> manage credentials -> Add credentials
    ```
        kind: username with password
        username: login username for dockerhub 
        password: login password
        id: dockerhub
        description: dockerhub
    ```

    - SSH into jenkins server and install docker
    ```
        # Add Docker's official GPG key:
        sudo apt-get update
        sudo apt-get install ca-certificates curl gnupg
        sudo install -m 0755 -d /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        sudo chmod a+r /etc/apt/keyrings/docker.gpg

        # Add the repository to Apt sources:
        echo \
        "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
        "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
        sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        sudo apt-get update
        sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

    ```
    Note: verify Docker documentation to be sure to use updated commands to install Docker

    Add Jenkins to docker user group
    ```
        sudo -i
        usermod -aG docker jenkins
        reboot
    ```
2. Plugins, kubernetes cluster and Helm

    - Back to the browser and login into jenkins and install plugins: manage jenkins -> plugins -> available
    ```
        Docker pipeline
        Docker
        pipeline utility steps
         -> install
    ```

    - SSH into kops server and create the kubernetes cluster
    ```
        kops create cluster --name=ndzenyuyjones.link --state=s3://vprofile-kops1245 --zones=us-east-2a,us-east-2b --node-count=2 --node-size=t3.small --master-size=t3.medium --dns-zone=ndzenyuyjones.link --node-volume-size=8 --master-volume-size=8

        kops update cluster --name ndzenyuyjones.link --state=s3://vprofile-kops1245 --yes --admin

        kops validate cluster --state=s3://vprofile-kops1245 --wait 10m
    ```
    Note: use your domain name for the flag --name, and your created s3 bucket for --state

    ![](kops clusters validation)

    - Install Helm: 
        ```
        cd /tmp/
        wget https://get.helm.sh/helm-v3.13.0-linux-amd64.tar.gz
        tar xzvf helm-v3.13.0-linux-amd64.tar.gz
        cd linux-amd64
        sudo mv helm /usr/local/bin/helm
        ```


3. Helm Charts and git repo Setup
    - Prepare our repository: Create a github public repository
    ```
       name: cicd-kube-docker
       public repository = true
    ```
    clone it to kops machine: Copy the Github repository url (you can either work on local machine and push to git hub, from kops machine clone the github repo)
    ```
        git clone <your repository url>
    ```
    clone this other url seperately, in another directory and prepare workspace 
    ```
        git clone https://github.com/Ndzenyuy/vprofile-project.git
        cd vprofile-project
        git checkout vp-docker
        cp -r * ../cicd-kube-docker/
        cd ../cicd-kube-docker/
        rm -rf Docker-db Docker-web compose ansible
        mv Docker-app/Dockerfile .
        rm -rf Docker-app helm
    ```

    create a helm workspace
    ```
    mkdir helm
    cd helm
    helm create vprofilecharts
    cd vprofilecharts/templates
    rm -rf *
    cd ../../../
    cp kubernetes/vpro-app/* helm/vprofilecharts/templates/
    cd helm/vprofilecharts/templates
    nano vproappdep.yml    
    ```

    update the contents of vproappdep.yml file to be
    ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: vproapp
    labels: 
        app: vproapp
    spec:
    replicas: 1
    selector:
        matchLabels:
        app: vproapp
    template:
        metadata:
        labels:
            app: vproapp
        spec:
        containers:
        - name: vproapp
            image: {{ .Values.appimage }}
            ports:
            - name: vproapp-port
            containerPort: 8080
        initContainers:
        - name: init-mydb
            image: busybox
            command: ['sh', '-c', 'until nslookup vprodb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done;']
        - name: init-memcache
            image: busybox
            command: ['sh', '-c', 'until nslookup vprocache01.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done;']
    ```

4. Pipeline Code
    In the project directory, create a Jenkinsfile with the following contents
    ```
        pipeline {

        agent any

        tools {
            maven "MAVEN3"
            jdk "OracleJDK8"
        }

        environment {
            registry = "ndzenyuy/vprofileapp"
            registryCredential = 'dockerhub'
        }

        stages{

            stage('BUILD'){
                steps {
                    sh 'mvn clean install -DskipTests'
                }
                post {
                    success {
                        echo 'Now Archiving...'
                        archiveArtifacts artifacts: '**/target/*.war'
                    }
                }
            }

            stage('UNIT TEST'){
                steps {
                    sh 'mvn test'
                }
            }

            stage('INTEGRATION TEST'){
                steps {
                    sh 'mvn verify -DskipUnitTests'
                }
            }

            stage ('CODE ANALYSIS WITH CHECKSTYLE'){
                steps {
                    sh 'mvn checkstyle:checkstyle'
                }
                post {
                    success {
                        echo 'Generated Analysis Result'
                    }
                }
            }


            stage('Building image') {
                steps{
                script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                }
                }
            }
            
            stage('Deploy Image') {
            steps{
                script {
                docker.withRegistry( '', registryCredential ) {
                    dockerImage.push("$BUILD_NUMBER")
                    dockerImage.push('latest')
                }
                }
            }
            }

            stage('Remove Unused docker image') {
            steps{
                sh "docker rmi $registry:$BUILD_NUMBER"
            }
            }

            stage('CODE ANALYSIS with SONARQUBE') {

                environment {
                    scannerHome = tool 'mysonarscanner4'
                }

                steps {
                    withSonarQubeEnv('sonar-pro') {
                        sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile-repo \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                    }

                    timeout(time: 10, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }

            stage ('Build App Image'){
                steps{
                    script{
                        dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                    }
                }
            }
            
            stage('Upload Image'){
                steps{
                    script {
                        docker.withRegistry('', registryCredential) {
                            dockerImage.push("V$BUILD_NUMBER")
                            dockerImage.push('latest')
                        }
                    }
                }
            }

            stage('Remove unused docker image') {
                steps{
                    sh "docker rmi $registry:V$BUILD_NUMBER"
                }
            }

            stage('Kubernetes Deploy') {
                agent {label 'KOPS'}
                steps {
                        sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
                }
            }
        }


    }

    ```
         Commit and push to git hub \

    - SSH into kops server
    ```
    mkdir jenkins-slave
    sudo apt install openjdk-11-jdk -y
    sudo mkdir /opt/jenkins-slave
    sudo chown ubuntu.ubuntu /opt/jenkins-slave -R  

    # Create prod namespace where the 
    kubectl create namespace prod
    ```

    - To the AWS console change inbound rules for the kops security group
    ```
    allow SSH(22) traffic from the jenkins sg
    ```

    - On to Jenkins dashboard: manage jenkins -> manage nodes -> new node
    ```
    node name: kops
     -> next
     Remote root directory: /opt/jenkins-slave
     Labels: KOPS
     Usage: Only build jobs with label expression matching this node
     Launch method: Launch agents via SSH
     Host: <private ip of the kops ec2 instance>
     credentials: -> add new jenkins
        -> name: kops-login
        -> Description: kops-login
        -> kind SSH username with private key
        -> username: ubuntu
        -> private key: (copy and paste the content of the kops login key pair file)
          -> save
     credentials: kops-login (select the credentials created above)
     save. 
     
     ```

     - Manage Jenkins -> tools -> sonarqube scanner 
     ```
       name = mysonnarscanner4
     ```


     Check the logs, we should see Agent succesfully connected and online
     ![](kops successful login)
     ![](jenkins nodes)
     ![](kops nodes)

5. Execution: Create a new pipeline on Jenkins

    Jenkins -> new Item:
    ```
    - item name = kube-cicd
    - pipeline = true -> ok
    - pipeline:
        Definition: Pipeline script from SCM
        SCM: Git
        Repository url: https://github.com/Ndzenyuy/Project_15-Continues-delivery-of-docker-containers.git (past the url to your repo)
        Branch specifier: */main
        script path: Jenkinsfile
          -> save
    - Build pipeline: Build now

    ```
    ![](pipeline successful build)
    ![](image in dockerhub)

    - SSH into kops terminal to obtain the endpoint to the created load balancer

    ```
    kubectl get svc -o wide -n prod
    ```
    ![](load balancer endpoint in service)

    - Copy the load balancer endpoint and search on the browser
    ![](login page)
    ![](successful login)
    ![]()
    ![]()
    ![]()

