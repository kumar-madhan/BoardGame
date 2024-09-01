pipeline {
    agent {label 'Build-Node'}

    environment {
        SONARQUBE = 'SonarQube'
        DOCKER_REGISTRY = '192.168.10.16:9092/artifactory/docker'
        DOCKER_REPO = "kumarmadhan/my-java-app"
    }
    
    tools {
        maven 'Maven3'
        dockerTool  'Docker'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/kumar-madhan/BoardGame.git'
            }
        }

        stage('Sonarqube-Analysis') {
            steps {
                script{
                    // def ScannerHome = tool 'SonarQube Scanner'
                    withSonarQubeEnv('SonarQube') {
                        sh 'sonar-scanner'
                    }
                }
            }
        }

        stage('OWASP Dependency-Check Scan') {
            steps {
                dir('myapp') {
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'Dependency-Check'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }

        stage('Unit Tests') {
            steps {
                sh "mvn test"
            }
        }

        stage('Build') {
            steps {
                sh "mvn clean package"
            }
        }

        stage('Deploy to Artifactory') {
            steps {
                script {
                    def server = Artifactory.server('artifactory')

                    def buildInfo = server.upload spec: """{
                        "files": [
                            {
                                "pattern": "target/*.jar",
                                "target": "maven/"
                            }
                        ]
                    }"""
                    server.publishBuildInfo(buildInfo)
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image
                    dockerImage = docker.build("${DOCKER_REPO}:${BUILD_NUMBER}")
                }
            }
        }
        
        stage('Tag Docker Image') {
            steps {
                script {
                    // Tag the image with "latest"
                    sh "docker tag ${DOCKER_REPO}:${BUILD_NUMBER} ${DOCKER_REPO}:latest"
                }
            }
        }
        
        stage('Push Docker Images') {
            steps {
                script {
                    docker.withRegistry('', "${DOCKER_CREDENTIALS_ID}") {
                        // Push the build number tagged image
                        dockerImage.push("${BUILD_NUMBER}")
                        // Push the "latest" tagged image
                        dockerImage.push("latest")
                    }
                }
            }
        }

        // stage('Dynamic Application Security Testing - OWASP ZAP') {
        //     steps {
        //         script {
        //             def zap = docker.image('owasp/zap2docker-stable')
        //             zap.inside {
        //                 sh "zap-baseline.py -t http://localhost:8080 -r zap_report.html"
        //             }
        //         }
        //     }
        //     post {
        //         always {
        //             archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
        //         }
        //     }
        // }

        // stage('Infrastructure as Code Scanning - Checkov') {
        //     steps {
        //         sh 'checkov -d ./iac/'
        //     }
        // }

        // stage('Security Gate') {
        //     steps {
        //         script {
        //             // Custom script to fail the build if any severe issues are found
        //             def issues = sh(script: 'grep "CRITICAL" zap_report.html', returnStatus: true)
        //             if (issues != 0) {
        //                 error("Security Gate failed. Critical issues found.")
        //             }
        //         }
        //     }
        // }

        // stage('Deploy to Staging') {
        //     steps {
        //         script {
        //             kubernetesDeploy configs: 'k8s/deployment.yaml', kubeconfigId: 'my-kubeconfig'
        //         }
        //     }
        // }

        // stage('Post-Deployment Testing') {
        //     parallel {
        //         stage('Security Testing') {
        //             steps {
        //                 sh 'zap-full-scan.py -t http://staging.myapp.com -r zap_full_report.html'
        //             }
        //         }
        //         stage('Performance Testing') {
        //             steps {
        //                 sh 'ab -n 1000 -c 10 http://staging.myapp.com/'
        //             }
        //         }
        //     }
        // }

        // stage('Deploy to Production') {
        //     steps {
        //         input message: 'Deploy to Production?', ok: 'Deploy'
        //         script {
        //             kubernetesDeploy configs: 'k8s/deployment.yaml', kubeconfigId: 'my-kubeconfig'
        //         }
        //     }
        // }
    }
    post {
        always {
            cleanWs()
            // Clean up the Docker environment after the build
            script {
                sh "docker rmi ${DOCKER_REPO}:${BUILD_NUMBER} || true"
                sh "docker rmi ${DOCKER_REPO}:latest || true"
            }
        }
    }
}
