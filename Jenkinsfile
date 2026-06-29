pipeline {
	agent any

	environment {
		IMAGE_NAME = "ram1uj/part-inventory-service"
		IMAGE_TAG = "${BUILD_NUMBER}"
		GITOPS_REPO_URL = "https://github.com/ramanujds/gitops-repo-forvia.git"
		GITOPS_BRANCH = "main"
		GITOPS_VALUES_FILE = "environments/prod/values/inventory-values.yaml"
		DOTNET_PROJECT = "PartInventoryService.DotNet/PartInventoryService.DotNet.csproj"
	}

	stages {
		stage('Source') {
			steps {
				echo 'Checking out source code from GitHub'
				git branch: 'main', url: 'https://github.com/ramanujds/idcube-src-repo'
			}
		}

		stage('Test') {
			steps {
				echo 'Running dotnet tests'
				sh '''
				export PATH=$PATH:/usr/local/bin:/opt/homebrew/bin
				dotnet test ${DOTNET_PROJECT} --configuration Release --no-build --logger "trx;LogFileName=test-results.trx" || true
				'''
			}
		}

		stage('Build Docker Image') {
			steps {
				echo 'Building Docker image'
				sh '''
				export PATH=$PATH:/usr/local/bin:/opt/homebrew/bin
				docker buildx build \
				-t ${IMAGE_NAME}:${IMAGE_TAG} \
				-t ${IMAGE_NAME}:latest \
				--platform linux/amd64,linux/arm64 \
				-f Dockerfile .
				'''
			}
		}

		stage('Trivy Scan') {
			steps {
				echo 'Scanning Docker image for vulnerabilities'
				sh '''
				export PATH=$PATH:/usr/local/bin:/opt/homebrew/bin
				curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
				trivy image \
				--exit-code 1 \
				--severity HIGH,CRITICAL \
				--no-progress \
				--format table \
				${IMAGE_NAME}:${IMAGE_TAG}
				'''
			}
		}

		stage('Push Image') {
			steps {
				withCredentials([usernamePassword(credentialsId: 'e4af9f44-e2b9-4253-b040-14b40090e1a6', passwordVariable: 'docker_password', usernameVariable: 'docker_user')]) {
					sh '''
					export PATH=$PATH:/usr/local/bin:/opt/homebrew/bin
					echo "$docker_password" | docker login -u "$docker_user" --password-stdin
					docker push ${IMAGE_NAME}:${IMAGE_TAG}
					docker push ${IMAGE_NAME}:latest
					docker logout
					'''
				}
			}
		}

		stage('Update GitOps Image Tag') {
			steps {
				withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'password', usernameVariable: 'username')]) {
					sh '''
					set -e
					rm -rf gitops-repo
					git clone -b ${GITOPS_BRANCH} ${GITOPS_REPO_URL} gitops-repo
					cd gitops-repo

					sed -i.bak "s|^    tag:.*|    tag: \"${IMAGE_TAG}\"|" ${GITOPS_VALUES_FILE} && rm -f ${GITOPS_VALUES_FILE}.bak

					git config user.name "jenkins-bot"
					git config user.email "jenkins-bot@users.noreply.github.com"
					git add "${GITOPS_VALUES_FILE}"

					if git diff --cached --quiet; then
						echo "No GitOps changes detected; skipping commit."
					else
						git commit -m "ci: update inventory image tag to ${IMAGE_TAG}"
						git push https://${username}:${password}@github.com/ramanujds/gitops-repo-forvia.git ${GITOPS_BRANCH}
					fi
					'''
				}
			}
		}
	}
}
