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
                    docker run -d --name ${CONTAINER_NAME} -p ${PORT}:8000 ${DOCKER_IMAGE}
                    echo "Container started:"
                    docker ps --filter "name=${CONTAINER_NAME}"
                    echo "Sleeping briefly to let process initialize..."
                    sleep 5
                    echo "Initial container logs (tail):"
                    docker logs --tail 100 ${CONTAINER_NAME} || true
                '''
            }
        }

        stage('Wait for API') {
            steps {
                echo "Waiting for API to become responsive inside the container..."
                sh '''
                    # Try up to 60 seconds (30 attempts x 2s)
                    for i in {1..30}; do
                        # run a tiny Python GET inside the container (no curl required)
                        docker exec ${CONTAINER_NAME} python - <<'PY' >/dev/null 2>&1 && rc=$? || rc=$?
import urllib.request, sys
try:
    urllib.request.urlopen("http://127.0.0.1:8000", timeout=2)
    sys.exit(0)
except Exception:
    sys.exit(1)
PY
                        if [ "$rc" -eq 0 ]; then
                            echo "API inside container is up (attempt $i)."
                            exit 0
                        fi
                        echo "Waiting for API... (attempt $i)"
                        sleep 2
                    done

                    echo "API failed to become ready. Container logs:"
                    docker logs ${CONTAINER_NAME} || true
                    exit 1
                '''
            }
        }

        stage('Send Valid Request') {
            steps {
                echo "Sending valid inference request (piping workspace file into container python)..."
                sh '''
                    # Pipe the workspace tests/valid_input.json into Python running inside the container
                    cat tests/valid_input.json | docker exec -i ${CONTAINER_NAME} python - <<'PY'
import sys, json, urllib.request
data = sys.stdin.read().encode('utf-8')
req = urllib.request.Request("http://127.0.0.1:8000/predict", data=data, headers={"Content-Type":"application/json"})
try:
    resp = urllib.request.urlopen(req, timeout=10)
    body = resp.read().decode('utf-8')
    print(body)
    # quick validation: must contain "wine_quality"
    if '"wine_quality"' not in body:
        print("VALIDATION ERROR: 'wine_quality' not present in response")
        raise SystemExit(2)
except Exception as e:
    print("REQUEST ERROR:", e)
    raise SystemExit(1)
PY
                '''
            }
        }

        stage('Send Invalid Request') {
            steps {
                echo "Sending invalid inference request to test error handling..."
                sh '''
                    # Pipe the workspace tests/invalid_input.json into Python inside the container,
                    # capture status by checking for an HTTP 200 (which would be a failure for this test).
                    # We'll use urllib and print both body and status.
                    cat tests/invalid_input.json | docker exec -i ${CONTAINER_NAME} python - <<'PY'
import sys, json, urllib.request, urllib.error
data = sys.stdin.read().encode('utf-8')
req = urllib.request.Request("http://127.0.0.1:8000/predict", data=data, headers={"Content-Type":"application/json"})
try:
    resp = urllib.request.urlopen(req, timeout=10)
    body = resp.read().decode('utf-8')
    print("BODY:", body)
    print("STATUS: 200")
    # If we got here, invalid input returned 200 -> treat as test failure
    raise SystemExit(2)
except urllib.error.HTTPError as e:
    # Expected path for invalid input: an HTTP error (4xx/5xx)
    try:
        err_body = e.read().decode('utf-8')
        print("ERROR BODY:", err_body)
    except:
        print("ERROR: (no body)")
    print("STATUS:", e.code)
    # success for this stage (invalid input produced non-200)
    raise SystemExit(0)
except Exception as e:
    print("REQUEST ERROR:", e)
    raise SystemExit(1)
PY
                '''
            }
        }
    }

    post {
        always {
            echo "Cleaning up container ${CONTAINER_NAME}..."
            sh '''
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
            '''
        }
        success {
            echo "Pipeline completed successfully! Inference validation passed."
        }
        failure {
            echo "Pipeline failed during one of the validation stages."
        }
    }
}