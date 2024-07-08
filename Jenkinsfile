pipeline {
    agent any

    environment {
        AWS_REGION = 'your-aws-region'
        ECR_REPO_NAME = 'your-ecr-repo-name'
        IMAGE_TAG = 'latest'
        EC2_INSTANCE_ID = 'your-ec2-instance-id'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageName = "${ECR_REPO_NAME}:${IMAGE_TAG}"
                    sh "docker build -t ${imageName} ."
                }
            }
        }

        stage('Login to AWS ECR') {
            steps {
                script {
                    sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin $(aws sts get-caller-identity --query 'Account' --output text).dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Tag and Push Image to ECR') {
            steps {
                script {
                    def accountId = sh(script: "aws sts get-caller-identity --query 'Account' --output text", returnStdout: true).trim()
                    def ecrUri = "${accountId}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}"
                    sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${ecrUri}"
                    sh "docker push ${ecrUri}"
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    def user = 'ec2-user'  // Replace with your EC2 username
                    def keyPath = '/path/to/your/private-key.pem'  // Replace with the path to your private key

                    def accountId = sh(script: "aws sts get-caller-identity --query 'Account' --output text", returnStdout: true).trim()
                    def ecrUri = "${accountId}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}"

                    sh '''
                    ssh -i ${keyPath} ${user}@${EC2_INSTANCE_ID} << EOF
                    $(aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${accountId}.dkr.ecr.${AWS_REGION}.amazonaws.com)
                    docker pull ${ecrUri}
                    docker run -d --name my-running-app -p 80:80 ${ecrUri}
                    EOF
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
