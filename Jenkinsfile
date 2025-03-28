pipeline {
    agent any

    triggers {
        githubPush() // Enables automatic triggering on push to GitHub
    }

    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS region')
        string(name: 'ECR_REPO', defaultValue: 'my-nodejs-application', description: 'ECR repository name')
        string(name: 'ECR_URL', defaultValue: 'Account_ID.dkr.ecr.us-east-1.amazonaws.com/my-nodejs-application', description: 'ECR registry URL')
        string(name: 'EC2_USER', defaultValue: 'ubuntu', description: 'EC2 SSH username')
        string(name: 'EC2_HOST', defaultValue: 'Public Address of server', description: 'EC2 instance IP')
        string(name: 'CONTAINER_PORT', defaultValue: '4000', description: 'Port exposed by the container')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Enter the branch name to deploy')
    }

    environment {
        GITHUB_REPO = "https://github.com/tobyoluyole/simple-node-application.git"
        GITHUB_CREDENTIALS_ID = "github-credentials"
        PATH = "/usr/local/bin:$PATH"
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.GIT_BRANCH}"]],
                        userRemoteConfigs: [[
                            url: env.GITHUB_REPO,
                            credentialsId: env.GITHUB_CREDENTIALS_ID
                        ]]
                    ])
                }
            }
        }

        stage('Create ECR Repository if not exists') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    script {
                        sh """
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                            
                            if ! aws ecr describe-repositories --repository-names ${params.ECR_REPO} --region ${params.AWS_REGION} > /dev/null 2>&1; then
                                echo "ECR repository does not exist. Creating it..."
                                aws ecr create-repository --repository-name ${params.ECR_REPO} --region ${params.AWS_REGION}
                            else
                                echo "ECR repository already exists."
                            fi
                        """
                    }
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    writeFile file: 'Dockerfile', text: """
                    FROM node:18 AS builder
                    WORKDIR /app
                    COPY package*.json ./
                    RUN npm install
                    COPY . .
                    
                    RUN if [ -f package.json ] && grep -q '"build":' package.json; then npm run build; else echo "No build step found"; fi
                    
                    FROM node:18-alpine
                    WORKDIR /app
                    COPY --from=builder /app .
                    CMD ["node", "server.js"]
                    EXPOSE ${params.CONTAINER_PORT}
                    """.stripIndent()
                }
            }
        }

        stage('Build Application') {
            steps {
                script {
                    sh 'npm install'
                    
                    def hasBuildScript = sh(script: "node -e \"console.log(require('./package.json').scripts?.build !== undefined)\"", returnStdout: true).trim()
                    if (hasBuildScript == "true") {
                        sh 'npm run build'
                    } else {
                        echo "No build script found in package.json. Skipping build step."
                    }
                }
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    script {
                        sh """
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                            aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${params.ECR_URL}
                        """
                    }
                }
            }
        }

        stage('Build and Tag Docker Image') {
            steps {
                script {
                    def imageTag = "${params.ECR_URL}:${env.BUILD_NUMBER}"
                    env.IMAGE_TAG = imageTag
                    sh "docker build -t ${imageTag} ."
                    sh "docker tag ${imageTag} ${params.ECR_URL}:latest"
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    sh "docker push ${env.IMAGE_TAG}"
                    sh "docker push ${params.ECR_URL}:latest"
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    // Create deployment script with enhanced container management
                    writeFile file: 'deploy.sh', text: """
                        #!/bin/bash
                        set -e
                        
                        echo "🔍 Checking for running containers..."
                        
                        # Function to check if a port is in use
                        check_port() {
                            netstat -tln | grep ":${params.CONTAINER_PORT} "
                            return \$?
                        }
                        
                        # Stop any containers using our port
                        if check_port; then
                            echo "📍 Port ${params.CONTAINER_PORT} is in use. Finding and stopping container..."
                            CONTAINER_ID=\$(docker ps -q --filter publish=${params.CONTAINER_PORT})
                            if [ ! -z "\$CONTAINER_ID" ]; then
                                echo "🛑 Stopping container \$CONTAINER_ID"
                                docker stop \$CONTAINER_ID
                                docker rm \$CONTAINER_ID
                            fi
                        fi
                        
                        # Stop any containers with our application name regardless of port
                        if docker ps -a --filter name=${params.ECR_REPO} -q | grep -q .; then
                            echo "🛑 Stopping existing application container..."
                            docker stop ${params.ECR_REPO} || true
                            docker rm ${params.ECR_REPO} || true
                        fi
                        
                        # Configure AWS CLI
                        echo "🔧 Configuring AWS CLI..."
                        export AWS_DEFAULT_REGION=${params.AWS_REGION}
                        
                        # Login to ECR
                        echo "🔑 Logging into ECR..."
                        aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${params.ECR_URL}
                        
                        # Pull latest image
                        echo "⬇️ Pulling latest image..."
                        docker pull ${params.ECR_URL}:latest
                        
                        # Run new container
                        echo "🚀 Starting new container..."
                        docker run -d \\
                            --name ${params.ECR_REPO} \\
                            -p ${params.CONTAINER_PORT}:${params.CONTAINER_PORT} \\
                            --restart always \\
                            ${params.ECR_URL}:latest
                        
                        echo "✅ Deployment completed successfully!"
                        
                        # Verify container is running
                        echo "🔍 Verifying deployment..."
                        if docker ps --filter name=${params.ECR_REPO} --format '{{.Names}}' | grep -q '^${params.ECR_REPO}\$'; then
                            echo "✅ Container is running successfully!"
                        else
                            echo "❌ Container failed to start!"
                            exit 1
                        fi
                    """.stripIndent()

                    // Deploy using SSH credentials
                    withCredentials([
                        sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY'),
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        // Copy deployment script to EC2
                        sh """
                            chmod +x deploy.sh
                            scp -i ${SSH_KEY} -o StrictHostKeyChecking=no deploy.sh ${params.EC2_USER}@${params.EC2_HOST}:/home/${params.EC2_USER}/
                        """

                        // Execute deployment script on EC2
                        sh """
                            ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} '
                                aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                                aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                                aws configure set default.region ${params.AWS_REGION}
                                bash /home/${params.EC2_USER}/deploy.sh
                            '
                        """
                    }

                    // Clean up
                    sh "rm -f deploy.sh"
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}
