# Jenkins Pipeline for CI/CD Deployment of Node.js Application on AWS EC2 via ECR

This repository contains a Jenkins pipeline script that automates the CI/CD process for deploying a Node.js application. The pipeline builds the application, creates a Docker image, pushes it to AWS Elastic Container Registry (ECR), and deploys it on an AWS EC2 instance.

## Features
- **Automated Deployment**: Deploys the latest code changes to an EC2 instance whenever changes are pushed to GitHub.
- **Dockerized Build Process**: Uses a multi-stage Docker build to optimize the application image.
- **AWS ECR Integration**: Pushes built images to AWS ECR for centralized container storage.
- **EC2 Deployment**: Ensures smooth deployment by stopping existing containers and replacing them with the latest image.
- **Parameterized Execution**: Supports customizable parameters like region, repository name, and EC2 instance details.

## Prerequisites
Ensure you have the following before setting up the pipeline:
1. **Jenkins** installed and configured with the required plugins:
   - Pipeline Plugin
   - Git Plugin
   - SSH Pipeline Steps Plugin
2. **AWS CLI** installed and configured on both Jenkins and EC2.
3. **AWS ECR** repository created or permissions to create one dynamically.
4. **EC2 instance** running with Docker installed.
5. **GitHub Repository** to store the application code.

## Pipeline Workflow
1. **Code Checkout**: Fetches the latest code from GitHub.
2. **ECR Repository Check**: Creates the AWS ECR repository if it doesn’t exist.
3. **Dockerfile Creation**: Generates a multi-stage `Dockerfile` dynamically.
4. **Application Build**: Installs dependencies and runs build scripts if required.
5. **Login to AWS ECR**: Authenticates Docker with AWS ECR.
6. **Docker Image Build & Push**: Builds the Docker image and pushes it to AWS ECR.
7. **EC2 Deployment**:
   - Stops any running containers on EC2.
   - Pulls the latest image from ECR.
   - Runs the new container with `--restart always` to ensure high availability.
8. **Post-Deployment Checks**: Confirms the application is running successfully.

## Environment Variables
The pipeline allows customization through parameters:

| Parameter      | Default Value                             | Description |
|---------------|-----------------------------------------|-------------|
| `AWS_REGION`  | `us-east-1`                            | AWS region where resources are deployed |
| `ECR_REPO`    | `my-nodejs-application`                | AWS ECR repository name |
| `ECR_URL`     | `Account_ID.dkr.ecr.us-east-1.amazonaws.com/my-nodejs-application` | AWS ECR URL |
| `EC2_USER`    | `ubuntu`                               | EC2 SSH username |
| `EC2_HOST`    | `Public Address of Server`                        | Public IP of EC2 instance |
| `CONTAINER_PORT` | `4000`                            | Port on which the application runs |
| `GIT_BRANCH`  | `main`                                | Branch to deploy |

## Setup Instructions
1. Clone this repository:
2. Configure Jenkins with GitHub credentials and AWS credentials.
3.Create a Jenkins Pipeline Job, and copy the Jenkinsfile content into it.
4.Add the necessary AWS credentials (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY) and SSH credentials for EC2.
5.Run the pipeline
