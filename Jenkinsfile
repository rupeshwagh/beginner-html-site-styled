pipeline {
    agent { label 'jenkins-master' }

    environment {
        DOCKER_IMAGE = "rupesh97669/beginner-html-site-styled"
        DOCKER_TAG = "latest"
        K8S_MASTER_IP = "10.10.1.252"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/rupeshwagh/beginner-html-site-styled.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image on Jenkins Master..."
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                '''
            }
        }

        stage('Login to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo "Pushing Docker image to DockerHub..."
                    docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                '''
            }
        }

        stage('Copy Kubernetes Manifests to K8s Master') {
            steps {
                sshagent(credentials: ['k8s-master-ssh']) {
                    sh '''
                        echo "Copying Kubernetes manifests to k8s-master..."
                        ssh -o StrictHostKeyChecking=no ubuntu@${K8S_MASTER_IP} "mkdir -p /home/ubuntu/website-k8s"

                        scp -o StrictHostKeyChecking=no k8s/deployment.yaml ubuntu@${K8S_MASTER_IP}:/home/ubuntu/website-k8s/deployment.yaml

                        scp -o StrictHostKeyChecking=no k8s/service.yaml ubuntu@${K8S_MASTER_IP}:/home/ubuntu/website-k8s/service.yaml
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sshagent(credentials: ['k8s-master-ssh']) {
                    sh '''
                        echo "Deploying application to Kubernetes..."
                        ssh -o StrictHostKeyChecking=no ubuntu@${K8S_MASTER_IP} "
                            kubectl apply -f /home/ubuntu/website-k8s/deployment.yaml &&
                            kubectl apply -f /home/ubuntu/website-k8s/service.yaml &&
                            kubectl rollout restart deployment beginner-html-site &&
                            kubectl rollout status deployment beginner-html-site --timeout=300s || true
                        "
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sshagent(credentials: ['k8s-master-ssh']) {
                    sh '''
                        echo "Checking Kubernetes resources..."
                        ssh -o StrictHostKeyChecking=no ubuntu@${K8S_MASTER_IP} "
                            kubectl get deployments &&
                            kubectl get pods -o wide &&
                            kubectl get svc
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully. Website deployed."
        }

        failure {
            echo "Pipeline failed. Check Jenkins console logs."
        }
    }
}
