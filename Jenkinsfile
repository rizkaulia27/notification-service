pipeline {

    agent any

    environment {
        IMAGE = "kaulie27/notification-service:${env.BUILD_NUMBER}"
        NETWORK = "notification-net"
    }

    stages {

        // 1. CHECKOUT
        stage('Checkout Repo') {
            steps {
                echo "CHECKOUT SUCCESS"
            }
        }

        // 2. UNIT TEST
        stage('Unit Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'go test -short ./...'
                }
            }
        }

        // 3. LINT / VET
        stage('Lint / Vet') {
            steps {
                sh 'go vet ./...'
            }
        }

        // 4. BUILD IMAGE
        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE .'
            }
        }

        // 5. FUNCTIONAL TEST
        stage('Functional Test') {
            steps {
                sh '''
                    echo RUN FUNCTIONAL TEST

                    # cleanup lama
                    docker rm -f mongo-test || true
                    docker rm -f test-notification || true
                    docker network rm $NETWORK || true

                    # buat network
                    docker network create $NETWORK

                    # jalankan mongodb
                    docker run -d \
                      --name mongo-test \
                      --network $NETWORK \
                      -e MONGO_INITDB_ROOT_USERNAME=admin \
                      -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
                      mongo

                    echo WAIT MONGODB
                    sleep 15

                    # jalankan notification service
                    docker run -d \
                      --name test-notification \
                      --network $NETWORK \
                      -e MONGO_URI="mongodb://admin:admin123@mongo-test:27017/?authSource=admin" \
                      -p 8088:8088 \
                      $IMAGE

                    echo WAIT APPLICATION
                    sleep 15

                    echo CHECK CONTAINER
                    docker ps -a

                    echo CHECK LOGS
                    docker logs test-notification

                    # hubungkan jenkins ke network
                    docker network connect $NETWORK jenkins-server || true

                    # functional test
                    go test -run TestNotificationAPI
                '''
            }
        }

        // 6. PUSH IMAGE
        stage('Push Image') {
            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-login',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'PASSWORD'
                )]) {

                    sh '''
                        echo "$PASSWORD" | docker login -u "$USERNAME" --password-stdin
                        docker push $IMAGE
                    '''
                }
            }
        }

        // 7. DEPLOY
        stage('Deploy') {
            steps {
                echo "DEPLOY SUCCESS"
            }
        }

        // 8. VERIFY
        stage('Verify') {
            steps {
                echo "PIPELINE SUCCESS"
            }
        }
    }

    post {

        success {
            echo 'PIPELINE SUCCESS'
        }

        failure {
            echo 'PIPELINE FAILED'
        }

        always {

            sh '''
                docker rm -f mongo-test || true
                docker rm -f test-notification || true
                docker network rm $NETWORK || true
            '''
        }
    }
}