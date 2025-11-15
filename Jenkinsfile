pipeline {
    agent any

    environment {
        AWS_REGION   = "ap-south-1"
        ECR_URL      = "406881348652.dkr.ecr.ap-south-1.amazonaws.com"
        ECR_REPO     = "solar-system"
        IMAGE_TAG    = "build-${BUILD_NUMBER}"
        SONAR_PROJECT = "solar-system"
    }

    tools {
        nodejs "NodeJS_18"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-creds',
                    url: 'https://github.com/sidd-harth/solar-system-gitea.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('Unit Tests') {
            steps {
                sh "npm test || true"
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT} \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('AWS ECR Login') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh """
                    aws ecr get-login-password --region $AWS_REGION \
                    | docker login --username AWS --password-stdin $ECR_URL
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${ECR_REPO}:${IMAGE_TAG} .
                docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URL}/${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh "docker push ${ECR_URL}/${ECR_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Approval For Production') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def approval = input(
                            message: "Deploy to Production?",
                            parameters: [choice(name: 'Deploy', choices: 'Yes\nNo', description: 'Choose Yes to continue')]
                        )
                        if (approval != 'Yes') {
                            error "Deployment aborted by user."
                        }
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    try {
                        sh """
                        kubectl set image deployment/solar-deployment \
                          solar-container=${ECR_URL}/${ECR_REPO}:${IMAGE_TAG} --record

                        kubectl rollout status deployment/solar-deployment --timeout=60s
                        """
                    } catch (err) {
                        echo "Deployment Failed! Rolling Back..."
                        sh "kubectl rollout undo deployment/solar-deployment"
                        error("Deployment Failed - Rolled Back")
                    }
                }
            }
        }
    }

    post {
        success {
            emailext(
                to: "team@company.com",
                subject: "Build Success: ${JOB_NAME} - #${BUILD_NUMBER}",
                body: "Deployment successful.\nImage: ${IMAGE_TAG}"
            )
        }
        failure {
            emailext(
                to: "team@company.com",
                subject: "Build Failed: ${JOB_NAME} - #${BUILD_NUMBER}",
                body: "Deployment Failed & Rollback Completed."
            )
        }
    }
}
