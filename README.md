# Three-Tier Application Deployment 

![Done](https://github.com/user-attachments/assets/543c6c65-423d-4071-bb26-992208e7b044)

This repository contains the code and configuration for deploying a three-tier application using Docker, Kubernetes, and AWS services. The project showcases a complete CI/CD pipeline for a web application using Jenkins, with automated notifications through Slack.

## Project Overview

The project architecture consists of three tiers:
1. **Web Tier**: Built with React.js, this layer handles the user interface and client-side logic, providing a seamless experience to users.
2. **Logical Tier**: Powered by Node.js, this layer contains the backend business logic and APIs, ensuring smooth and efficient processing of user requests.
3. **Database Tier**: Utilizes MongoDB for robust and scalable data storage solutions.


## Provisioning Infrastructure

![Copy of Untitled Diagram drawio](https://github.com/user-attachments/assets/6376fc76-6178-4c1b-b0ba-ab30437263d1)

The AWS infrastructure for this project includes:
- A VPC with two public subnets (Main subnet and Backup subnet).
- An EC2 instance in the main subnet.
- An EKS cluster spanning across the subnets.
- ECR repositories for storing Docker images.

The infrastructure is provisioned manually from the AWS Management Console.

## Prerequisites

Before you begin, ensure you have the following installed:
- Docker
- Kubernetes (kubectl)
- AWS CLI
- Jenkins with requred plugnis
- Slack account

  
## Key Components

- **Docker**: Containerizes each application component to ensure consistency across different environments.
- **AWS ECR (Elastic Container Registry)**: Stores Docker images securely and at scale.
- **Amazon EKS (Elastic Kubernetes Service)**: Manages and orchestrates the deployment of containers.
- **Kubernetes Ingress**: Manages external access to services in a Kubernetes cluster.
- **Load Balancer**: Distributes incoming traffic efficiently to ensure high availability.
- **AWS CloudWatch**: Provides comprehensive monitoring and logging.
- **Jenkins**: Automates the CI/CD pipeline.
- **Slack**: Sends notifications on pipeline success or failure.

## Jenkins Pipeline

The Jenkins pipeline automates the following steps:
1. **Build and Push Docker Images**: Builds Docker images for the web, logic, and database tiers and pushes them to AWS ECR.
2. **Deploy to EKS**: Deploys the application components to the EKS cluster.
3. **Setup Ingress and Load Balancer**: Configures Kubernetes Ingress and the load balancer.
4. **Slack Notifications**: Sends notifications to a Slack channel upon pipeline success or failure:
   - On success, the pipeline fetches the Load Balancer DNS and sends a Slack notification with the details.
   - On failure, it sends a Slack notification indicating the need to check Jenkins logs for more details.


## Jenkinfile

### Enviroment Variables

```groovy
pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'   		//'your-aws-region'
        AWS_ACCOUNT_ID = '180899995182'		//'your-aws-account-id'
        ECR_REPOSITORY_1 = 'public.ecr.aws/w2k2d3f8/frontend:latest'	//'your-ecr-repo-1'
        ECR_REPOSITORY_2 = 'public.ecr.aws/w2k2d3f8/backend:latest'	//'your-ecr-repo-2'
        DOCKER_IMAGE_1 = 'frontend'		//'frontend'
        DOCKER_IMAGE_2 = 'backend'		//'backend'
        KUBE_NAMESPACE = 'default'		//'default'
        INGRESS_NAMESPACE = 'ingress-nginx'	//'ingress-nginx'
        CLUSTER_NAME = '3-tier-cluster'
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')        
        SLACK_CHANNEL = '3-tier-app-channel'  // Replace with your Slack channel
        SLACK_CREDENTIAL_ID = 'slack-webhook'  // The ID of your Slack webhook credentials in Jenkins
    }
```

### Initial Configuration ECR and EKS

```groovy
    stages {
        stage('Login to AWS ECR') {
            steps {
                script {
                    sh '''
                    aws ecr-public get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin public.ecr.aws/w2k2d3f8
                    '''
                }
            }
        }

        stage('Configure kubectl for EKS') {
            steps {
                script {
                    sh '''
                    aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
                    '''
                }
            }
        }
```

### Build docker files then push docker images to ECR Repos

```groovy
        stage('Build and Push Docker Images') {
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
```

### Deployment configuration for EKS 

```groovy
        stage('Deploy Frontend, Backend, and MongoDB') {
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
```

### Deploy Nginx Ingress Controller Using Helm

```groovy
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
        stage('Initialize Nginx Ingress Controller') {
            steps {
                sleep time: 20, unit: 'SECONDS'
                echo 'Nginx Ingress Controller initialization complete.'
            }
        }
        stage('Deploy Ingress Resource') {
            steps {
                script {
                    sh '''
                    kubectl apply -f ./K8s/ingress.yaml --namespace ${KUBE_NAMESPACE}
                    '''
                }
            }
        }
```
### Fetch Load Balancer DNS and configure Slack notifier 

```groovy
        stage('Fetch Load Balancer DNS') {
            steps {
                sleep time: 10, unit: 'SECONDS'
                echo 'Load Balancer initialization complete.'
            }
        }

        stage('Print DNS of Load Balancer') {
            steps {
                script {
                    def ingressName = "main-ingress"
                    def namespace = "${KUBE_NAMESPACE}"
                    def loadBalancerDNS = sh(script: "kubectl get ing ${ingressName} --namespace ${namespace} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                    echo "Load Balancer DNS: ${loadBalancerDNS}"
                }
            }
        }
    }

   post {
        success {
            script {
                def ingressName = "main-ingress"
                def namespace = "${KUBE_NAMESPACE}"
                def loadBalancerDNS = sh(script: "kubectl get ing ${ingressName} --namespace ${namespace} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                slackSend(channel: "${SLACK_CHANNEL}", message: "Pipeline completed successfully. Load Balancer DNS: ${loadBalancerDNS}", tokenCredentialId: "${SLACK_CREDENTIAL_ID}")
            }
        }
        failure {
            script {
                slackSend(channel: "${SLACK_CHANNEL}", message: "Pipeline failed. Check the Jenkins logs for details.", tokenCredentialId: "${SLACK_CREDENTIAL_ID}")
            }
        }
    }
}
```

### Running the Pipeline

Once everything is set up, pushing code changes to the GitHub repository will automatically trigger the Jenkins pipeline, build the Docker images, push them to ECR, deploy them to the EKS cluster, and notify the team via Slack.

### Conclusion

This project demonstrates a robust and automated CI/CD pipeline using GitHub, Jenkins, Docker, AWS ECR, and AWS EKS. It streamlines the process from code push to deployment, ensuring high availability and efficient collaboration through Slack notifications. This setup enhances development workflow, deployment speed, and infrastructure scalability.


