pipeline {
    agent any

    environment {
        DOCKER_IMAGE = '2022bcs0135ruthwik/wine_mlops_lab4_2022bcs0135:latest'
        CONTAINER_NAME = 'wine-mlops-api'
        PORT = '8000'
    }

    stages {

        stage('Pull Docker Image') {
            steps {
                echo "Pulling Docker image ${DOCKER_IMAGE}"
                sh "docker pull ${DOCKER_IMAGE}"
            }
        }

        stage('Run Container') {
            steps {
                echo "Starting container ${CONTAINER_NAME}"

                sh '''
                    docker rm -f ${CONTAINER_NAME} || true

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${PORT}:8000 \
                        ${DOCKER_IMAGE}

                    echo "Container started:"
                    docker ps

                    echo "Container logs:"
                    sleep 5
                    docker logs ${CONTAINER_NAME}
                '''
            }
        }

        stage('Wait for API') {
            steps {
                echo "Waiting for API to start..."

                sh '''
                    for i in {1..30}; do
                        if curl -s http://localhost:8000 > /dev/null; then
                            echo "API is up!"
                            exit 0
                        fi
                        echo "Waiting..."
                        sleep 2
                    done

                    echo "API failed to start"
                    docker logs ${CONTAINER_NAME}
                    exit 1
                '''
            }
        }

        stage('Send Valid Request') {
            steps {
                echo "Sending valid inference request"

                sh '''
                    RESPONSE=$(curl -s -X POST http://localhost:8000/predict \
                        -H "Content-Type: application/json" \
                        -d @tests/valid_input.json)

                    echo "Response:"
                    echo $RESPONSE

                    if ! echo "$RESPONSE" | grep -q '"wine_quality"'; then
                        echo "Validation failed: wine_quality not found"
                        exit 1
                    fi

                    echo "Valid request test passed"
                '''
            }
        }

        stage('Send Invalid Request') {
            steps {
                echo "Sending invalid request"

                sh '''
                    RESPONSE=$(curl -s -w "\\n%{http_code}" -X POST http://localhost:8000/predict \
                        -H "Content-Type: application/json" \
                        -d @tests/invalid_input.json)

                    BODY=$(echo "$RESPONSE" | sed '$d')
                    STATUS=$(echo "$RESPONSE" | tail -n1)

                    echo "Response body:"
                    echo "$BODY"

                    echo "HTTP status:"
                    echo "$STATUS"

                    if [ "$STATUS" -eq 200 ]; then
                        echo "Validation failed: invalid input returned 200"
                        exit 1
                    fi

                    echo "Invalid request test passed"
                '''
            }
        }
    }

    post {

        always {
            echo "Cleaning up container..."

            sh '''
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
            '''
        }

        success {
            echo "Pipeline completed successfully! Inference validation passed."
        }

        failure {
            echo "Pipeline failed during validation."
        }
    }
}