pipeline {
	agent any
	
	tools {
		nodejs 'NodeJS'
	}

	
	environment {
		SONAR_PROJECT_KEY = 'CICD-Project-1'
		SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
		DOCKER_HUB_REPO = 'pratiksinha20/cicd-project-1'
		JOB_NAME_NOW = 'cicd02'
	}
	
	stages {
		stage('GitHub Checkout'){
			steps {
				git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/pratiksinha-kol/CICD-for-NodeJS.git'
			}
		}
		
		stage('Unit Test'){
			steps {
				sh 'npm test'
				sh 'npm install'		
			}
		}
		
		stage('SonarQube Analysis'){
			steps {
				withCredentials([string(credentialsId: 'cicd-project-token', variable: 'SONAR_TOKEN')]) {
    					
					withSonarQubeEnv('SonarQube') {
    						sh """
						${SONAR_SCANNER_HOME}/bin/sonar-scanner \
						-Dsonar.projectKey=${SONAR_PROJECT_KEY} \
						-Dsonar.sources=. \
						-Dsonar.host.url=http://sonarqube-dind:9000 \
						-Dsonar.login=${SONAR_TOKEN}
						"""
					}
				}
			}
		}
		
		stage('Docker Image') {
			steps {
				script {
					docker.build("${JOB_NAME_NOW}:latest")
				}
			}
		}	
		/*
		stage('Docker Image'){
			steps {
				script {
					docker.build("${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}")
				}
			}
		}
		*/
		
		stage('Trivy Scan'){
			steps {
				sh 'trivy --severity HIGH,CRITICAL --no-progress --format table -o trivy-report.html image ${JOB_NAME_NOW}:latest'
			}
		}
		/*
		stage('Login to ECR'){
			steps {
				sh """
				aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 358966077154.dkr.ecr.us-east-1.amazonaws.com
				"""
			}
		}
		stage('Push Image to ECR'){
			steps {
				script {
				docker.image("${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}").push()
				}
			}
		}
	*/	
	}	
}
