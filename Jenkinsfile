pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'   		//'your-aws-region'
        AWS_ACCOUNT_ID = '975049995182'		//'your-aws-account-id'
        ECR_REPOSITORY_1 = 'your-ecr-repo-1'	//'your-ecr-repo-1'
        ECR_REPOSITORY_2 = 'your-ecr-repo-2'	//'your-ecr-repo-2'
        DOCKER_IMAGE_1 = 'frontend'		//'frontend'
        DOCKER_IMAGE_2 = 'backend'		//'backend'
        KUBE_NAMESPACE = 'default'		//'default'
        INGRESS_NAMESPACE = 'ingress-nginx'	//'ingress-nginx'
    }

    stages {
        stage('Login to AWS ECR') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
            }
            steps {
                script {
                    sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Configure kubectl for EKS') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
            }
            steps {
                script {
                    sh '''
                    aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
                    '''
                }
            }
        }

        stage('Build, Tag, and Push Docker Images') {
            steps {
                script {
                    // Build and push the first Docker image
                    sh '''
                    docker build -t ${DOCKER_IMAGE_1} -f path/to/Dockerfile1 .
                    docker tag ${DOCKER_IMAGE_1}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY_1}:latest
                    docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY_1}:latest
                    '''

                    // Build and push the second Docker image
                    sh '''
                    docker build -t ${DOCKER_IMAGE_2} -f path/to/Dockerfile2 .
                    docker tag ${DOCKER_IMAGE_2}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY_2}:latest
                    docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY_2}:latest
                    '''
                }
            }
        }

        stage('Deploy Frontend, Backend, and MongoDB') {
            steps {
                script {
                    sh '''
                    kubectl apply -f path/to/frontend --namespace ${KUBE_NAMESPACE}
                    kubectl apply -f path/to/backend --namespace ${KUBE_NAMESPACE}
                    kubectl apply -f path/to/mongo --namespace ${KUBE_NAMESPACE}

                    '''
                }
            }
        }

        stage('Deploy Nginx Ingress Controller') {
            steps {
                script {
                    sh '''
                    kubectl create namespace ${INGRESS_NAMESPACE} || true
                    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
                    helm repo update
                    helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ${INGRESS_NAMESPACE}
                    '''
                }
            }
        }

        stage('Deploy Ingress Resource') {
            steps {
                script {
                    sh '''
                    kubectl apply -f path/to/ingress.yaml --namespace ${KUBE_NAMESPACE}
                    '''
                }
            }
        }

        stage('Print DNS of Load Balancer') {
            steps {
                script {
                    def ingressName = "your-ingress-name"
                    def namespace = "${KUBE_NAMESPACE}"
                    def loadBalancerDNS = sh(script: "kubectl get ing ${ingressName} --namespace ${namespace} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                    echo "Load Balancer DNS: ${loadBalancerDNS}"
                }
            }
        }
    }
}
