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





