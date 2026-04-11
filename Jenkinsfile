pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = '612915905322.dkr.ecr.ap-south-1.amazonaws.com'
        CLUSTER_NAME = 'ekscluster'
    }

    stages {


        stage('Build Docker Images') {
            steps {
                script {
                    sh '''
                    docker build -t vote ./vote
                    docker build -t result ./result
                    docker build -t worker ./worker
                    '''
                }
            }
        }

        stage('Tag Images for ECR') {
            steps {
                sh '''
                docker tag vote $ECR_REPO/vote:latest
                docker tag result $ECR_REPO/result:latest
                docker tag worker $ECR_REPO/worker:latest
                '''
            }
        }

        stage('Login to AWS ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | \
                docker login --username AWS --password-stdin $ECR_REPO
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                docker push $ECR_REPO/vote:latest
                docker push $ECR_REPO/result:latest
                docker push $ECR_REPO/worker:latest
                '''
            }
        }

        stage('Update kubeconfig') {
            steps {
                sh '''
                aws eks update-kubeconfig \
                --region $AWS_REGION \
                --name $CLUSTER_NAME
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                kubectl apply -f k8s-specifications/
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                kubectl get pods
                kubectl get svc
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment Successful 🚀'
        }
        failure {
            echo 'Deployment Failed ❌'
        }
    }
}