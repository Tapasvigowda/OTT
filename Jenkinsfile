pipeline {

    agent any

    tools {
        maven 'Maven3'
        jdk 'JDK17'
    }

    environment {
        IMAGE_NAME = "ott-platform"
        DOCKERHUB_REPO = "tapasvigowda/ott-platform"
        IMAGE_TAG = "${BUILD_NUMBER}"
        MYSQL_DATABASE = "ott_db"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Tapasvigowda/OTT.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    withCredentials([
                        string(credentialsId: 'sonar', variable: 'SONAR_TOKEN')
                    ]) {
                        sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=ott-platform \
                        -Dsonar.projectName="OTT Platform" \
                        -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_REPO}:${IMAGE_TAG}
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_REPO}:latest
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    docker push $DOCKERHUB_REPO:$IMAGE_TAG
                    docker push $DOCKERHUB_REPO:latest

                    docker logout
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'mysql-creds',
                        usernameVariable: 'MYSQL_USER',
                        passwordVariable: 'MYSQL_PASSWORD'
                    ),
                    string(
                        credentialsId: 'mysql-root-pass',
                        variable: 'MYSQL_ROOT_PASSWORD'
                    )
                ]) {
                    sh '''
                    echo "Stopping old containers..."
                    docker-compose down || true

                    echo "Setting environment variables..."
                    export MYSQL_DATABASE=ott_db
                    export MYSQL_USER=$MYSQL_USER
                    export MYSQL_PASSWORD=$MYSQL_PASSWORD
                    export MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD

                    echo "Starting containers..."
                    docker-compose up -d --build
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                echo "Waiting for application to start..."
                sleep 40

                curl --fail http://localhost:8082/actuator/health
                '''
            }
        }

        stage('Cleanup') {
            steps {
                sh '''
                docker image prune -f
                docker system df
                '''
            }
        }
    }

    post {

        success {
            echo "=========== BUILD SUCCESS ==========="
            sh 'docker ps'
        }

        failure {
            echo "=========== BUILD FAILED ==========="
            sh 'docker ps -a'
        }

        always {
            cleanWs()
        }
    }
}
