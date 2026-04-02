pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "swach/multibranch-flask-app"
        DOCKER_TAG = "build-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Push Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {

                        sh """
                        docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push $DOCKER_IMAGE:$DOCKER_TAG
                        """
                    }
                }
            }
        }

        stage('Update K8s Manifest') {
    steps {
        script {
            withCredentials([string(credentialsId: 'github-token', variable: 'GIT_TOKEN')]) {
                sh '''
                set -e

                git config user.name "swachand"
                git config user.email "your-email@example.com"

                git fetch origin
                git checkout main
                git reset --hard origin/main

                sed -i "s|image:.*|image: swach/multibranch-flask-app:${BUILD_TAG}|" k8s/deployment.yml

                git add k8s/deployment.yml
                git commit -m "Updated image to ${BUILD_TAG}"

                git push https://${GIT_TOKEN}@github.com/swachand/Main-Branch-Code.git main
                '''
            }
        }
    }
}
        // 🔥 OPTIONAL (Next Step - EKS Deployment)
        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {

                        sh """
                        export KUBECONFIG=$KUBECONFIG_FILE

                        kubectl apply -f k8s/deployment.yml
                        kubectl apply -f k8s/service.yml

                        kubectl get pods
                        """
                    }
                }
            }
        }
    }
}
