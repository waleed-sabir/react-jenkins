# Automated CI/CD Pipeline for React Application Deployment Using Jenkins and Docker on Azure VM

This project demonstrates an automated CI/CD pipeline for a React application. The pipeline utilizes Jenkins to build, test, and deploy the application in a Docker Nginx container whenever there is a push commit to the GitHub repository. The Jenkins server is hosted on an Azure Virtual Machine (VM).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup Azure VM](#setup-azure-vm)
  - [Create an Azure VM](#create-an-azure-vm)
  - [Install Jenkins](#install-jenkins)
  - [Install Docker](#install-docker)
- [Setup Jenkins](#setup-jenkins)
  - [Install Necessary Plugins](#install-necessary-plugins)
  - [Install Node.js](#install-nodejs)
  - [Add GitHub Credentials](#add-github-credentials)
  - [Configure Docker Hub Credentials](#configure-docker-hub-credentials)
  - [Create a Jenkins Pipeline Job](#create-a-jenkins-pipeline-job)
- [Configure GitHub Webhook](#configure-github-webhook)
- [Jenkins Pipeline](#jenkins-pipeline)
- [How to Run](#how-to-run)

## Prerequisites

- Azure account
- Jenkins server setup and running on an Azure VM
- Docker installed on the Jenkins server
- GitHub repository containing your React application
- GitHub personal access token with `repo` access
- Docker Hub account

## Setup Azure VM

### Create an Azure VM

1. Go to the Azure portal and create a new VM.
2. Choose the appropriate size and region for your VM.
3. Select an image (e.g., Ubuntu 22.04 LTS).
4. Set up authentication (SSH key or password).
5. Create and attach a new public IP address.
6. Review and create the VM.
7. Configure Network Security Group (NSG) attached with the VM to allow traffic on port 8080 (Jenkins) and 80 (Nginx)

### Install Jenkins

1. SSH into your Azure VM:

   ```sh
   ssh username@your-vm-public-ip

   ```

2. Update the package index and install Jenkins:

   ```sh
    sudo apt update
    sudo apt install fontconfig openjdk-17-jre
    java -version
    sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
    https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
    echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null
    sudo apt-get update
    sudo apt-get install jenkins
   ```

3. Start Jenkins and enable it to start on boot:

```sh
    sudo systemctl start jenkins
    sudo systemctl enable jenkins
```

### Install Docker

1. Update the package index and install Docker:

```sh
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y docker-ce
```

3. Add the Jenkins user to the Docker group:

```sh
    sudo usermod -aG docker jenkins
    sudo systemctl restart jenkins
```

## Setup Jenkins

### Install Necessary Plugins

Ensure the following plugins are installed:

- GitHub Integration Plugin
- GitHub API Plugin
- GitHub Authentication Plugin
- Git Plugin
- NodeJS
- Docker Pipeline

### Install Node.js

1. Go to Manage Jenkins -> Global Tool Configuration.
2. Scroll down to NodeJS.
3. Click Add NodeJS.
4. Enter a name (e.g., NodeJS 14) and select the version you want to install (e.g., NodeJS 14.17.0).
5. Check the box Install automatically.
6. Save the configuration.

### Add GitHub Credentials

1. Go to `Manage Jenkins` -> `Manage Credentials`.
2. Choose the appropriate domain (e.g., `Global`).
3. Click `Add Credentials`.
4. Select `Secret text` as the kind.
5. Paste your GitHub personal access token in the `Secret` field.
6. Give it an ID (e.g., `github-token`).

### Configure Docker Hub Credentials

1. Go to Manage Jenkins -> Manage Credentials.
2. Choose the appropriate domain (e.g., Global).
3. Click Add Credentials.
4. Select Username with password as the kind.
5. Enter your Docker Hub username and personal access token.
6. Give it an ID (e.g., dockerhub-credentials-id).
7. Save the credentials.

### Create a Jenkins Pipeline Job

1. From the Jenkins dashboard, click `New Item`.
2. Enter a name for your job, select `Pipeline`, and click `OK`.
3. In the job configuration, scroll down to the `Pipeline` section.
4. Select `Pipeline script from SCM`.
5. Choose `Git` as the SCM.
6. Enter your repository URL and select the appropriate credentials from the `Credentials` dropdown (the GitHub personal access token you added).
7. Specify the branch to build (e.g., `main`).
8. Specify the `Jenkinsfile` path if it's not in the root directory.
9. In the `Build Triggers` section, check `GitHub hook trigger for GITScm polling`.
10. Save the configuration.

## Configure GitHub Webhook

1. Go to your GitHub repository.
2. Navigate to `Settings` -> `Webhooks` -> `Add webhook`.
3. Enter your Jenkins URL followed by `/github-webhook/` (e.g., `http://your-jenkins-url:8080/github-webhook/`).
4. Choose `application/json` as the Content type.
5. Select `Just the push event` to trigger builds on push events.
6. Click `Add webhook`.

## Jenkins Pipeline

Create a `Jenkinsfile` in the root of your React project:

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "your-dockerhub-username/your-react-app"
        BRANCH_NAME = "main"
        NODEJS_VERSION = "NodeJS 14"
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials-id"
        GITHUB_CREDENTIALS_ID = "github-token"
        HOST_PORT = "8082"
    }

    tools {
        nodejs "${env.NODEJS_VERSION}"
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME}",
                    url: 'https://github.com/your-username/your-repo.git',
                    credentialsId: "${env.GITHUB_CREDENTIALS_ID}"
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('Run Tests') {
            steps {
                sh 'npm test'
            }
        }
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${env.DOCKER_IMAGE}:latest")
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "${env.DOCKER_CREDENTIALS_ID}") {
                        dockerImage.push()
                    }
                }
            }
        }
        stage('Deploy Docker Container') {
            steps {
                script {
                    sh "docker stop react_app || true"
                    sh "docker rm react_app || true"
                    sh "docker run -d -p ${env.HOST_PORT}:80 --name react_app ${env.DOCKER_IMAGE}:latest"
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
```

### How to Run

1. Push your code to the GitHub repository:

```sh
    git add .
    git commit -m "Initial commit"
    git push origin main
```

2. The Jenkins pipeline will automatically trigger on push events, building, testing, and deploying your React application in a Docker Nginx container.

### Explanation

- **Prerequisites**: Includes Azure account as a prerequisite.
- **Setup Azure VM**: New section detailing how to create an Azure VM, install Jenkins, Docker, and configure the firewall.
- **Setup Jenkins**: Ensures necessary plugins are installed and credentials are added.
- **Configure GitHub Webhook**: Details how to set up a webhook in GitHub.
- **Jenkins Pipeline**: Provides a `Jenkinsfile` example for the pipeline.
- **How to Run**: Steps to push code and trigger the Jenkins pipeline.

This `README.md` provides a comprehensive guide for setting up and running the CI/CD pipeline for your React application on an Azure VM using Jenkins and Docker.
