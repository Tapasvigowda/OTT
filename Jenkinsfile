pipeline {
    agent any

    environment {
        IMAGE_NAME = "tapasvigowda/ott-platform:latest"
        CONTAINER_NAME = "ott-platform"
        NETWORK = "ott_default"
        DB_HOST = "mysql"
        DB_NAME = "ott_db"
        DB_USER = "ottuser"
    }

    stages {

        stage('Cleanup Old Container') {
            steps {
                sh '''
                docker rm -f $CONTAINER_NAME || true
                '''
            }
        }

        stage('Pull Image') {
            steps {
                sh '''
                docker pull $IMAGE_NAME
                '''
            }
        }

        stage('Start MySQL (if not running)') {
            steps {
                script {
                    sh '''
                    if [ "$(docker ps -q -f name=mysql)" ]; then
                        echo "MySQL already running"
                    else
                        echo "Starting MySQL..."
                        docker rm -f mysql || true

                        docker run -d \
                          --name mysql \
                          --network $NETWORK \
                          -e MYSQL_ROOT_PASSWORD=root \
                          -e MYSQL_DATABASE=$DB_NAME \
                          -e MYSQL_USER=$DB_USER \
                          -e MYSQL_PASSWORD=$MYSQL_PASSWORD \
                          -p 3306:3306 \
                          mysql:8.0
                    fi
                    '''
                }
            }
        }

        stage('Wait for MySQL') {
            steps {
                sh '''
                echo "Waiting for MySQL to be ready..."
                until docker exec mysql mysqladmin ping -h "localhost" --silent; do
                  sleep 5
                done
                echo "MySQL is ready"
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                withCredentials([string(credentialsId: 'mysql-password', variable: 'MYSQL_PASSWORD')]) {
                    sh '''
                    docker run -d \
                      --name $CONTAINER_NAME \
                      --network $NETWORK \
                      -p 8082:8082 \
                      -e SPRING_DATASOURCE_URL=jdbc:mysql://$DB_HOST:3306/$DB_NAME \
                      -e SPRING_DATASOURCE_USERNAME=$DB_USER \
                      -e SPRING_DATASOURCE_PASSWORD=$MYSQL_PASSWORD \
                      $IMAGE_NAME
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                echo "Checking application health..."

                for i in {1..10}; do
                  if curl -s http://localhost:8082/actuator/health | grep UP; then
                    echo "Application is UP"
                    exit 0
                  fi
                  echo "Waiting for app..."
                  sleep 10
                done

                echo "Application failed to start"
                docker logs $CONTAINER_NAME
                exit 1
                '''
            }
        }
    }

    post {
        success {
            echo "=========================="
            echo "BUILD SUCCESS"
            echo "=========================="
        }
        failure {
            echo "=========================="
            echo "BUILD FAILED"
            echo "=========================="
            sh 'docker logs ott-platform || true'
        }
    }
}
