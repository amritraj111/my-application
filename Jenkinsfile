pipeline {

    agent any

    environment {

        // AWS Region
        AWS_REGION = 'ap-south-1'

        // ECR repository name
        ECR_REPOSITORY = 'my-application'

        // Jenkins automatically creates this number
        IMAGE_TAG = "${BUILD_NUMBER}"

        // Application EC2 PRIVATE IP
        APPLICATION_SERVER = '172.31.7.175'

        // Docker container name on Application Server
        CONTAINER_NAME = 'my-application'
    }

    stages {

        // ============================================================
        // 1. CHECKOUT
        // ============================================================

        stage('Checkout') {

            steps {

                echo 'Checking out source code from GitHub...'

                checkout scm
            }
        }


        // ============================================================
        // 2. TEST
        // ============================================================

        stage('Test') {

            steps {

                echo 'Running basic application tests...'

                sh '''
                    set -e

                    echo "Checking Dockerfile..."

                    test -f Dockerfile

                    echo "Dockerfile found."

                    echo "Checking application file..."

                    test -f app/index.html

                    echo "Application file found."

                    echo "Basic tests completed successfully."
                '''
            }
        }


        // ============================================================
        // 3. BUILD DOCKER IMAGE
        // ============================================================

        stage('Docker Build') {

            steps {

                echo "Building Docker image..."

                sh '''
                    set -e

                    echo "Building image:"
                    echo "${ECR_REPOSITORY}:${IMAGE_TAG}"

                    docker build \
                        -t ${ECR_REPOSITORY}:${IMAGE_TAG} \
                        .

                    echo "Docker image build completed."

                    docker images | grep ${ECR_REPOSITORY} || true
                '''
            }
        }


        // ============================================================
        // 4. LOGIN TO AWS ECR
        // ============================================================

        stage('ECR Login') {

            steps {

                echo 'Logging in to AWS ECR...'

                sh '''
                    set -e

                    AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                        --query Account \
                        --output text)

                    echo "AWS Account ID:"
                    echo "${AWS_ACCOUNT_ID}"

                    ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                    echo "ECR Registry:"
                    echo "${ECR_REGISTRY}"

                    aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}

                    echo "ECR login successful."
                '''
            }
        }


        // ============================================================
        // 5. TAG + PUSH IMAGE TO ECR
        // ============================================================

        stage('Push to ECR') {

            steps {

                echo 'Tagging and pushing Docker image to ECR...'

                sh '''
                    set -e

                    AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                        --query Account \
                        --output text)

                    ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                    ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

                    echo "ECR Image:"
                    echo "${ECR_IMAGE}"

                    echo "Tagging Docker image..."

                    docker tag \
                        ${ECR_REPOSITORY}:${IMAGE_TAG} \
                        ${ECR_IMAGE}

                    echo "Pushing image to ECR..."

                    docker push ${ECR_IMAGE}

                    echo "Docker image successfully pushed to ECR."
                '''
            }
        }


        // ============================================================
        // 6. DEPLOY TO APPLICATION EC2
        // ============================================================

        stage('Deploy to Application Server') {

            steps {

                echo 'Deploying application to Application EC2...'

                sh '''
                    set -e

                    AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                        --query Account \
                        --output text)

                    ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                    ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

                    echo "Deployment details:"
                    echo "Application Server: ${APPLICATION_SERVER}"
                    echo "ECR Image: ${ECR_IMAGE}"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o ConnectTimeout=10 \
                        ubuntu@${APPLICATION_SERVER} \
                        "AWS_REGION='${AWS_REGION}' ECR_IMAGE='${ECR_IMAGE}' CONTAINER_NAME='${CONTAINER_NAME}' bash -s" << 'REMOTE_SCRIPT'

                        set -e

                        echo "========================================"
                        echo "Application Server Deployment"
                        echo "========================================"

                        echo "Server:"
                        hostname

                        echo "Current Docker containers:"
                        docker ps

                        echo "========================================"
                        echo "Getting AWS Account Information"
                        echo "========================================"

                        AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                            --query Account \
                            --output text)

                        echo "AWS Account ID:"
                        echo "${AWS_ACCOUNT_ID}"

                        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                        echo "ECR Registry:"
                        echo "${ECR_REGISTRY}"

                        echo "========================================"
                        echo "Logging into ECR"
                        echo "========================================"

                        aws ecr get-login-password \
                            --region ${AWS_REGION} | \
                        docker login \
                            --username AWS \
                            --password-stdin ${ECR_REGISTRY}

                        echo "ECR login successful."

                        echo "========================================"
                        echo "Pulling New Docker Image"
                        echo "========================================"

                        echo "Image:"
                        echo "${ECR_IMAGE}"

                        docker pull ${ECR_IMAGE}

                        echo "Image pulled successfully."

                        echo "========================================"
                        echo "Stopping Old Container"
                        echo "========================================"

                        docker stop ${CONTAINER_NAME} || true

                        echo "Old container stopped."

                        echo "========================================"
                        echo "Removing Old Container"
                        echo "========================================"

                        docker rm ${CONTAINER_NAME} || true

                        echo "Old container removed."

                        echo "========================================"
                        echo "Starting New Container"
                        echo "========================================"

                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            --restart unless-stopped \
                            -p 80:80 \
                            ${ECR_IMAGE}

                        echo "New container started."

                        echo "========================================"
                        echo "Checking Container"
                        echo "========================================"

                        sleep 5

                        docker ps \
                            --filter "name=${CONTAINER_NAME}"

                        echo "========================================"
                        echo "Testing Application"
                        echo "========================================"

                        curl -f http://localhost/ || {
                            echo "Application health check failed."
                            exit 1
                        }

                        echo "========================================"
                        echo "Deployment Successful"
                        echo "========================================"

                        echo "Application is running successfully."

REMOTE_SCRIPT
                '''
            }
        }
    }


    // ================================================================
    // POST ACTIONS
    // ================================================================

    post {

        success {

            echo '''
            ========================================
            DEPLOYMENT SUCCESSFUL
            ========================================
            Application deployed successfully.
            ========================================
            '''
        }


        failure {

            echo '''
            ========================================
            DEPLOYMENT FAILED
            ========================================
            Please check the Jenkins Console Output.
            ========================================
            '''
        }


        always {

            echo 'Cleaning unused Docker images on Jenkins...'

            sh '''
                docker image prune -f || true
            '''
        }
    }
}
