pipeline {
    agent any
    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO_WEB = '836759839628.dkr.ecr.ap-south-1.amazonaws.com/prod/web'
        ECR_REPO_API = '836759839628.dkr.ecr.ap-south-1.amazonaws.com/prod/api'
        IMAGE_TAG = "${gitCommit}-${BUILD_NUMBER}"
    }
    stages {
        stage('Cleaning Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Static Code Analysis') {
            parallel {
                stage('SonarQube SAST Analysis') {
                    steps {
                        withSonarQubeEnv('sonar-scanner') {
                            sh '''
                            $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=test1 \
                            -Dsonar.java.binaries=. -Dsonar.coverage.exclusions=**/test/** \
                            -Dsonar.coverage.minimumCoverage=80 -Dsonar.issue.severity=HIGH \
                            -Dsonar.security.hotspots=true
                            '''
                        }
                    }
                }

                stage('OWASP Dependency-Check') {
                    steps {
                        dependencyCheck additionalArguments: '--scan ./ --format ALL', 
                                        odcInstallation: 'dp', 
                                        stopBuild: true
                        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                    }
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    // Run unit tests for the web app
                    dir('web') {
                        sh 'npm run test:unit'
                    }
                    // Run integration tests for the web app
                    dir('web') {
                        sh 'npm run test:integ'
                    }
                    // Run integration tests for the API service
                    dir('api') {
                        sh 'python3 test.py'
                    }
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build Web Image') {
                    steps {
                        script {
                            dir('web') {
                                // Build web app image for prod
                                sh 'docker build -t $ECR_REPO_WEB:$IMAGE_TAG .'
                            }
                        }
                    }
                }

                stage('Build API Image') {
                    steps {
                        script {
                            dir('api') {
                                // Build API app image for prod
                                sh 'docker build -t $ECR_REPO_API:$IMAGE_TAG .'
                            }
                        }
                    }
                }
            }
        }

        stage('Anchore Image Scan') {
            parallel {
                stage('Scan Web Image') {
                    steps {
                        script {
                            // Scan the web image for vulnerabilities
                            sh 'anchore scan --output json --image $ECR_REPO_WEB:$IMAGE_TAG > anchore-scan-web-prod.json'
                            archiveArtifacts allowEmptyArchive: true, artifacts: 'anchore-scan-web-prod.json'
                        }
                    }
                }

                stage('Scan API Image') {
                    steps {
                        script {
                            // Scan the API image for vulnerabilities
                            sh 'anchore scan --output json --image $ECR_REPO_API:$IMAGE_TAG > anchore-scan-api-prod.json'
                            archiveArtifacts allowEmptyArchive: true, artifacts: 'anchore-scan-api-prod.json'
                        }
                    }
                }
            }
        }

        stage('Push Images to ECR') {
            parallel {
                stage('Push Web Image to Prod') {
                    steps {
                        script {
                            // Log in to AWS ECR and push the web image for prod
                            sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin 836759839628.dkr.ecr.$AWS_REGION.amazonaws.com'
                            sh 'docker push $ECR_REPO_WEB:$IMAGE_TAG'
                        }
                    }
                }

                stage('Push API Image to Prod') {
                    steps {
                        script {
                            // Log in to AWS ECR and push the API image for prod
                            sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin 836759839628.dkr.ecr.$AWS_REGION.amazonaws.com'
                            sh 'docker push $ECR_REPO_API:$IMAGE_TAG'
                        }
                    }
                }
            }
        }
    }
}
