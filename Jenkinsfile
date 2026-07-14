pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        AWS_ACCOUNT_ID = "623900187979"

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        VOTE_IMAGE   = "${ECR_REGISTRY}/vote"
        RESULT_IMAGE = "${ECR_REGISTRY}/result"
        WORKER_IMAGE = "${ECR_REGISTRY}/worker"

        IMAGE_TAG = "${BUILD_NUMBER}"

        CLUSTER_NAME = "ekscluster"
        NAMESPACE = "voting-app"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                url: '<your-github-repo-url>'
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                docker --version
                aws --version
                kubectl version --client
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | \
                docker login \
                --username AWS \
                --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Build Images') {
            parallel {

                stage('Vote') {
                    steps {
                        dir('vote') {
                            sh """
                            docker build -t vote:${IMAGE_TAG} .
                            docker tag vote:${IMAGE_TAG} ${VOTE_IMAGE}:${IMAGE_TAG}
                            """
                        }
                    }
                }

                stage('Result') {
                    steps {
                        dir('result') {
                            sh """
                            docker build -t result:${IMAGE_TAG} .
                            docker tag result:${IMAGE_TAG} ${RESULT_IMAGE}:${IMAGE_TAG}
                            """
                        }
                    }
                }

                stage('Worker') {
                    steps {
                        dir('worker') {
                            sh """
                            docker build -t worker:${IMAGE_TAG} .
                            docker tag worker:${IMAGE_TAG} ${WORKER_IMAGE}:${IMAGE_TAG}
                            """
                        }
                    }
                }

            }
        }

        stage('Push Images') {
            steps {
                sh """
                docker push ${VOTE_IMAGE}:${IMAGE_TAG}
                docker push ${RESULT_IMAGE}:${IMAGE_TAG}
                docker push ${WORKER_IMAGE}:${IMAGE_TAG}
                """
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

        stage('Deploy') {
            steps {
                sh '''
                kubectl apply -f k8s-specifications/

                kubectl set image deployment/vote \
                    vote=${VOTE_IMAGE}:${IMAGE_TAG} \
                    -n ${NAMESPACE}

                kubectl set image deployment/result \
                    result=${RESULT_IMAGE}:${IMAGE_TAG} \
                    -n ${NAMESPACE}

                kubectl set image deployment/worker \
                    worker=${WORKER_IMAGE}:${IMAGE_TAG} \
                    -n ${NAMESPACE}
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                kubectl rollout status deployment/vote -n ${NAMESPACE}
                kubectl rollout status deployment/result -n ${NAMESPACE}
                kubectl rollout status deployment/worker -n ${NAMESPACE}

                kubectl get pods -n ${NAMESPACE}
                kubectl get svc -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment Successful"
        }

        failure {
            echo "Deployment Failed"
        }

        always {
            sh 'docker image prune -af || true'
        }
    }
}
