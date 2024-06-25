pipeline {
    agent any

    // Define environment variables
    environment {
        DOCKER_IMAGE = "wsabir/react-jenkins" // Replace with your Docker Hub username and app name
        BRANCH_NAME = "main" // Replace with the branch you want to check out
        NODEJS_VERSION = "NodeJS" // Replace with your NodeJS installation name
        DOCKER_CREDENTIALS_ID = "dockerhub-wsabir" // Replace with your Docker Hub credentials ID
        GITHUB_REPO_URL = "https://github.com/waleed-sabir/react-jenkins.git"
        HOST_PORT = "8082"
        GITHUB_CREDENTIALS = "github-creds"
    }

    tools {
        nodejs "${env.NODEJS_VERSION}" // Use the environment variable for NodeJS version
    }

    // triggers {
    //     githubPush() // This line enables the GitHub webhook trigger
    // }

    stages {
        stage('Checkout') {
            steps {
                // Use the environment variable for branch name
                git branch: "${env.BRANCH_NAME}", url: "${env.GITHUB_REPO_URL}", credentialsId: "${GITHUB_CREDENTIALS}" // Replace with your repository URL
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
                    dockerImage = docker.build("${env.DOCKER_IMAGE}:${env.BUILD_ID}")
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
                    sh "docker run -d -p ${env.HOST_PORT}:80 --name react_app ${env.DOCKER_IMAGE}:${env.BUILD_ID}"
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
