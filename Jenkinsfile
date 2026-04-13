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
                sh '''
                docker build -t vote ./vote
                docker build -t result ./result
                docker build -t worker ./worker
                '''
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
                withCredentials([[
                  $class: 'AmazonWebServicesCredentialsBinding',
                  credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ECR_REPO
                    '''
                }
            }
        }

        stage('Push Images to ECR') {
            steps {
                withCredentials([[
                  $class: 'AmazonWebServicesCredentialsBinding',
                  credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    docker push $ECR_REPO/vote:latest
                    docker push $ECR_REPO/result:latest
                    docker push $ECR_REPO/worker:latest
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                  $class: 'AmazonWebServicesCredentialsBinding',
                  credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    aws sts get-caller-identity

                    aws eks update-kubeconfig \
                    --region $AWS_REGION \
                    --name $CLUSTER_NAME

                    # ✅ Step 1: Create namespace FIRST
                    kubectl apply -f k8s-specifications/namespace.yaml

                    # ✅ Step 2: Wait for namespace to be ready
                    sleep 10

                    # ✅ Step 3: Deploy remaining resources
                    kubectl apply -f k8s-specifications/
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([[
                  $class: 'AmazonWebServicesCredentialsBinding',
                  credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    kubectl get pods -n voting-app
                    kubectl get svc -n voting-app
                    '''
                }
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