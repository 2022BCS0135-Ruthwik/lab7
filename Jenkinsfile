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
                script {
                    echo "Pulling Docker image ${DOCKER_IMAGE}"
                    sh "docker pull ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    echo "Running container ${CONTAINER_NAME} on port ${PORT}"
                    // remove old container if exists
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh "docker run -d --rm --name ${CONTAINER_NAME} -p ${PORT}:8000 ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Wait for API') {
            steps {
                script {
                    echo "Waiting for API to become available..."
                    sh '''
                        timeout 30s bash -c 'until curl -s http://localhost:8000/docs > /dev/null; do echo "Waiting..."; sleep 2; done'
                        echo "API is up and running!"
                    '''
                }
            }
        }

        stage('Send Valid Request') {
            steps {
                script {
                    echo "Sending valid inference request..."
                    sh '''
                        response=$(curl -s -X POST http://localhost:8000/predict \\
                            -H "Content-Type: application/json" \\
                            -d @tests/valid_input.json)
                        echo "Valid Request Response: $response"
                        
                        # Validate response contains wine_quality and value is numeric
                        if ! echo "$response" | grep -qE '"wine_quality":\\s*[0-9]+(\\.[0-9]+)?'; then
                            echo "Validation failed: Response does not contain numeric 'wine_quality'"
                            exit 1
                        fi
                        echo "Valid request validation passed."
                    '''
                }
            }
        }

        stage('Send Invalid Request') {
            steps {
                script {
                    echo "Sending invalid inference request to test error handling..."
                    sh '''
                        response=$(curl -s -w "\\n%{http_code}" -X POST http://localhost:8000/predict \\
                            -H "Content-Type: application/json" \\
                            -d @tests/invalid_input.json)
                            
                        body=$(echo "$response" | sed -e '$ d')
                        http_code=$(echo "$response" | tail -n1)
                        
                        echo "Invalid Request Response Body: $body"
                        echo "Invalid Request HTTP Code: $http_code"
                        
                        if [ "$http_code" -eq 200 ]; then
                            echo "Validation failed: Invalid request returned 200 OK. Expected an error."
                            exit 1
                        fi
                        echo "Invalid request correctly generated an error response."
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Cleaning up container ${CONTAINER_NAME}..."
                sh "docker stop ${CONTAINER_NAME} || true"
            }
        }
        success {
            echo "Pipeline completed successfully! Inference validation passed."
        }
        failure {
            echo "Pipeline failed during one of the validation stages."
        }
    }
}
