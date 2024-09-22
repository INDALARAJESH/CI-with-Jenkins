pipeline {
    agent any

    environment {
        NAME = "spring-app"
        VERSION = "${env.BUILD_ID}"
        IMAGE_REPO = "indalarajesh"
        GIT_REPO_NAME = "DevOps_MasterPiece-CD-with-argocd"
        GIT_USER_NAME = "INDALARAJESH"
        KUBE_CONFIG = "/path/to/kubeconfig" // Ensure the path to your kubeconfig file is correct
    }

    tools { 
        maven 'maven-3.8.6' 
    }

    stages {
        stage('Checkout git') {
            steps {
                git branch: 'main', url: 'https://github.com/indalarajesh/CI-with-Jenkins.git'
            }
        }

        stage('Get Git Commit Hash') {
            steps {
                script {
                    env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                }
            }
        }
        
        stage('Build & JUnit Test') {
            steps {
                sh 'mvn clean install'
            }
        }
        
        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT} .'
            }
        }
        
        stage('Docker Image Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-credentials-id', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                    sh 'docker push ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT}'
                    sh 'docker rmi ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT}'
                }
            }
        }
        
        stage('Clone/Pull k8s deployment Repo') {
            steps {
                script {
                    if (fileExists('DevOps_MasterPiece-CD-with-argocd')) {
                        echo 'Cloned repo already exists - Pulling latest changes'
                        dir("DevOps_MasterPiece-CD-with-argocd") {
                            sh 'git pull'
                        }
                    } else {
                        echo 'Repo does not exist - Cloning the repo'
                        sh 'git clone -b feature https://github.com/indalarajesh/DevOps_MasterPiece-CD-with-argocd.git'
                    }
                }
            }
        }
        
        stage('Update deployment Manifest') {
            steps {
                dir("DevOps_MasterPiece-CD-with-argocd/yamls") {
                    sh 'sed -i "s#indalarajesh.*#${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT}#g" deployment.yaml'
                    sh 'cat deployment.yaml'
                }
            }
        }
        
        stage('Commit & Push changes to feature branch') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    dir("DevOps_MasterPiece-CD-with-argocd/yamls") {
                        sh "git config --global user.email 'rajeshindala1997@gmail.com'"
                        sh "git config --global user.name 'INDALARAJESH'"
                        sh 'git remote set-url origin https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}'
                        sh 'git checkout feature'
                        sh 'git add deployment.yaml'
                        sh "git commit -am 'Updated image version for Build- ${VERSION}-${GIT_COMMIT}'"
                        sh 'git push origin feature'
                    }
                }
            }
        }
        
        stage('Raise PR') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    dir("DevOps_MasterPiece-CD-with-argocd/yamls") {
                        // Pass the GITHUB_TOKEN directly to gh auth login
                        sh '''
                            unset GITHUB_TOKEN
                            echo "${GITHUB_TOKEN}" | gh auth login --with-token
                        '''
                        sh 'git checkout feature'
                        sh "gh pr create -t 'image tag updated' -b 'check and merge it'"
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    // Configure kubectl to use your EKS cluster
                    sh 'aws eks --region YOUR_REGION update-kubeconfig --name YOUR_CLUSTER_NAME'

                    // Deploy to EKS using kubectl
                    sh 'kubectl apply -f DevOps_MasterPiece-CD-with-argocd/yamls/deployment.yaml'
                }
            }
        }
    }
}
