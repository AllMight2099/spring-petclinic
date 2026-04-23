pipeline {
    agent any

    parameters {
        booleanParam(name: 'RUN_DEPLOYMENT', defaultValue: false, description: 'Deploy the built artifact with Ansible after analysis')
    }

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
                        mvn -B sonar:sonar \
                            -DskipTests \
                            -Dsonar.projectKey=spring-petclinic \
                            -Dsonar.projectName="Spring Petclinic" \
                            -Dsonar.host.url="${SONAR_HOST_URL}" \
                            -Dsonar.login="${SONARQUBE_USERNAME}" \
                            -Dsonar.password="${SONARQUBE_PASSWORD}"
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

        stage('Deploy to Production') {
            when {
                expression { params.RUN_DEPLOYMENT }
            }
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

        stage('Smoke Test') {
            when {
                expression { params.RUN_DEPLOYMENT }
            }
            steps {
                script {
                    echo "Running smoke tests..."
                    sh '''
                        sleep 5
                        curl -f http://192.168.1.100:8888/ | head -20
                        echo "Application is responding"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Please check the build logs and SonarQube status."
        }
    }
}
