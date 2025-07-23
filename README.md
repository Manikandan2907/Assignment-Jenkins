# Jenkins CI/CD Pipeline – Assignment Project

This repository contains Jenkins pipeline scripts that automate the CI/CD process for a Spring Boot application hosted in another GitHub repository: [`Assignment-app`]([https://github.com/Manikandan2907/assignment-app](https://github.com/Manikandan2907/Assignment-App.git)). The automation covers code testing, static code analysis, containerization, and deployment(Only manual step, as it requires an approval) to a remote EC2 server.

## Why Three Pipelines?

Three separate pipelines were created to handle different scenarios: PR validation(`Runs on any branch`), main branch builds(`Runs only when the branch is main`), and production deployments(`Needs approval at the runtime`) with approval.  
This approach ensures clean separation of logic and better control over each stage.

---

## 🔧 Project Overview

This project defines **three Jenkins pipelines** with the following responsibilities:

1. **Pull Request Validation Pipeline**
2. **Main Branch Build & Push Pipeline**
3. **Manual Deployment Pipeline**

All three pipelines use the `assignment-app` repository as the application source.

---

## 📂 Pipeline Breakdown

### 1. ✅ Pull Request Validation Pipeline

**Trigger:**  
Automatically triggered via a GitHub webhook on `Assignment-app` when a **Pull Request (PR)** is opened.

**Steps:**
- Webhook sends a payload containing repository URL and branch name.
- Jenkins parses the payload via **Generic Webhook Trigger**.
- Clones the source code from the PR branch.
- Runs **unit and integration tests** using the Maven wrapper (`mvnw`).
- Performs **static code analysis** using **SonarQube**.
  - SonarQube is hosted as a Docker container on the same Jenkins server.
- Marks build status based on test and analysis results.

**Outcome:**  
Ensures the PR meets code quality and test standards before merge.

---

### 2. 🚀 Main Branch Build & Push Pipeline

**Trigger:**  
Triggered automatically via a second GitHub webhook **when code is merged to `main` branch** of the `assignment-app` repo.

**Steps:**
- Extracts repository URL and branch info from webhook payload.
- Clones the main branch and **builds the code**, skipping tests.
- Packages the application as a **JAR file**.
- Builds a **Docker image** using this JAR.
- Scans the Docker image using **[Trivy](https://github.com/aquasecurity/trivy)** for vulnerabilities (HIGH/CRITICAL).
- Pushes the scanned Docker image to **Docker Hub**.
- Pushes the Docker image to **Docker Hub**:
  - Docker Hub credentials are securely stored in Jenkins credentials store.

**Outcome:**  
Produces and distributes a production-ready Docker image for deployment.

---

### 3. 🚢 Manual Deployment Pipeline

**Trigger:**  
This is a **manually-triggered** Jenkins job, not triggered by webhook.

**Parameters:**
- Accepts an environment input: `stage` or `prod` as choice.

**Steps:**
- Connects to a remote EC2 server via SSH (using Jenkins credentials).
- The SSH Key is stored in Jenkins. 
- Pulls the Docker image from Docker Hub.
- Stops any existing container with the same name.
- Starts a new container using the pulled image.
- For `prod` environment:
  - Prompts a manual approval step before deployment proceeds.

**Outcome:**  
Deploys the latest image to the EC2 server in either staging or production mode.

---

## 🧰 Technologies Used

| Tool         | Purpose                                         |
|--------------|--------------------------------------------------|
| Jenkins      | CI/CD orchestration                             |
| Maven Wrapper (`mvnw`) | Builds & tests Spring Boot application|
| SonarQube    | Static code analysis                            |
| Docker       | Containerization of the application             |
| Docker Hub   | Container registry for storing images           |
| Trivy        | Container image vulnerability scanning (HIGH/CRITICAL)|
| GitHub Webhooks | Triggers Jenkins pipelines on PR and merge events |
| EC2 (Ubuntu) | Remote server for deployment                    |

---

## 🔐 Security & Credentials

- All sensitive values (e.g., Docker Hub credentials, Sonarqube Credentials and SSH private keys) are securely stored in **Jenkins Credentials Manager**.
- Remote deployment uses `sshagent` and `withCredentials` for secure communication.

---

## 📈 Workflow Summary

```mermaid
graph TD
    A[Developer Raises PR] --> B[PR Pipeline Triggered]
    B --> C[Test + SonarQube Analysis]
    C --> D[Build Success or Fail]
    D --> E[PR Review & Merge to Main]
    E --> F[Main Branch Pipeline Triggered]
    F --> G[Build Jar, Dockerize, Trivy Image Scan]
    G --> H[Push to Dockerhub]
    H --> I[Manual Deployment Pipeline Triggered]
    I --> J{Stage or Prod?}
    J -->|Stage| K[Deploy to EC2]
    J -->|Prod| K[Manual Approval -> Deploy to EC2]
