# Petclinic CI/CD Pipeline Documentation

Welcome to the documentation for the Petclinic application's CI/CD pipeline. This document provides comprehensive information on the pipeline processes, tools, configurations, and best practices.

## Table of Contents
1. [Introduction](#introduction)
    - [Project Overview](#project-overview)
    - [Purpose of the CI/CD Pipeline](#purpose-of-the-cicd-pipeline)
2. [Pipeline Architecture](#pipeline-architecture)
    - [High-Level Architecture](#high-level-architecture)
    - [Tools and Technologies](#tools-and-technologies)
3. [Server Setup for Pipeline](#server-setup-for-pipeline)
    - [Why To Use Kubernetes](#why-to-use-kubernetes)
    - [Jenkins on Kubernetes](#jenkins-on-kubernetes)
    - [SonarQube on Kubernetes](#sonarqube-on-kubernetes)
    - [Sonatype Nexus on Kubernetes](#sonatype-nexus-on-kubernetes)
4. [Pipeline Configuration](#pipeline-configuration)
    - [Branching Strategy](#branching-strategy)
    - [Triggers](#triggers)
    - [Environment Variables](#environment-variables)
5. [Pipeline Stages](#pipeline-stages)
    - [Stage 1. Checkout SCM](#stage-1-checkout-scm)
    - [Stage 2. Maven Build](#stage-2-maven-build)
    - [Stage 3. SonarQube Scan](#stage-3-sonarqube-scan)
    - [Stage 4. Quality Gate](#stage-4-quality-gate)
    - [Stage 5. Publish Artifacts to Nexus](#stage-5-publish-artifacts-to-nexus)
    - [Stage 6. Build and Push Docker Image](#stage-6-build-and-push-docker-image)
    - [Stage 7. Update Chart Version](#stage-7-update-chart-version)
    - [Stage 8. Package Helm Chart](#stage-8-package-helm-chart)
    - [Stage 9. Publish Helm Chart to Nexus](#stage-9-publish-helm-chart-to-nexus)
    - [Stage 10. Release Helm Chart](#stage-10-release-helm-chart)

6. [Monitoring and Logging](#monitoring-and-logging)
    - [Monitoring Tools](#monitoring-tools)
    - [Logging](#logging)
7. [Troubleshooting](#troubleshooting)
    - [Common Issues](#common-issues)
    - [Error Handling](#error-handling)
8. [Best Practices](#best-practices)
    - [CI/CD Best Practices](#cicd-best-practices)
    - [Pipeline Optimization](#pipeline-optimization)


## Introduction

### Project Overview
The Petclinic application is a sample Spring Boot application designed to demonstrate the capabilities of the Spring framework. I'am going to design and implement a robust CI/CD pipeline to automate the build, test and deployment of the application.

### Purpose of the CI/CD Pipeline
The CI/CD pipeline for the Petclinic application is designed to automate the processes of building, testing, and deploying the application, leading to faster and more reliable software delivery.

## Pipeline Architecture

### High-Level Architecture
![Pipeline Diagram](path/to/diagram.png)
This high-level architecture provides an overview of the stages involved in the pipeline, from source control integration to deployment, with considerations for testing, security, and monitoring at each step.

### Tools and Technologies
- **Git:** Widely used distributed version control system for tracking changes in source code files.
- **Jenkins:** Continuous Integration and Continuous Deployment (CI/CD) automation server, allowing for the automation of building, testing, and deploying software.
- **SonarQube:** Code quality and security analysis tool that identifies issues such as bugs, vulnerabilities, and code smells in codebases.
- **Nexus:** Repository manager for storing and managing binary components such as Docker images, JAR files, and other artifacts.
- **Docker:** Platform for developing, shipping, and running applications using containerization, ensuring consistency across different environments.
- **Kubernetes:** Container orchestration platform for automating the deployment, scaling, and management of containerized applications.
- **Helm:** Kubernetes package manager simplifying the deployment and management of applications on Kubernetes clusters through reusable templates called charts.

## Server Setup for Pipeline

Setting up a server for Continuous Integration and Continuous Deployment (CI/CD) involves creating an environment where code changes are automatically tested, integrated, and deployed. This process typically includes setting up a version control system (like Git), a CI/CD platform (such as Jenkins, GitLab CI, or GitHub Actions), and necessary build and deployment tools (like Docker, Kubernetes). Proper configuration of webhooks, environment variables, and security measures (like SSH keys and access controls) is essential to ensure a seamless and secure CI/CD pipeline.

**Note:** I will be using Kubernetes for CICD Pipeline

### Why To Use Kubernetes
Kubernetes (K8s) is essential for modern app deployment, offering powerful orchestration for containers. It automates deployment, scaling, and management, ensuring consistency across environments. Its robust ecosystem and wide adoption make it a key tool in cloud-native development, enabling efficient and reliable application management.

Using Kubernetes for your CI/CD pipeline infrastructure provides robust, scalable, and flexible management capabilities, making it an ideal choice for modern, cloud-native environments. It helps in automating and streamlining the CI/CD process, ensuring efficient use of resources and providing high availability and security for your development workflows.

- **Automatic Scaling:** Kubernetes can automatically scale your CI/CD tools (like Jenkins agents) based on load, ensuring efficient resource utilization.
- **High Availability:** Kubernetes can manage multiple replicas of your services, ensuring they are always available even if some instances fail.
- **Container Orchestration:** Kubernetes efficiently manages containerized applications, ensuring optimal resource use across the cluster.
- **Resource Limits:** You can set resource limits and requests for containers to ensure that CI/CD tools do not exceed allocated resources.
- **Isolation:** Each stage of the CI/CD pipeline can run in isolated containers, improving security and reliability.

### Jenkins on Kubernetes
- Jenkins on Kubernetes is the integration of the Jenkins automation server with the Kubernetes container orchestration platform. This combination streamlines the deployment, scaling, and management of Jenkins instances and build environments.
- Jenkins leverages Kubernetes’ dynamic resource allocation to create `ephemeral build agents` (slaves) as containers, which are automatically scaled and terminated based on demand. This integration provides flexibility, increased resource efficiency, and reduced operational overhead.

### Jenkins Installation

#### Prerequisites
- ***Kubernetes Cluster:*** A running Kubernetes cluster.
- ***kubectl:*** Kubernetes command-line tool installed and configured to interact with your cluster.
- ***Helm:*** Helm package manager installed.

### 1. Install Jenkins with Helm
```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
# Use NodePort or LoadBalancer Service to access from browser
helm install jenkins jenkins/jenkins --set controller.serviceType=LoadBalancer
```
Retrieve the Jenkins Initial Password
```bash
kubectl get secret --namespace default jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode
```
Access Jenkins via the Kubernetes cluster's external IP or domain at http://<external-ip>:8080.

### 2. Configure Kubernetes Plugin & Kubernetes in Jenkins
- **Install Kubernetes Plugin:** Go to Manage Jenkins > Manage Plugins > Install the `Kubernetes plugin`.

- **Configure Kubernetes Cloud:** Go to Manage Jenkins > Manage Nodes and Clouds > Configure Clouds > Add a new cloud > Select `Kubernetes`> Configure the Kubernetes cloud

**Note:** Since Helm is used to install jenkins, all necessary configuration will automatically be filled in jenkins server. 


## SonarQube on Kubernetes

#### Step 1: Install SonarQube with Helm

```bash
helm repo add sonarqube https://github.com/SonarSource/helm-chart
helm repo update
# Use NodePort or LoadBalancer Service to access from browser
helm install sonarqube sonarqube/sonarqube --set controller.serviceType=NodePort
```
#### Step 2: Setup SonarQube
- **Access SonarQube:** Open your browser and go to `http://You_IP:9000`. Log in with the default credentials `(admin/admin)`.

- **Create a Project in SonarQube:** Navigate to the `Projects` tab > Click on `Create Project` > Provide a project key and a project name, then click `Set Up`.

- **Generate a New Token:** My Account > Security > Token > Generate Token and Save it somewhere to in integrate with jenkins

- **Set Up Webhooks in SonarQube**:
  - Go to the "Administration" menu by clicking on the gear icon in the top-right corner.
  - Select "Configuration" -> "Webhooks".
  - lick on the "Create" button to add a new webhook.
  - Provide a name for the webhook.
  - Enter the URL of your webhook endpoint. For Jenkins, this would typically be in the format: `http://<JENKINS_URL>/sonarqube-webhook/`
  - Click "Save" to create the webhook.



#### Step 3: Integrate SonarQube with Jenkins
- **Add Credentials:** Manage Jenkins > Credentials > Select 'Secret text' > Add any ID > Put the `SonarQube token` in Secret.

- **Install Required Plugins:** Manage Jenkins > Manage Plugins > Avilable plugins > Search `SonarQube Scanner` and install it.

- **Configure SonarQube in Jenkins:** Manage Jenkins > Configure System > Scroll down to the "SonarQube servers" section and click "Add SonarQube" >
Provide a `name` for the SonarQube server > Enter the SonarQube server URL (e.g., http://Your_IP:9000) > Add the `Credential` generated earlier.

## Sonatype Nexus on Kubernetes

#### Step 1: Install Nexus with Helm
```bash
helm repo add sonatype https://sonatype.github.io/helm3-charts/
helm repo update
# Use NodePort or LoadBalancer Service to access from browser
helm install nexus sonatype/nexus-repository-manager --set controller.serviceType=LoadBalancer
```
Get Initial Password
```
kubectl exec -it <nexus-pod> -- bash
cat /nexus-data/admin.password
```
#### Step 2: Setup Nexus
- **Access Nexus:** Visit on `http://You_IP:8081` > Log in with the credentials `username: admin`, `password: above one` > Change Username and Password.

- **Create Maven Hosted Repository:**
  - Select Maven (hosted): Choose "maven2 (hosted)" from the list of repository formats.
  - Configure the Repository: Name, Version policy(Release/Snapshot), Layout policy(Strict/Permissive)
  - Other options can be configured as needed, such as deployment policy, write policy, etc.
  - Save the Repository

#### Steps 3: Integrate Nexus with Jenkins
- **Add Credentials:** Manage Jenkins > Credentials > Select 'Username and password' > Add Nexus's Username > Add  Nexus's Password.
- **Install Plugins:** Go to Manage Jenkins -> Manage Plugins. -> Install following plugins
  - `Nexus Artifact Uploader`: This plugin is used to upload artifacts to Nexus.
  - `Pipeline Utility Steps`: If you're using Jenkins pipelines, this plugin provides various steps including file operations(e.g. reading pom.xml)

