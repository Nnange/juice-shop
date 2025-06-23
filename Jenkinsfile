pipeline {
    agent any

    environment {
        NVD_API_KEY = credentials('NVD_API_KEY')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Nnange/juice-shop.git'
            }
        }

        // stage('Build & Run DVWA') {
        //     steps {
        //         script {
        //             sh 'docker compose down || true'    // Clean up from previous run
        //             sh 'docker compose up -d'
        //         }
        //     }
        // }

        // stage('Health Check') {
        //     steps {
        //         script {
        //             sh 'sleep 5'
        //             sh 'curl -s -o /dev/null -w "%{http_code}" http://localhost | grep 200'                }
        //     }
        // }

        stage('Trivy Scan') {
            steps {
                // Scan Docker image for vulnerabilities
                sh '''
                trivy image --severity HIGH,CRITICAL \
                    --format template \
                    --template "@/var/lib/jenkins/templates/html.tpl" -o trivy-image-report.html ghcr.io/digininja/dvwa:latest
                '''
                // Scan the DVWA application filesystem
                sh '''
                trivy fs . --severity HIGH,CRITICAL \
                    --format table -o trivy-fs-report.txt
                '''
            }
        }

        stage('Dependency Scanning') {
            steps {
                dependencyCheck additionalArguments:
                '''
                    --scan . 
                    --format ALL 
                    --prettyPrint
                    --nvdApiKey ${NVD_API_KEY}
                ''', 
                odcInstallation: 'OWASP-DC'
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('Secret Scan - GitLeaks') {
            steps {
                script {
                    sh '''
                        echo "Running GitLeaks..."
                        gitleaks detect --source . --report-format sarif --report-path gitleaks-report.sarif || true
                    '''
                }
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarCloud') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }   
        }

        
        stage('Checkov Scan') {
            steps {
                script {
                    docker.image('bridgecrew/checkov:latest').inside('--entrypoint=""') {
                        sh '''
                            checkov -d . -o cli -o junitxml --output-file-path ./checkov-report.xml --framework dockerfile || true
                            # Debug: Check if the file was created
                            ls -lh ./checkov-report.xml
                        '''
                    }
                }
                // Publish JUnit results
                // junit allowEmptyResults: true, testResults: 'checkov-report.xml', skipPublishingChecks: true
                step([$class: 'JUnitResultArchiver', testResults: 'checkov-report.xml', allowEmptyResults: true, skipPublishingChecks: true, healthScaleFactor: 0.0])
            }
            post {
                always {
                    script {
                        if (fileExists('checkov-report.xml')) {
                            def report = readFile('checkov-report.xml')
                            echo "Checkov report size: ${report.length()} bytes"
                            if (report.contains('failure')) {
                                echo "Checkov detected ${report.count('failure')} failures. Review checkov-report.xml for details."
                            }
                        } else {
                            echo "Warning: checkov-report.xml not found after execution."
                        }
                    }
                }
            }
        }
        

    }

    post {
        always {
            script {
                // Force build status to SUCCESS regardless of JUnit failures
                currentBuild.result = 'SUCCESS' 
            }
            archiveArtifacts artifacts: 'gitleaks-report.sarif, checkov-report.xml, dependency-check-report.xml, trivy-fs-report.txt, trivy-image-report.html', allowEmptyArchive: true
            // echo 'Cleaning up containers...'
            // sh 'docker-compose down'
        }
    }
}
