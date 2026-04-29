pipeline {
    agent any

    environment {
        VM_BACK = 'sysko@192.168.146.132'
        SSH_KEY = '/var/jenkins_home/.ssh/id_ed25519'
        IMAGE_NAME = 'crisisview-api'
        CONTAINER_NAME = 'api'
        MYSQL_CONTAINER = 'mysql-crisiview'
        NETWORK_NAME = 'crisisview'
        API_PORT = '3001'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare reports') {
            steps {
                sh 'mkdir -p reports/tests reports/security reports/quality reports/deploy'
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run tests') {
            steps {
                sh 'npm test | tee reports/tests/api-tests.log'
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'sonar-scanner | tee reports/quality/sonarqube-api.log'
                }
            }
        }

        stage('Security audit') {
            steps {
                sh 'npm audit --json > reports/security/npm-audit-api.json || true'
            }
        }

        stage('Docker build') {
            steps {
                sh '''
                    docker build -t \:\ .
                    docker save \:\ -o \.tar
                '''
            }
        }

        stage('Deploy API on VM Back') {
            steps {
                sh '''
                    scp -i \ -o StrictHostKeyChecking=no \.tar \:/tmp/

                    ssh -i \ -o StrictHostKeyChecking=no \ "
                        docker load -i /tmp/\.tar &&
                        docker network create \ || true &&

                        docker inspect \ >/dev/null 2>&1 || docker run -d \\
                            --name \ \\
                            --network \ \\
                            -e MYSQL_ROOT_PASSWORD=root \\
                            -e MYSQL_DATABASE=incident_db \\
                            -p 3307:3306 \\
                            mysql:8.4.8 &&

                        docker rm -f \ || true &&

                        docker run -d \\
                            --name \ \\
                            --network \ \\
                            -p \:\ \\
                            -e DB_HOST=\ \\
                            -e DB_PORT=3306 \\
                            -e DB_USER=root \\
                            -e DB_PASSWORD=root \\
                            -e DB_NAME=incident_db \\
                            \:\
                    "
                '''
            }
        }

        stage('Smoke test API') {
            steps {
                sh '''
                    sleep 5
                    curl -f http://192.168.146.132:3001/techniciens | tee reports/deploy/api-smoke-test.log
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
        }

        success {
            echo 'API pipeline success.'
        }

        failure {
            echo 'API pipeline failed.'
        }
    }
}
