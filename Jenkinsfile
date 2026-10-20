pipeline {
    agent { label 'node-1' }
    
    tools {
        git 'Default'
    }

    environment {
        DOCKER_IMAGE = 'bus57790/devops-lab-app'
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
                    withSonarQubeEnv('SonarQubeServer') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=devops-lab-app"
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                        docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} -t ${DOCKER_IMAGE}:latest .
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-access-token', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh """
                        rm -rf config-repo
                        git clone https://x-access-token:${GH_TOKEN}@${GITHUB_CONFIG_REPO} config-repo
                        cd config-repo
                        
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" deployment.yaml
                        
                        git config user.email "jenkins@lab.local"
                        git config user.name "Jenkins CI"
                        
                        git add deployment.yaml
                        git diff-index --quiet HEAD || git commit -m "Update image to version ${BUILD_NUMBER}"
                        
                        git push https://x-access-token:${GH_TOKEN}@${GITHUB_CONFIG_REPO} HEAD:main
                    """
                }
            }
        }
    }
}
