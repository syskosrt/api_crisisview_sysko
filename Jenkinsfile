pipeline {
    agent any

    environment {
        VM_BACK = 'sysko@192.168.146.132'
        SSH_KEY = '/var/jenkins_home/.ssh/id_ed25519'
        IMAGE_NAME = 'crisisview-api'
        CONTAINER_NAME = 'api'
        MYSQL_CONTAINER = 'mysql-crisiview'
        TEST_MYSQL_CONTAINER = 'mysql-crisisview-tests'
        NETWORK_NAME = 'crisisview'
        API_PORT = '3001'
        SONAR_TOKEN = 'sqa_781355da3ded9cb6172ec77b17488075d163dbe8'
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

stage('Start MySQL for tests') {
    steps {
        sh '''
            docker rm -f ${TEST_MYSQL_CONTAINER} || true
            docker network create ${NETWORK_NAME} || true

            docker run -d \
                --name ${TEST_MYSQL_CONTAINER} \
                --network ${NETWORK_NAME} \
                -e MYSQL_ROOT_PASSWORD=root \
                -e MYSQL_DATABASE=incident_db \
                mysql:8.4.8

            sleep 25
        '''
    }
}

        stage('Install dependencies') {
            steps {
                sh 'npm install'
            }
        }

stage('Run tests') {
    steps {
        sh '''
            DB_HOST=${TEST_MYSQL_CONTAINER} \
            DB_PORT=3306 \
            DB_USER=root \
            DB_PASSWORD=root \
            DB_NAME=incident_db \
            npm test | tee reports/tests/api-tests.log
        '''
    }
}

        stage('SonarQube analysis') {
            steps {
                sh '''
                    sonar-scanner \
                      -Dsonar.projectKey=crisisview-api \
                      -Dsonar.projectName="CrisisView API" \
                      -Dsonar.sources=. \
                      -Dsonar.exclusions=node_modules/**,reports/**,coverage/**,*.tar \
                      -Dsonar.host.url=http://sonarqube:9000 \
                      -Dsonar.token=${SONAR_TOKEN} | tee reports/quality/sonarqube-api.log
                '''
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
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                    docker save ${IMAGE_NAME}:${BUILD_NUMBER} -o ${IMAGE_NAME}.tar
                '''
            }
        }

        stage('Deploy API on VM Back') {
            steps {
                sh '''
                    scp -i ${SSH_KEY} -o StrictHostKeyChecking=no ${IMAGE_NAME}.tar ${VM_BACK}:/tmp/

                    ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${VM_BACK} "
                        docker load -i /tmp/${IMAGE_NAME}.tar &&
                        docker network create ${NETWORK_NAME} || true

                        docker inspect ${MYSQL_CONTAINER} >/dev/null 2>&1 || docker run -d \
                            --name ${MYSQL_CONTAINER} \
                            --network ${NETWORK_NAME} \
                            -e MYSQL_ROOT_PASSWORD=root \
                            -e MYSQL_DATABASE=incident_db \
                            -p 3307:3306 \
                            mysql:8.4.8

                        docker rm -f ${CONTAINER_NAME} || true

                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            --network ${NETWORK_NAME} \
                            -p ${API_PORT}:${API_PORT} \
                            -e DB_HOST=${MYSQL_CONTAINER} \
                            -e DB_PORT=3306 \
                            -e DB_USER=root \
                            -e DB_PASSWORD=root \
                            -e DB_NAME=incident_db \
                            ${IMAGE_NAME}:${BUILD_NUMBER}
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
            sh 'docker rm -f ${TEST_MYSQL_CONTAINER} || true'
            archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
        }
    }
}