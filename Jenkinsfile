pipeline {
    agent { label 'node-1' }
    
    tools {
        git 'Default'
    }

    environment {
        DOCKER_IMAGE = 'bus57790/devops-lab-app'
        // Store only the domain/path without the scheme for clean token authentication
        GITHUB_CONFIG_REPO = 'github.com/bus57790/devops-gitops-config.git'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=devops-lab-app"
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                        def customImage = docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                        customImage.push()
                        customImage.push('latest')
                    }
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-access-token', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    // Using double quotes (") allows Groovy to interpolate env variables
                    sh """
                        rm -rf config-repo
                        git clone https://${GH_TOKEN}@${GITHUB_CONFIG_REPO} config-repo
                        cd config-repo
                        
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" deployment.yaml
                        
                        git config user.email "jenkins@lab.local"
                        git config user.name "Jenkins CI"
                        
                        git add deployment.yaml
                        git diff-index --quiet HEAD || git commit -m "Update image to version ${BUILD_NUMBER}"
                        git push origin main
                    """
                }
            }
        }
    }
}
