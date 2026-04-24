
// NOTE: Add this file to the spring-petclinic root directory and commit it to your GitHub repository. 
//Jenkins will automatically detect it and run the pipeline on each push to the main branch.

pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo 'Checking out Spring Petclinic...'
                    deleteDir()
                    git branch: 'main', url: 'https://github.com/AllMight2099/spring-petclinic.git'
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    echo "Building Spring Petclinic (Build #${BUILD_NUMBER})..."
                    sh 'mvn -B clean package -DskipTests'
                    sh 'ls -lh target/spring-petclinic-*.jar'
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    echo "Running unit tests..."
                    sh 'mvn -B test'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    echo 'Waiting for SonarQube to become ready...'
                    sh '''
                        for i in $(seq 1 30); do
                            response="$(curl -fsS "${SONAR_HOST_URL}/api/system/status" || true)"
                            if echo "$response" | grep -q '"status":"UP"'; then
                                echo "SonarQube is ready"
                                exit 0
                            fi
                            echo "SonarQube not ready yet, retrying in 10s..."
                            sleep 10
                        done

                        echo "SonarQube did not become ready in time"
                        exit 1
                    '''
                    echo 'Running SonarQube analysis...'
                    sh '''
                        mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:4.0.0.4121:sonar \
                            -DskipTests \
                            -Dsonar.projectKey=spring-petclinic \
                            -Dsonar.projectName="Spring Petclinic" \
                            -Dsonar.host.url="${SONAR_HOST_URL}" \
                            -Dsonar.token="${SONAR_TOKEN}"
                    '''
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                script {
                    echo "Archiving build artifact..."
                    archiveArtifacts artifacts: 'target/spring-petclinic-*.jar',
                                     allowEmptyArchive: false
                    sh 'cp target/spring-petclinic-*.jar /tmp/app.jar'
                }
            }
        }

        stage('Deploy to Production VM') {
            steps {
                script {
                    echo "Deploying to production via Ansible..."
                    sh '''
                        cd /var/jenkins_home/ansible
                        ansible-playbook \
                            -i inventory.ini \
                            -e "jar_source=/tmp/app.jar" \
                            deploy.yml -v
                    '''
                }
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    echo "Running security scan against petclinic-target..."
                    
                    // Use the absolute path so Jenkins finds the script outside the workspace
                    sh '/var/jenkins_home/security-scan.sh || true'
                    
                    sh '''
                        echo "===== NMAP REPORT ====="
                        cat security-reports/nmap-report.txt || echo "Nmap report not found"
                        
                        echo "===== TRIVY REPORT ====="
                        cat security-reports/trivy-report.txt || echo "Trivy report not found"
                        
                        echo "===== FFUF REPORT ====="
                        cat security-reports/ffuf-report.json || echo "FFUF report not found"
                    '''
                }
            }
        }

    }

    post {
        always {
            archiveArtifacts artifacts: 'security-reports/**',
                allowEmptyArchive: true
        }
        
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Please check the build logs and SonarQube status."
        }
    }
}
