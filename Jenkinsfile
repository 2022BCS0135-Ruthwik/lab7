pipeline {
    agent any

    environment {
        DOCKER_IMAGE = '2022bcs0135ruthwik/wine_mlops_lab4_2022bcs0135:latest'
        CONTAINER_NAME = 'wine-mlops-api'
    }

    stages {

        stage('Pull Docker Image') {
            steps {
                sh 'docker pull ${DOCKER_IMAGE}'
            }
        }

        stage('Run Container') {
            steps {
                sh '''
                docker rm -f ${CONTAINER_NAME} || true
                docker run -d --name ${CONTAINER_NAME} -p 8000:8000 ${DOCKER_IMAGE}

                echo "Running containers:"
                docker ps
                sleep 5
                '''
            }
        }

        stage('Wait for API') {
            steps {
                sh '''
                echo "Checking API inside container..."

                i=1
                while [ $i -le 20 ]; do
                    docker exec ${CONTAINER_NAME} curl -s http://localhost:8000/predict \
                    -X POST \
                    -H "Content-Type: application/json" \
                    -d @tests/valid_input.json \
                    > /dev/null && echo "API ready" && exit 0

                    echo "Waiting for API..."
                    sleep 2
                    i=$((i + 1))
                done

                echo "API failed to start"
                docker logs ${CONTAINER_NAME}
                exit 1
                '''
            }
        }

        stage('Send Valid Request') {
            steps {
                sh '''
                RESPONSE=$(docker exec ${CONTAINER_NAME} curl -s \
                -X POST http://localhost:8000/predict \
                -H "Content-Type: application/json" \
                -d @tests/valid_input.json)

                echo "Valid Response:"
                echo "$RESPONSE"

                echo "$RESPONSE" | grep wine_quality
                '''
            }
        }

        stage('Send Invalid Request') {
            steps {
                sh '''
                RESPONSE=$(docker exec ${CONTAINER_NAME} curl -s -w "\\n%{http_code}" \
                -X POST http://localhost:8000/predict \
                -H "Content-Type: application/json" \
                -d @tests/invalid_input.json)

                BODY=$(echo "$RESPONSE" | sed '$d')
                CODE=$(echo "$RESPONSE" | tail -n1)

                echo "Invalid Response Body:"
                echo "$BODY"

                echo "HTTP Code:"
                echo "$CODE"

                if [ "$CODE" -eq 200 ]; then
                    echo "Invalid request incorrectly succeeded"
                    exit 1
                fi
                '''
            }
        }
    }

    post {
        always {
            sh '''
            docker stop ${CONTAINER_NAME} || true
            docker rm ${CONTAINER_NAME} || true
            '''
        }
    }
}