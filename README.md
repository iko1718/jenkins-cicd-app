# CI/CD Pipeline Setup: Git to Kubernetes Automation

## Problem Statement

**CI/CD Pipeline Setup with Git, Jenkins, and Kubernetes**

### Scenario
You are tasked with setting up a **Continuous Integration/Continuous Deployment (CI/CD)** pipeline using **Git**, **Jenkins**, and **Kubernetes (K8s)**.  
The goal is to **automate** the process of building artifacts upon code commits to specific Git repositories and **deploying** them to designated Kubernetes clusters.



## 1. Project Overview and Goal

This document outlines the implementation of a modern, automated **Continuous Delivery** solution using the **Jenkins Declarative Groovy language**.  
The core objective is to **streamline the software delivery lifecycle**, ensuring that code changes committed to the Git repository are rapidly and reliably deployed to a designated Kubernetes cluster.

### Core Technologies

- **Source Code Management:** Git  
- **Automation Server:** Jenkins (using Declarative Pipeline Groovy Scripting)  
- **Deployment Target:** Kubernetes (via the native `kubectl` CLI)

### Solution Highlights

- **Adaptability and Scalability:**  
  - The pipeline uses environment variables, making it easy to configure for different repositories and target clusters.

- **Robust Credential Handling:**  
  - The script includes advanced logic to securely process and utilize Kubernetes credentials.  
  - Ensures a reliable and safe connection for deployment.

- **Automation:**  
  - Designed for **automatic execution via Webhooks** upon every code commit.



## 2. Prerequisites and Initial Setup (VM / Local Machine)

For the Jenkins pipeline to run successfully, the **underlying environment** and the **application repository** must be correctly prepared.



### 2.1 Host Machine Requirements

The **Jenkins agent (or VM)** must have the following essential tools installed to execute pipeline scripts:

- **Git:**  
  - Required for cloning the application source code from the repository.

- **Standard Linux Utilities:**  
  - Includes commands like `curl`, `chmod`, `sed`, `base64`, and `tr`.  
  - These tools are used for:  
    - Installing `kubectl`  
    - Handling encoded credentials  
    - Processing certificate data securely



### 2.2 Git Repository Structure

The application's **Git repository** must contain the following core files in its **root directory**:

- **`Jenkinsfile`**  
  - The main pipeline script containing all **Groovy logic** for CI/CD stages.

- **Kubernetes Manifests**  
  - Files that define how the application runs within the Kubernetes cluster:
  
  - **`deployment.yaml`**  
    - Defines the application container image, number of replicas, and deployment strategy.
  
  - **`service.yaml`**  
    - Defines how the application is exposed over the network  
    - Example: Using **NodePort** or **LoadBalancer** for external access.



### 2.3 Kubernetes Access Requirements

The **CI server (Jenkins)** must have authenticated access to the **Kubernetes cluster**.

**Requirements:**
- Possession of a valid **`kubeconfig`** file with sufficient permissions to:
  - Manage **Deployments**
  - Manage **Services**
  - Manage **Pods**

- The **Kubernetes API Server** must be:
  - Reachable from the Jenkins agent  
  - Configured with correct IP and port in the environment variables  
    - Example: `10.2.0.4:8443`
## ⚙️ 3. Jenkins Configuration (The CI Server Setup)

To enable Jenkins to interact securely with **Git** and **Kubernetes**, specific credentials must be configured, and the pipeline job must be properly set up.



### 3.1 Secure Credential Setup (Critical Step)

For secure communication with the Kubernetes cluster, the **`kubeconfig`** file must be stored as a **Jenkins Secret File Credential**.  
This ensures that sensitive authentication data is **not hardcoded** inside the pipeline script.

**Steps:**
1. **Navigate:**  
   - In the Jenkins UI, go to **Manage Jenkins → Manage Credentials**.
2. **Add Credential:**  
   - Select your **Global Domain** (or desired store) and click **Add Credentials**.
3. **Type:**  
   - Choose **Secret file**.
4. **File:**  
   - Upload your **raw Kubernetes `kubeconfig` file**.
5. **ID:**  
   - Set the Credential ID to **`kubeconfig`**.  
     *(This ID is referenced by the `KUBE_CONFIG_ID` variable in the `Jenkinsfile`.)*



### 3.2 Pipeline Job Creation

Once the credentials are added, create a new **Jenkins Pipeline Job** to connect the Git repository and execute the CI/CD pipeline.

**Steps:**
1. **Create New Item:**  
   - From the Jenkins Dashboard, click **New Item** and select **Pipeline**.
2. **General Configuration:**  
   - Enable **GitHub hook trigger for GITScm polling** if you plan to use **webhooks** for automatic build triggers upon code commits.
3. **Pipeline Configuration:**
   - **Definition:** Select **Pipeline script from SCM**.  
   - **SCM:** Choose **Git**.  
   - **Repository URL:** Enter your Git repository URL.  
     - Example: `https://github.com/iko1718/jenkins-cicd-app`
   - **Script Path:** Keep the default value: **`Jenkinsfile`**.



### 3.3 Environment Variables Setup

The flexibility of the Jenkins pipeline is driven by environment variables defined at the top of the **`Jenkinsfile`**.  
Before running the job, make sure these variables are correctly configured.

| **Variable** | **Purpose** | **Example Value** |
| :------------ | :----------- | :---------------- |
| `GIT_REPO` | The source Git repository URL containing application code. | `https://github.com/my-user/my-app.git` |
| `KUBE_CONFIG_ID` | Credential ID for accessing Kubernetes (`kubeconfig` file). | `kubeconfig` |
| `KUBE_SERVER_IP` | The reachable IP and port of the Kubernetes API Server. | `10.2.0.4:8443` |

## 4. Pipeline Implementation: The Jenkinsfile Logic

This section explains the logic and flow of the **Declarative Groovy Pipeline**, implemented within the `Jenkinsfile`.  
It describes how Continuous Integration (CI) and Continuous Deployment (CD) are achieved step by step.

---

### Environment Block Configuration

The flexibility of this Jenkins pipeline comes from its **Environment Variables**, defined at the top of the `Jenkinsfile`.  
These variables are consistently used across all pipeline stages, allowing for easy configuration and reuse.

| **Variable** | **Purpose** | **Example Value** |
| :------------ | :----------- | :---------------- |
| `GIT_REPO` | Source code repository URL. | `https://github.com/iko1718/jenkins-cicd-app` |
| `KUBE_CONFIG_ID` | Jenkins Credential ID for the kubeconfig file. | `kubeconfig` |
| `KUBE_SERVER_IP` | IP address and port of the Kubernetes API server. | `10.2.0.4:8443` |
| `KUBE_API_ENDPOINT` | Full API server address used for cluster connection. | `https://10.2.0.4:8443` |
| `KUBECONFIG_CLEAN_PATH` | Internal path for the clean, generated kubeconfig file. | `kubeconfig_clean.yaml` |

These variables make the pipeline **adaptable** and **scalable** for different repositories and Kubernetes clusters without modifying the logic in every stage.

---

### Stage 1: Build & Setup (Continuous Integration)

This stage prepares the Jenkins agent environment for interaction with Kubernetes.

**Key Steps:**

- **Tool Installation:**  
  - Uses `curl` to dynamically fetch the **latest stable kubectl binary**, ensuring the Jenkins agent is self-sufficient for K8s interaction.  
  - Example command:  
    ```bash
    curl -LO "https://storage.googleapis.com/kubernetes-release/release/$KUBE_VERSION/bin/linux/amd64/kubectl"
    chmod +x ./kubectl
    ```

- **Artifact Simulation:**  
  - Instead of an actual build, this example simulates the **core CI process**:  
    - Code compilation  
    - Unit testing  
    - Docker image building (placeholder for `docker build -t myapp:$BUILD_ID .`)  
  - Marks the step as complete before proceeding to deployment.

 **Outcome:**  
A ready environment with `kubectl` installed and validated.

---

### Stage 2: Initialize Kubeconfig (Critical Fix)

This stage is **vital** for ensuring secure and reliable communication with the Kubernetes cluster.  
It solves a **common CI/CD issue** where Jenkins file-based secrets may introduce whitespace or encoding errors in base64 certificate data, leading to *TLS PEM corruption* errors.

**Key Steps:**

1. **Credential Loading:**  
   - The `withCredentials` block loads the **raw kubeconfig secret** from Jenkins into a temporary file (`KUBECONFIG_SOURCE`).

2. **Secure Extraction & Cleaning:**  
   - Uses shell commands with `sed` and `tr -d '[:space:]'` to clean **base64-encoded certificate data**, removing all whitespace and invalid formatting.  
   - Fixes errors such as:
     ```
     tls: failed to find any PEM data in key input
     ```

3. **File Generation:**  
   - The cleaned data is decoded using `base64 -d` into three separate files:  
     - `ca.crt`  
     - `client.crt`  
     - `client.key`

4. **Final Kubeconfig Creation:**  
   - Generates a new, clean kubeconfig file (`kubeconfig_clean.yaml`) referencing these decoded files for authentication.  
   - Updates the server address dynamically using:
     ```bash
     sed -i 's|server: https://.*|server: ${env.KUBE_API_ENDPOINT}|' ${KUBECONFIG_CLEAN_PATH}
     ```

5. **Connection Test:**  
   - Validates connectivity using:
     ```bash
     ./kubectl cluster-info --insecure-skip-tls-verify
     ```
   - Ensures Jenkins can successfully communicate with the Kubernetes cluster before deployment.

**Outcome:**  
A verified, working kubeconfig enabling secure cluster access.

---

### Stage 3: Deploy Application (Continuous Deployment)

This is the **deployment** stage where the verified configuration is used to deploy the application to Kubernetes.

**Key Steps:**

- **Kubeconfig Loading:**  
  - The `KUBECONFIG` environment variable is set to the path of the clean file generated in Stage 2.  
  - Ensures all subsequent `kubectl` commands use the secure configuration.

- **Manifest Application:**  
  - Executes:
    ```bash
    ./kubectl apply -f deployment.yaml
    ./kubectl apply -f service.yaml
    ```
  - Applies both Kubernetes manifests:
    - `deployment.yaml` → Defines application container, replicas, and image.  
    - `service.yaml` → Defines how the app is exposed (e.g., NodePort, LoadBalancer).

- **Validation:**  
  - Confirms whether manifests exist before applying, preventing runtime errors.  
  - Logs success or warnings accordingly.

**Outcome:**  
The application is deployed successfully to the Kubernetes cluster.

## 5. Validation, Monitoring, and Security Recommendations

This section describes the procedures required to validate the CI/CD pipeline, ensure correct functionality, and maintain long-term reliability and security of the Jenkins–Kubernetes deployment environment.

### 5.1. Test Cases and Validation Procedures

To confirm that the CI/CD automation works as intended, the following validation tests must be performed after the first successful pipeline execution.

| **Procedure** | **Stage Validated** | **Expected Outcome** |
| :------------- | :----------------- | :------------------- |
| Commit Trigger Test | Entire Pipeline | A new Jenkins build should start automatically when a code change is pushed to the configured Git branch. |
| Kubeconfig Connection Test | Initialize Kubeconfig | The pipeline logs should show a successful `kubectl cluster-info` result without any certificate or PEM data errors. |
| Deployment Existence | Deploy Application | Running `kubectl get deployment <app-name>` should confirm that the deployment exists with the expected replica count. |
| Pod Health Check | Deploy Application | Running `kubectl get pods` should list all application pods in the **Running** state with a READY count of **1/1** (or the expected number). |
| External Service Access | Deploy Application | When using a LoadBalancer or NodePort Service, the application should be reachable and functional via its external IP or URL. |

These tests validate that both the Continuous Integration (CI) and Continuous Deployment (CD) aspects of the pipeline are functioning correctly from end to end.



### 5.2. Recommendations for Monitoring and Logging

Monitoring and logging are critical for ensuring ongoing reliability, identifying performance issues, and diagnosing deployment failures quickly.

#### Monitoring Strategies

| **Area** | **Tool or Strategy** | **Purpose** |
| :-------- | :------------------ | :----------- |
| Pipeline Health | Jenkins Stage View | Displays a visual timeline of each pipeline stage (Build, Initialize, Deploy). Helps monitor stage durations and identify failures or slowdowns. |
| Application Metrics | Prometheus and Grafana | Collects metrics on CPU, memory, network usage, and application-specific data such as latency and error rates within the Kubernetes cluster. |
| Cluster Health | `kubectl get events` | Tracks real-time Kubernetes events during deployment, helping to identify issues like image pull errors, insufficient node resources, or pod scheduling failures. |

#### Logging Strategies

| **Log Type** | **Implementation** | **Best Practice** |
| :------------ | :----------------- | :---------------- |
| Pipeline Logs | `echo` commands and Jenkins console output | Use descriptive `echo` statements such as “Starting Stage X…” in the Jenkinsfile to clearly indicate progress. Capture all `sh` command output for detailed execution tracing. |
| Application Logs | Centralized Logging (ELK / Loki Stack) | Configure the application to write logs to stdout/stderr. Use Fluentd, Logstash, or Loki to collect logs across containers and centralize them for analysis. |
| Error Handling | Built-in Jenkins error reporting and log inspection | Ensure that all critical shell commands include proper exit checks. Log detailed error messages for quick identification and resolution of failures. |



### 5.3. Security Best Practices

To maintain a secure, production-ready pipeline, follow these key security principles throughout the CI/CD workflow:

1. **Credential Masking**  
   Always use Jenkins’ `withCredentials` block when handling secrets such as the kubeconfig file. Jenkins automatically masks sensitive information in the console output, preventing accidental exposure of tokens or keys.

2. **Principle of Least Privilege (Kubernetes)**  
   The Service Account referenced in the kubeconfig file should have only the minimum required RBAC permissions. It should be limited to operations necessary for deployment (e.g., `get`, `create`, `update` on Deployments and Services) and should not have administrative privileges.

3. **Workspace Cleanup**  
   The `post` block in the Jenkinsfile ensures that temporary files like `ca.crt`, `client.crt`, `client.key`, and `kubeconfig_clean.yaml` are deleted after each run. This step is essential to prevent residual sensitive data from being accessed by unauthorized users or processes.

4. **Secure Communication**  
   All connections between Jenkins and the Kubernetes API server should use HTTPS with properly verified certificates. If `--insecure-skip-tls-verify` is used during testing, ensure that it is disabled in production environments.

5. **Access Control and Auditing**  
   Restrict access to the Jenkins dashboard and credentials to authorized personnel only. Enable auditing in both Jenkins and Kubernetes to track user actions, credential usage, and deployment activity.
---

This completes the end-to-end documentation for **CI/CD Pipeline Setup with Git, Jenkins, and Kubernetes**.





