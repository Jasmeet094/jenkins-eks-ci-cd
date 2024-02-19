


# In this Project we are creating a CI-CD pipeline in which we are using Jenkins , sonarQube, ArgoCd , Helm and Kubernetes running on AWS EKS. {Insipred by Abhishek Veeramalla DevOps Tutorials}


### Description

 In world of DevOps every devops engineer must know the concepts of Continous Integration and Continous Deployment. In this Project i am going to use CiCd concepts. This is a simple Springboot Java application which will be deploy on AWS EKS. I am using GitHub as SCM. Jenkins is used here to build the code , triggered some tests and then SonarQUBE will also be used.


### > Continous Integration Part Explained :
In the continous integration pipeline we are storing our code in Github Repo called Ci-Cd-app. Then we setup Jenkins server on ec2 which is intergrated with the SCM github using webhooks. We are using Maven to build code and Docker containers as Jenkins Agents. So i am using a image `abhishekf5/maven-abhishek-docker-agent:v1` which have maven and docker installed by default. We are using sonarqube which is deployed on the same instance on which jenkins is deployed.


### > Continous Deployment Part Explained :
asdcsfdvfgbdgf
asdcadscdsacadscas



### Deploy Jenkins Server on AWS EC2 using ubuntu AMI and choose type t2.large
#### 1. Install java 11


```
sudo apt update
sudo apt install openjdk-11-jre
```

#### 2. Add required keys and Install Jenkins
```
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

#### 3. Check if Jenkins is running and get the admin password to login to Jenkins
`ps -aux | grep jenkins`


` sudo service jenkins status`

`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`


#### 4. Install Docker Pipeline Plugin 
`Dashboard > manage jenkins > plugins > Available Plugins > docker pipeline > install`

#### 5. Install SonarQube  Plugin 
`Dashboard > manage jenkins > plugins > Available Plugins > SonarQube Scanner > install`

#### 6. Provide permissions to Docker and Jenkins Users for Docker Daemon
```
sudo su - 
usermod -aG docker jenkins
usermod -aG docker ubuntu
systemctl restart docker
systemctl restart jenkins
``` 

### Deploy SonarQube Server on same AWS EC2 on which our jenkins server is deployed.
#### 1. Install SonarQube

```
1. apt install unzip
2. adduser sonarqube
3. sudo su sonarqube && cd /home/sonarqube 
4. wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
5. unzip *
6. chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
7. chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
8. cd sonarqube-9.4.0.54424/bin/linux-x86-64/
9. ./sonar.sh start
10. Login to sonaqube using http://ip_address_ec2:9000
11. Username and Password is admin-admin , update the password as requirement

```
#### 2. Itegrate SonarQube with Jenkins.
    Now SonarQube also  need to intergrate withe jenkins so it can fetch the info from jenkins.

```
1. Login to sonarqube and defualt password is `admin` and set up new password.
2. Go to my account on right top side then click in security and generate a token with name Jenkins. it will generate a random string.
3. Now login to jenkins server and > Dashboard > manage jenkins > credentials > system > global creadentials > add credentials
4. Select kind secrets txt and under secret put the generated token. under ID put `Sonarqube` as name.  

```
### Deploy AWS EKS Cluster.
#### 1. Create EKS Cluster
`eksctl create cluster --name test-demo-eks --region us-east-1 --nodegroup-name test-ng --node-type t3.micro --nodes 2 --node-private-networking --managed`

#### 2. Deploy ArgoCd CRDs

```
1. Go to operatorhub.io and search for ArgoCd operator and install as per the instructions given
```


### Jenkinsfile Explained
```
pipeline {
  agent {
    docker {
      image 'abhishekf5/maven-abhishek-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock' // mount Docker socket to access the host's Docker daemon
    }
  }

  # This stage us checking out to the our github repo. If you dont want to add jenkinsfile in your SCM then we need to add this stage if your jenkins file is already in github repo then we can skip this stage.

  stages { 
    stage('Checkout') {
      steps {
        sh 'echo passed'
        //git branch: 'main', url: 'https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero.git'
      }
    }

    # this stage is building our application. It will follow up pom.xml file and install the required dependencies for this java app and maven will create the app. After this command there will be a folder get created in root dorectory calledd `target` and in this folder maven will create a spring-boot-web.jar file. In Dockerfile we are copying this file to /opt/app folder

    stage('Build and Test') {
      steps {
        sh 'ls -ltr'
        // build the project and create a JAR file
        sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn clean package'
      }
    }

    # this stage for sonarqube to get infor from jenkins
    stage('Static Code Analysis') {
      environment {
        SONAR_URL = "http://34.201.116.83:9000"
      }
      steps {
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
          sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
        }
      }
    }

    # here we are building the image and pushing it to docker hub
    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "jasmeesingh94/jenkins-ci-cd:${BUILD_NUMBER}"
        // DOCKERFILE_LOCATION = "java-maven-sonar-argocd-helm-k8s/spring-boot-app/Dockerfile"
        REGISTRY_CREDENTIALS = credentials('docker-cred')
      }
      steps {
        script {
            sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
            def dockerImage = docker.image("${DOCKER_IMAGE}")
            docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                dockerImage.push()
            }
        }
      }
    }

    # In this stage we are changing the tag of the image in out deployment file and pushes back the changes to github
    stage('Update Deployment File') {
        environment {
            GIT_REPO_NAME = "jenkins-eks-ci-cd"
            GIT_USER_NAME = "Jasmeet094"
        }
        steps {
            withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                sh '''
                    git config user.email "jasmeetpabla4u@gmail.com"
                    git config user.name "Jasmeet094"
                    BUILD_NUMBER=${BUILD_NUMBER}
                    sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" springboot_app/springboot_deployment_file/deployment.yml
                    git add springboot_app/springboot_deployment_file/deployment.yml
                    git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                    git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                '''
            }
        }
    }
  }
}

```

#### Create Secret in Jenkins
```
1. In out jenkinsfile we are using 2 credentials. First is to login to docker hub and other is to use for github itself so that we can commit the changes to our repo. these changes are when we change the image tag of our deployment file then we need to push the changes.

2. Create secret in jenkins Dashboard > manage jenkins > credentials > system > global creadentials > add credentials  > secret type is username and password and provide the docker username and password and ID should be docker-cred

3. Create github secret Dashboard > manage jenkins > credentials > system > global creadentials > add credentials > secret type is secret txt and add the github token and use the name of secret as `github`


```


### Install ArgoCd controller

```
1. Go to https://argocd-operator.readthedocs.io/en/latest/usage/basics/#:~:text=The%20following%20example%20shows%20the%20most%20minimal%20valid%20manifest%20to%20create%20a%20new%20Argo%20CD%20cluster%20with%20the%20default%20configuration.

2. Copy the content as it is and create the resource on eks cluster.

3. After pods are in running state , we need to edit the service `example-argocd-server` and changes its type from CLusterIp to Loadbalancer. Make sure the security group of your instances is allowing all traffic from 0.0.0.0 . 

4. There will be a classic load balancer deployed , grab the dns name of lb and now you can access the argocd server.

5. Now Get the password using command
  k get secret example-argocd-cluster -o yaml 

6. Username is `admin` and you can decode the secret 
   echo "VmNwaVBtWTNaQkF3RHNLNlMyOEU5RmdubElIclVrUnY=" | base64 --decode  

```



