pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME  = "swach/multibranch-flask-app"
        GIT_USER    = "swachand"
        GIT_EMAIL   = "your-email@example.com"
        AWS_REGION  = "us-east-1"
        EKS_CLUSTER = "kastro-cluster"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Push Image') {
            when { branch 'main' }
            steps {
                script {
                    env.IMAGE_TAG = "build-${BUILD_NUMBER}"

                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                        docker build -t $IMAGE_NAME:$IMAGE_TAG .
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE_NAME:$IMAGE_TAG
                        '''
                    }
                }
            }
        }

        stage('Update K8s Manifest') {
            when { branch 'main' }
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'github-creds',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        sh '''
                        set -e
                        git config user.name "$GIT_USER"
                        git config user.email "$GIT_EMAIL"

                        git fetch origin
                        git checkout main
                        git reset --hard origin/main

                        sed -i "s|image:.*|image: $IMAGE_NAME:$IMAGE_TAG|" k8s/deployment.yml

                        git add k8s/deployment.yml
                        git diff --cached --quiet || git commit -m "Updated image to $IMAGE_TAG"
                        git push https://$GIT_USERNAME:$GIT_TOKEN@github.com/swachand/Main-Branch-Code.git main
                        '''
                    }
                }
            }
        }

stage('Deploy to EKS') {
    steps {
        script {
            withCredentials([[
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws-creds'
            ]]) {
                sh '''
                set -e

                export AWS_DEFAULT_REGION=us-east-1

                echo "🔍 Checking AWS Identity..."
                aws sts get-caller-identity

                echo "🔄 Updating kubeconfig..."
                aws eks update-kubeconfig --region us-east-1 --name kastro-cluster

                echo "🧪 Testing Kubernetes access..."
                kubectl get nodes

                echo "🚀 Deploying to Kubernetes..."
                kubectl apply -f k8s/deployment.yml
                kubectl apply -f k8s/service.yml
                '''
            }
        }
    }   
}

    post {
        success {
            echo "✅ Deployment Successful!"
        }
        failure {
            echo "❌ Pipeline Failed!"
        }
    }
}
