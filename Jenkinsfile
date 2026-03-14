pipeline {
  agent any

  environment {
    DOCKER_IMAGE    = '2022bcs0135ruthwik/wine_mlops_lab4_2022bcs0135:latest'
    CONTAINER_NAME  = 'wine-mlops-api'
    PORT            = '8000'
  }

  stages {
    stage('Pull Image') {
      steps {
        echo "Pulling ${DOCKER_IMAGE}"
        sh 'docker pull ${DOCKER_IMAGE}'
      }
    }

    stage('Run Container') {
      steps {
        echo "Starting container ${CONTAINER_NAME}"
        sh '''
          docker rm -f ${CONTAINER_NAME} || true
          docker run -d --name ${CONTAINER_NAME} -p ${PORT}:8000 ${DOCKER_IMAGE}
          echo "Container status:"
          docker ps --filter "name=${CONTAINER_NAME}"
          sleep 3
          echo "Container logs (tail):"
          docker logs --tail 100 ${CONTAINER_NAME} || true
        '''
      }
    }

    stage('Wait for API') {
      steps {
        echo "Waiting for API on http://localhost:${PORT}"
        sh '''
          for i in {1..30}; do
            if curl -s --fail http://localhost:${PORT} >/dev/null 2>&1; then
              echo "API is up (attempt $i)"
              exit 0
            fi
            echo "Waiting for API... (attempt $i)"
            sleep 2
          done
          echo "API did not become ready; container logs:"
          docker logs ${CONTAINER_NAME} || true
          exit 1
        '''
      }
    }

    stage('Send Valid Request') {
      steps {
        echo "Sending valid request to /predict"
        sh '''
          RESPONSE=$(curl -s -X POST http://localhost:${PORT}/predict \
            -H "Content-Type: application/json" \
            -d @tests/valid_input.json)

          echo "Response: $RESPONSE"

          if ! echo "$RESPONSE" | grep -q '"wine_quality"'; then
            echo "Validation failed: 'wine_quality' not found in response"
            exit 1
          fi

          echo "Valid request passed"
        '''
      }
    }

    stage('Send Invalid Request') {
      steps {
        echo "Sending invalid request to confirm error handling"
        sh '''
          FULL=$(curl -s -w "\\n%{http_code}" -X POST http://localhost:${PORT}/predict \
            -H "Content-Type: application/json" \
            -d @tests/invalid_input.json)

          BODY=$(echo "$FULL" | sed '$d')
          CODE=$(echo "$FULL" | tail -n1)

          echo "Invalid-response body: $BODY"
          echo "HTTP status: $CODE"

          if [ "$CODE" -eq 200 ]; then
            echo "Validation failed: invalid input returned 200"
            exit 1
          fi

          echo "Invalid input correctly returned non-200 ($CODE)"
        '''
      }
    }
  }

  post {
    always {
      echo "Stopping and removing container ${CONTAINER_NAME}"
      sh '''
        docker stop ${CONTAINER_NAME} || true
        docker rm ${CONTAINER_NAME} || true
      '''
    }
    success {
      echo "Pipeline succeeded: inference validation passed."
    }
    failure {
      echo "Pipeline failed: see console output for details."
    }
  }
}