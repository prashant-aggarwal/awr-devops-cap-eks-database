pipeline {
    agent any

	// Set the environment variables
    environment {
        PATH = "${env.HOME}/bin:${env.PATH}"
    }

	// Multistage pipeline
    stages {
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
						echo "kubectl is already installed: $(kubectl version --client)"
					fi
                '''
            }
        }

		// Stage 3 - Install Helm
		stage('Install Helm') {
			steps {
				sh '''
					if ! command -v helm >/dev/null 2>&1; then
						echo "Installing Helm..."
						HELM_VERSION="v3.14.4"
						curl -LO https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz
						tar -xzf helm-${HELM_VERSION}-linux-amd64.tar.gz
						mkdir -p $HOME/bin
						cp linux-amd64/helm $HOME/bin/helm
						chmod +x $HOME/bin/helm
						echo "Helm installed at: $HOME/bin/helm"
					else
						echo "Helm is already installed: $(helm version)"
					fi
				'''
			}
		}

		// Stage 4 - Build and Push Docker Image to ECR
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

		// Stage 5 - Deploy MySQL database on EKS Cluster
        stage('Deploy Database') {
            steps {
				script {
				// Install AWS Steps plugin to make this work
				withAWS(region: "${AWS_REGION}", credentials: 'AWS') {
						try {
							sh '''
								# Configure EKS access
								aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION} --role-arn ${ROLE_ARN}
								
								if helm status "${DATABASE_NAME}" >/dev/null 2>&1; then
									echo "Helm database '${DATABASE_NAME}' already exists. Skipping installation."
								else
									echo "Installing MariaDB using Helm..."
									helm install "${DATABASE_NAME}" oci://registry-1.docker.io/bitnamicharts/mariadb --wait --timeout 10m
									kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
								fi

								# Wait for MariaDB pod to be ready
								echo "Waiting for MariaDB pod to be ready..."
								DB_POD_LABEL="app.kubernetes.io/instance=${DATABASE_NAME}"

								# Wait up to 5 minutes for the pod to be ready
								for i in {1..60}; do
									READY=$(kubectl get pods -l "${DB_POD_LABEL}" -o jsonpath="{.items[0].status.containerStatuses[0].ready}" 2>/dev/null)
									if [[ "$READY" == "true" ]]; then
										echo "MariaDB pod is ready."
										break
									fi
									echo "Waiting for MariaDB... ($i/60)"
									sleep 10
								done

								if [[ "$READY" != "true" ]]; then
									echo "MariaDB did not become ready in time. Exiting."
									exit 1
								fi
								
								cd deploy

								# Render job file with dynamic image values
								sed "s|\\${IMAGE_NAME}|${IMAGE_REPO}|g" database-initializer.yaml | \
								sed "s|\\${IMAGE_TAG}|${IMAGE_TAG}|g" | \
								sed "s|\\${DATABASE_NAME}|${DATABASE_NAME}|g" > database-initializer-rendered.yaml

								# Re-run initializer job
								kubectl delete job db-initializer --ignore-not-found
								kubectl delete -f database-initializer-rendered.yaml --ignore-not-found
								kubectl apply -f database-initializer-rendered.yaml

								# Show pod status
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
