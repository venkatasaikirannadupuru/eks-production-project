pipeline {

    agent any

    environment {
        AWS_REGION = 'ap-south-1'

        PROJECT_NAME = 'eks-production'

        ECR_REPOSITORY = 'eks-production-app'

        EKS_CLUSTER_NAME = 'eks-production-cluster'

        K8S_NAMESPACE = 'production'

        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo '========== CHECKOUT =========='

                checkout scm
            }
        }

        stage('Terraform Init') {
            steps {
                echo '========== TERRAFORM INIT =========='

                dir('terraform') {
                    sh '''
                        terraform init
                    '''
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                echo '========== TERRAFORM APPLY =========='

                dir('terraform') {
                    sh '''
                        terraform apply -auto-approve
                    '''
                }
            }
        }

        stage('Get ECR Repository') {
            steps {
                echo '========== GET ECR REPOSITORY =========='

                script {
                    env.ECR_REPOSITORY_URL = sh(
                        script: '''
                            cd terraform
                            terraform output -raw ecr_repository_url
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "ECR Repository: ${env.ECR_REPOSITORY_URL}"
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo '========== DOCKER BUILD =========='

                dir('app') {
                    sh """
                        docker build \
                          -t ${ECR_REPOSITORY}:${IMAGE_TAG} \
                          .
                    """
                }
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                echo '========== ECR LOGIN =========='

                sh """
                    aws ecr get-login-password \
                      --region ${AWS_REGION} \
                    | docker login \
                      --username AWS \
                      --password-stdin ${ECR_REPOSITORY_URL}
                """
            }
        }

        stage('Tag and Push Docker Image') {
            steps {
                echo '========== PUSH IMAGE TO ECR =========='

                sh """
                    docker tag \
                      ${ECR_REPOSITORY}:${IMAGE_TAG} \
                      ${ECR_REPOSITORY_URL}:${IMAGE_TAG}

                    docker push \
                      ${ECR_REPOSITORY_URL}:${IMAGE_TAG}
                """
            }
        }

        stage('Configure kubectl') {
            steps {
                echo '========== CONFIGURE KUBECTL =========='

                sh """
                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER_NAME}

                    kubectl cluster-info
                """
            }
        }

        stage('Deploy Kubernetes Application') {
            steps {
                echo '========== DEPLOY KUBERNETES APPLICATION =========='

                sh """
                    kubectl apply \
                      -f k8s/namespace.yaml

                    sed \
                      "s|IMAGE_PLACEHOLDER|${ECR_REPOSITORY_URL}:${IMAGE_TAG}|g" \
                      k8s/deployment.yaml \
                      | kubectl apply -f -
                    
                    kubectl apply \
                      -f k8s/service.yaml
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                echo '========== VERIFY DEPLOYMENT =========='

                sh """
                    kubectl get nodes

                    kubectl get pods \
                      -n ${K8S_NAMESPACE}

                    kubectl get deployment \
                      -n ${K8S_NAMESPACE}

                    kubectl get service \
                      -n ${K8S_NAMESPACE}
                """
            }
        }
    }

    post {

        success {
            echo '========== PIPELINE SUCCESS =========='
            echo 'EKS application deployed successfully.'
        }

        failure {
            echo '========== PIPELINE FAILED =========='
            echo 'Check the Jenkins console output for the failed stage.'
        }

        always {
            echo '========== PIPELINE COMPLETED =========='
        }
    }
}
