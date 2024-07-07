pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'   		//'your-aws-region'
        AWS_ACCOUNT_ID = '975049995182'		//'your-aws-account-id'
        ECR_REPOSITORY_1 = 'public.ecr.aws/w2k2d3f8/frontend:latest'	//'your-ecr-repo-1'
        ECR_REPOSITORY_2 = 'public.ecr.aws/w2k2d3f8/backend:latest'	//'your-ecr-repo-2'
        DOCKER_IMAGE_1 = 'frontend'		//'frontend'
        DOCKER_IMAGE_2 = 'backend'		//'backend'
        KUBE_NAMESPACE = 'default'		//'default'
        INGRESS_NAMESPACE = 'ingress-nginx'	//'ingress-nginx'
        CLUSTER_NAME = '3-tier-cluster'
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
                    aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/w2k2d3f8
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
                    docker build -t ${DOCKER_IMAGE_1} -f ./Docker/frontend/Dockerfile Docker/frontend/
                    docker tag ${DOCKER_IMAGE_1}:latest ${ECR_REPOSITORY_1}
                    docker push ${ECR_REPOSITORY_1}
                    '''

                    // Build and push the second Docker image
                    sh '''
                    docker build -t ${DOCKER_IMAGE_2} -f ./Docker/backend/Dockerfile Docker/backend/
                    docker tag ${DOCKER_IMAGE_2}:latest ${ECR_REPOSITORY_2}
                    docker push ${ECR_REPOSITORY_2}
                    '''
                }
            }
        }

        stage('Deploy Frontend, Backend, and MongoDB') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
            }
            steps {
                script {
                    sh '''
                    kubectl apply -f ./K8s/frontend --namespace ${KUBE_NAMESPACE}
                    kubectl apply -f ./K8s/backend --namespace ${KUBE_NAMESPACE}
                    kubectl apply -f ./K8s/mongo-db --namespace ${KUBE_NAMESPACE}

                    '''
                }
            }
        }
        
        stage('Deploy Nginx Ingress Controller') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
            }
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

        stage('Sleep for 5 minutes') {
            steps {
                // Sleep for 5 minutes (300 seconds)
                sleep time: 30, unit: 'SECONDS'
                echo 'Woke up after 1 minute!'
            }
        }

        stage('Deploy Ingress Resource') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
            }
            steps {
                script {
                    sh '''
                    kubectl apply -f ./K8s/ingress.yaml --namespace ${KUBE_NAMESPACE}
                    '''
                }
            }
        }

        stage('Print DNS of Load Balancer') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
            }
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
