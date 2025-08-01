pipeline {
    agent any

    environment {
        AWS_REGION      = 'us-east-1'
        AWS_ACCOUNT_ID  = credentials('aws-account-id')   // Secret Text: 12-digit AWS account ID
        ECR_REPO_NAME   = 'my-app-repo'
        CLUSTER_NAME    = 'my-ecs-cluster'
        SERVICE_NAME    = 'my-app-service'
        TASK_DEF_FAMILY = 'my-task-def'
        CONTAINER_NAME  = 'my-app-container'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'github-creds', url: 'https://github.com/your-org/your-repo.git', branch: 'main'
            }
        }

        stage('Build and Deploy to ECS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding', 
                    credentialsId: 'aws-credentials'       // AWS Access Key + Secret Key
                ]]) {
                    script {
                        sh """
                        echo "Logging in to Amazon ECR..."
                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

                        echo "Building Docker image..."
                        docker build -t $ECR_REPO_NAME:$IMAGE_TAG .

                        echo "Tagging image..."
                        docker tag $ECR_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:$IMAGE_TAG

                        echo "Pushing image to ECR..."
                        docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:$IMAGE_TAG

                        echo "Checking if ECS cluster exists..."
                        clusterExists=\$(aws ecs describe-clusters --clusters $CLUSTER_NAME --region $AWS_REGION --query 'clusters[0].status' --output text || echo 'MISSING')
                        if [ "\$clusterExists" = "MISSING" ] || [ "\$clusterExists" = "INACTIVE" ]; then
                            echo "Creating ECS cluster: $CLUSTER_NAME"
                            aws ecs create-cluster --cluster-name $CLUSTER_NAME --region $AWS_REGION
                        fi

                        echo "Generating ECS Task Definition..."
                        cat > taskdef.json << EOF
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
                        EOF

                        echo "Registering Task Definition..."
                        aws ecs register-task-definition --cli-input-json file://taskdef.json --region $AWS_REGION

                        echo "Checking if ECS Service exists..."
                        serviceExists=\$(aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME --region $AWS_REGION --query 'services[0].status' --output text || echo 'MISSING')
                        if [ "\$serviceExists" = "MISSING" ] || [ "\$serviceExists" = "INACTIVE" ]; then
                            echo "Creating ECS Service..."
                            aws ecs create-service \
                              --cluster $CLUSTER_NAME \
                              --service-name $SERVICE_NAME \
                              --task-definition $TASK_DEF_FAMILY \
                              --launch-type FARGATE \
                              --desired-count 1 \
                              --network-configuration 'awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}' \
                              --region $AWS_REGION
                        else
                            echo "Updating ECS Service..."
                            aws ecs update-service \
                              --cluster $CLUSTER_NAME \
                              --service $SERVICE_NAME \
                              --task-definition $TASK_DEF_FAMILY \
                              --region $AWS_REGION
                        fi
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment to ECS successful. Check the AWS Console for details."
        }
        failure {
            echo "❌ Deployment failed. Check Jenkins logs and AWS Console for diagnostics."
        }
    }
}
