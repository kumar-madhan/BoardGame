pipeline {
    agent any

    environment {
        SONARQUBE = 'SonarQube'
        DOCKER_REGISTRY = '192.168.10.17:9092/artifactory/docker/'
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
                    def ScannerHome = tool 'SonarQube Scanner'
                    withSonarQubeEnv('SonarQube') {
                        sh '${ScannerHome}/bin/sonar-scanner'
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

        stage('Containerize Application') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/my-java-app:${env.BUILD_ID}")
                    docker.push("${DOCKER_REGISTRY}/my-java-app:${env.BUILD_ID}")
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
        }
    }
}
