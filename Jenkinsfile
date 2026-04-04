pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
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
                        set -e

                        echo "🐳 Building Docker Image..."
                        docker build -t $IMAGE_NAME:$IMAGE_TAG .

                        echo "🔐 Logging into DockerHub..."
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        echo "📤 Pushing Image..."
                        docker push $IMAGE_NAME:$IMAGE_TAG

                        echo "🧹 Cleaning old images..."
                        docker image prune -f
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

                        echo "🔧 Configuring Git..."
                        git config user.name "$GIT_USER"
                        git config user.email "$GIT_EMAIL"

                        git fetch origin
                        git checkout main
                        git reset --hard origin/main

                        echo "✏️ Updating image in deployment.yaml..."
                        sed -i "s|image:.*|image: $IMAGE_NAME:$IMAGE_TAG|" k8s/deployment.yml

                        git add k8s/deployment.yml

                        if git diff --cached --quiet; then
                            echo "No changes to commit"
                        else
                            git commit -m "Updated image to $IMAGE_TAG"
                            git push https://$GIT_USERNAME:$GIT_TOKEN@github.com/$GIT_USERNAME/Main-Branch-Code.git main
                        fi
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS') {
    when { branch 'main' }
    steps {
        script {
            withCredentials([usernamePassword(
                credentialsId: 'aws-creds',
                usernameVariable: 'AWS_ACCESS_KEY_ID',
                passwordVariable: 'AWS_SECRET_ACCESS_KEY'
            )]) {
                sh '''
                set -e

                export AWS_DEFAULT_REGION=$AWS_REGION

                echo "🔍 Checking AWS Identity..."
                aws sts get-caller-identity

                echo "🔄 Updating kubeconfig..."
                aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER

                echo "⏳ Waiting for cluster..."
                sleep 10

                echo "🧪 Testing Kubernetes access..."
                kubectl get nodes

                echo "🚀 Deploying to Kubernetes..."
                kubectl apply -f k8s/deployment.yml
                kubectl apply -f k8s/service.yml

                echo "📊 Checking rollout..."
                kubectl rollout status deployment/flask-app
                '''
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
        always {
            echo "📦 Build Number: ${BUILD_NUMBER}"
        }
    }
}
