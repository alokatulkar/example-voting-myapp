pipeline {
    agent any

    environment {
        AWS_REGION     = "ap-south-1"
        AWS_ACCOUNT_ID = "623900187979"
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        VOTE_IMAGE   = "${ECR_REGISTRY}/vote"
        RESULT_IMAGE = "${ECR_REGISTRY}/result"
        WORKER_IMAGE = "${ECR_REGISTRY}/worker"

        IMAGE_TAG    = "${BUILD_NUMBER}"

        CLUSTER_NAME = "ekscluster"
        NAMESPACE    = "voting-app"
    }

    stages {

        stage('Verify Environment') {
            steps {
                sh '''
                echo "===== Environment ====="
                whoami
                pwd
                docker --version
                aws --version
                kubectl version --client

                echo "===== AWS Identity ====="
                aws sts get-caller-identity

                echo "===== Repository ====="
                ls -ltr
                '''
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | \
                docker login \
                  --username AWS \
                  --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Build Docker Images') {
            parallel {

                stage('Vote Image') {
                    steps {
                        dir('vote') {
                            sh '''
                            docker build -t vote:$IMAGE_TAG .
                            docker tag vote:$IMAGE_TAG $VOTE_IMAGE:$IMAGE_TAG
                            '''
                        }
                    }
                }

                stage('Result Image') {
                    steps {
                        dir('result') {
                            sh '''
                            docker build -t result:$IMAGE_TAG .
                            docker tag result:$IMAGE_TAG $RESULT_IMAGE:$IMAGE_TAG
                            '''
                        }
                    }
                }

                stage('Worker Image') {
                    steps {
                        dir('worker') {
                            sh '''
                            docker build -t worker:$IMAGE_TAG .
                            docker tag worker:$IMAGE_TAG $WORKER_IMAGE:$IMAGE_TAG
                            '''
                        }
                    }
                }
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                docker push $VOTE_IMAGE:$IMAGE_TAG
                docker push $RESULT_IMAGE:$IMAGE_TAG
                docker push $WORKER_IMAGE:$IMAGE_TAG
                '''
            }
        }

        stage('Configure kubectl') {
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

                kubectl set image deployment/vote \
                    vote=$VOTE_IMAGE:$IMAGE_TAG \
                    -n $NAMESPACE

                kubectl set image deployment/result \
                    result=$RESULT_IMAGE:$IMAGE_TAG \
                    -n $NAMESPACE

                kubectl set image deployment/worker \
                    worker=$WORKER_IMAGE:$IMAGE_TAG \
                    -n $NAMESPACE
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                kubectl rollout status deployment/vote -n $NAMESPACE
                kubectl rollout status deployment/result -n $NAMESPACE
                kubectl rollout status deployment/worker -n $NAMESPACE

                kubectl get pods -n $NAMESPACE
                kubectl get svc -n $NAMESPACE
                '''
            }
        }
    }

    post {

        success {
            echo "==================================="
            echo "Deployment Completed Successfully"
            echo "==================================="
        }

        failure {
            echo "==================================="
            echo "Deployment Failed"
            echo "==================================="
        }

        always {
            sh '''
            docker image prune -af || true
            '''
        }
    }
}
