pipeline {
    agent any

	// Set the environment variables
    environment {
        PATH = "${env.HOME}/bin:${env.PATH}"
    }

	// Multistage pipeline
    stages {
		// Stage 1 - Build and Push Docker Image to DockerHub
		// stage('Build and Push to DockerHub') {
        //     steps {
        //         script {
        //             docker.withRegistry('https://index.docker.io/v1/', 'DockerHub') {
        //                 def customImage = docker.build("${IMAGE_REPO}:${IMAGE_TAG}")
        //                 customImage.push()
        //             }
        //         }
        //     }
        // }

		// Stage 1 - Install AWS CLI
        stage('Install AWS CLI') {
			steps {
				sh '''
					if ! command -v aws >/dev/null 2>&1; then
						echo "AWS CLI not found. Installing..."
						curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
						unzip -q awscliv2.zip
						./aws/install -i $HOME/aws-cli -b $HOME/bin
					else
						echo "AWS CLI is already installed: $(aws --version)"
					fi
				'''
			}
		}

		// Stage 2 - Install kubectl
        stage('Install kubectl') {
            steps {
                sh '''
					if ! command -v kubectl >/dev/null 2>&1; then
						echo "Installing kubectl..."
						curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.2/2024-11-15/bin/linux/amd64/kubectl
						chmod +x ./kubectl
						mkdir -p $HOME/bin
						cp ./kubectl $HOME/bin/kubectl
					else
						echo "kubectl is already installed: $(kubectl version --client --short)"
					fi
                '''
            }
        }

		// Stage 3 - Build and Push Docker Image to ECR
		stage('Build & Push to ECR') {
			steps {
				script {
					withAWS(region: "${AWS_REGION}", credentials: 'AWS') {
						sh '''
							echo "Logging into Amazon ECR..."
							aws ecr get-login-password --region ${AWS_REGION} | \
							docker login --username AWS --password-stdin ${IMAGE_REGISTRY}

							echo "Building Docker image..."
							docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

							echo "Tagging image for ECR..."
							docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_REPO}:${IMAGE_TAG}

							echo "Pushing image to ECR..."
							docker push ${IMAGE_REPO}:${IMAGE_TAG}
						'''
					}
				}
			}
		}

		// Stage 4 - Deploy MySQL database on EKS Cluster
        stage('Deploy Database') {
            steps {
				script {
				// Install AWS Steps plugin to make this work
				withAWS(region: "${AWS_REGION}", credentials: 'AWS') {
						try {
							sh '''
								cd deploy
								sed "s|\\${IMAGE_NAME}|${IMAGE_REPO}|g" database-initializer.yaml | \
  								sed "s|\\${IMAGE_TAG}|${IMAGE_TAG}|g" > database-initializer-rendered.yaml
								aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION} --role-arn ${ROLE_ARN}
								kubectl apply -f database-secret.yaml
								kubectl apply -f database-initializer-rendered.yaml
								kubectl get pods
							'''
						} catch (exception) {
							echo "❌ Failed to deploy database on EKS cluster: ${exception}"
							error("Halting pipeline due to database deployment failure.")
						}
					}
                }
            }
        }
    }

    // Cleanup the workspace in the end
	post {
        always {
            cleanWs()
        }
    }
}
