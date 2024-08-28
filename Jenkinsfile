pipeline {
    agent any

    environment {
        MAVEN_HOME = tool 'Maven'
        SONARQUBE = 'SonarQube'
        DOCKER_REGISTRY = 'mydockerregistry.com'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/myrepo/my-java-app.git'
            }
        }

        stage('Build') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn clean package"
            }
        }

        stage('Static Code Analysis - SonarQube') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "${MAVEN_HOME}/bin/mvn sonar:sonar"
                }
            }
        }

        stage('Dependency Scanning - OWASP Dependency-Check') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn org.owasp:dependency-check-maven:check"
            }
        }

        stage('Unit Tests') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn test"
            }
        }

        stage('Containerize Application') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/my-java-app:${env.BUILD_ID}")
                }
            }
        }

        stage('Dynamic Application Security Testing - OWASP ZAP') {
            steps {
                script {
                    def zap = docker.image('owasp/zap2docker-stable')
                    zap.inside {
                        sh "zap-baseline.py -t http://localhost:8080 -r zap_report.html"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
                }
            }
        }

        stage('Infrastructure as Code Scanning - Checkov') {
            steps {
                sh 'checkov -d ./iac/'
            }
        }

        stage('Security Gate') {
            steps {
                script {
                    // Custom script to fail the build if any severe issues are found
                    def issues = sh(script: 'grep "CRITICAL" zap_report.html', returnStatus: true)
                    if (issues != 0) {
                        error("Security Gate failed. Critical issues found.")
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                script {
                    kubernetesDeploy configs: 'k8s/deployment.yaml', kubeconfigId: 'my-kubeconfig'
                }
            }
        }

        stage('Post-Deployment Testing') {
            parallel {
                stage('Security Testing') {
                    steps {
                        sh 'zap-full-scan.py -t http://staging.myapp.com -r zap_full_report.html'
                    }
                }
                stage('Performance Testing') {
                    steps {
                        sh 'ab -n 1000 -c 10 http://staging.myapp.com/'
                    }
                }
            }
        }

        stage('Deploy to Production') {
            steps {
                input message: 'Deploy to Production?', ok: 'Deploy'
                script {
                    kubernetesDeploy configs: 'k8s/deployment.yaml', kubeconfigId: 'my-kubeconfig'
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
