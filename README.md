# Lab 7: Automated Inference Validation Pipeline

## Student Details
- **Name:** Mode Ruthwik
- **Roll Number:** 2022BCS0135

## Project Overview
This repository (`lab7`) implements a Jenkins CI/CD pipeline designed to perform automated inference validation on a deployed Machine Learning API. The API is a FastAPI application that predicts wine quality, pre-packaged in a Docker image hosted on Docker Hub.

## Pipeline Workflow
The Jenkins pipeline defined in the `Jenkinsfile` executes the following automated stages to safely validate the deployed machine learning model inference:

1. **Pull Docker Image:** 
   Connects to Docker Hub to pull the latest version of the target prediction API image (`2022bcs0135ruthwik/wine-mlops-2022bcs0135_lab6:latest`).

2. **Run Container:** 
   Starts the FastAPI application container in detached mode, exposing it on port `8000`.

3. **Wait for API:** 
   Actively polls the `/docs` endpoint of the API up to 30 seconds to ensure the application has fully initialized and is ready to accept HTTP traffic before proceeding. 

4. **Send Valid Request:** 
   Reads telemetry data from `tests/valid_input.json` and submits a `POST` request to the `/predict` endpoint. It prints the API response to the console log, and uses shell scripts to validate that the returned JSON object contains a numeric `wine_quality` property. If validation fails, the pipeline errors out.

5. **Send Invalid Request:** 
   Submits intentionally malformed data from `tests/invalid_input.json` (breaking the schema with string instead of numeric inputs) to the `/predict` endpoint. It checks the HTTP response code to ensure the machine learning API gracefully handles bad data without returning a 200 OK status. Responses are logged.

6. **Cleanup:** 
   Uses Jenkins `post { always { ... } }` declarative configurations to guarantee the detached Docker container is stopped and removed, ensuring no background processes leak regardless of whether the pipeline succeeded or failed.

## File Structure
- `tests/valid_input.json`: Valid JSON payload matching the expected schema.
- `tests/invalid_input.json`: Invalid JSON payload structured to trigger an API validation error.
- `Jenkinsfile`: Defines the automated pipeline logic.
- `README.md`: This file explaining the pipeline workflow and contents.
