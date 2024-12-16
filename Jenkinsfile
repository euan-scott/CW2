pipeline {
    agent any

environment {
    SSH_PRIVATE_KEY = credentials('labsuserCW.pem')  // Correct ID for the SSH key credential
    KUBECONFIG = '/home/ubuntu/.kube/config'  // Path to kubeconfig for production server
}


    stages {
        stage('Checkout Code') {
            steps {
                echo 'Checking out source code from GitHub...'
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "euanscott16/cw2-server:${env.BUILD_NUMBER}" // Unique tag based on build number
                    echo "Building Docker Image with tag: ${imageTag}..."
                    sh "docker build -t ${imageTag} ."
                    sh "docker push ${imageTag}"
                    env.DOCKER_IMAGE = imageTag // Set as environment variable for later stages
                }
            }
        }

        stage('Test Docker Container') {
            steps {
                script {
                    try {
                        echo 'Testing Docker Container...'
                        sh '''
                        docker stop test-container || true
                        docker rm test-container || true
                        docker run --rm --name test-container -d -p 8083:8080 ${DOCKER_IMAGE}
                        sleep 10
                        CONTAINER_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' test-container)
                        curl -f http://$CONTAINER_IP:8080
                        '''
                        echo 'Test passed: Container is responding!'
                    } catch (Exception e) {
                        echo 'Test failed: Container not responding. Fetching logs...'
                        sh '''
                        docker logs test-container || true
                        '''
                        error 'Container test failed!'
                    }
                }
            }
        }

        stage('Set up SSH Tunnel to Production') {
            steps {
                script {
                    echo 'Setting up SSH tunnel to the production server...'
                    // Set up SSH tunnel to forward local port 8081 to production server's 8080 port
                    sh """
                        ssh -i ${SSH_PRIVATE_KEY} -N -L 8081:localhost:8080 ubuntu@ec2-3-80-221-91.compute-1.amazonaws.com &
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                echo 'Pushing Docker Image to DockerHub...'
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh '''
                        echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin
                        docker push ${DOCKER_IMAGE}
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo 'Deploying to Kubernetes...'
                    // Set Kubernetes context to production
                    sh "kubectl config use-context minikube" // Use the correct Kubernetes context
                    // Deploy the application to the Kubernetes cluster
                    sh "kubectl set image deployment/cw2-app cw2-server=${DOCKER_IMAGE} --record"
                    // Ensure the deployment is rolled out successfully
                    sh "kubectl rollout status deployment/cw2-app"
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed. Please check the logs.'
        }
    }
}
