pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = credentials('aws-account-id')  // from Jenkins Credentials
        ECR_REPO_NAME = 'my-app-repo'
        CLUSTER_NAME = 'my-ecs-cluster'
        SERVICE_NAME = 'my-app-service'
        TASK_DEF_FAMILY = 'my-task-def'
        CONTAINER_NAME = 'my-app-container'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/your-org/your-repo.git', branch: 'main'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
                    sh "docker build -t $ECR_REPO_NAME:$IMAGE_TAG ."
                    sh "docker tag $ECR_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:$IMAGE_TAG"
                    sh "docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:$IMAGE_TAG"
                }
            }
        }

        stage('Create ECS Cluster if not exists') {
            steps {
                script {
                    def clusterExists = sh(script: "aws ecs describe-clusters --clusters $CLUSTER_NAME --region $AWS_REGION --query 'clusters[0].status' --output text || echo 'MISSING'", returnStdout: true).trim()
                    if (clusterExists == "MISSING" || clusterExists == "INACTIVE") {
                        sh "aws ecs create-cluster --cluster-name $CLUSTER_NAME --region $AWS_REGION"
                    }
                }
            }
        }

        stage('Register Task Definition') {
            steps {
                script {
                    def taskDefJson = """
                    {
                      "family": "$TASK_DEF_FAMILY",
                      "networkMode": "awsvpc",
                      "requiresCompatibilities": ["FARGATE"],
                      "cpu": "256",
                      "memory": "512",
                      "containerDefinitions": [
                        {
                          "name": "$CONTAINER_NAME",
                          "image": "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:$IMAGE_TAG",
                          "essential": true,
                          "portMappings": [
                            {
                              "containerPort": 80,
                              "protocol": "tcp"
                            }
                          ]
                        }
                      ]
                    }
                    """
                    writeFile file: 'taskdef.json', text: taskDefJson
                    sh "aws ecs register-task-definition --cli-input-json file://taskdef.json --region $AWS_REGION"
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    def serviceExists = sh(script: "aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME --region $AWS_REGION --query 'services[0].status' --output text || echo 'MISSING'", returnStdout: true).trim()
                    if (serviceExists == "MISSING" || serviceExists == "INACTIVE") {
                        sh """
                        aws ecs create-service \
                          --cluster $CLUSTER_NAME \
                          --service-name $SERVICE_NAME \
                          --task-definition $TASK_DEF_FAMILY \
                          --launch-type FARGATE \
                          --desired-count 1 \
                          --network-configuration 'awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}' \
                          --region $AWS_REGION
                        """
                    } else {
                        sh "aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME --task-definition $TASK_DEF_FAMILY --region $AWS_REGION"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployment to ECS successful. Check the AWS Console for details."
        }
        failure {
            echo "Deployment failed. Check Jenkins logs and AWS Console for diagnostics."
        }
    }
}
