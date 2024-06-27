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
    - [Ingress Controller to Expose Services](#ingress-controller-to-expose-services)
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

6. [Monitoring](#monitoring)
    - [Monitoring Tools](#monitoring-tools)




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

### Ingress Controller to Expose Services
I have used Ingress Controller for Exposing services like Jenkins, SonarQube, and Nexus via an Ingress controller in Kubernetes which offers several advantages:
- Ingress provides a `single entry point` for managing external access to multiple services within your Kubernetes cluster. Instead of exposing each service individually with `NodePort`or `LoadBalancer` services, use Ingress to consolidate routing and SSL termination at the Ingress controller level. This reduces the number of `external IP addresses` and `load balancers` needed, thus cutting down on associated costs`.

- Ingress allows you to define rules (based on hostnames, paths, etc.) to route traffic to different services within your cluster. This simplifies traffic management and load balancing across applications.
![Image Alt Text](https://www.checkmateq.com/blog/wp-content/uploads/2022/10/Hire-Dedicated-Cloud-Engineer5-1.png)


**Ingress Manifest:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: custom-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"  # Add this line
spec:
  ingressClassName: nginx
  rules:
  - host: jenkins.klouds.site
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080
  - host: sonar.klouds.site
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sonarqube-sonarqube
            port:
              number: 9000
  - host: nexus.klouds.site
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nexus-nexus-repository-manager
            port:
              number: 8081

```
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

## Pipeline Configuration
Pipeline configuration in the context of CI/CD involves setting up automated workflows that are triggered by specific events. In this case, using a `GitHub webhook` is a common approach to trigger these pipelines.

- A `webhook` is a way for an application to provide other applications with real-time information. It delivers data to other applications as it happens, meaning you get data immediately.
- When an event(push, pull_request, issue etc) that you subscribed to occurs, GitHub sends an `HTTP POST` request to the URL you specified. This request contains a JSON payload with details about the event.
- The server at the specified URL (often a CI/CD tool like `Jenkins`, CircleCI, Travis CI, etc.) receives the payload and triggers the appropriate pipeline or workflow.

### Branching Strategy
In this project, I assume `Trunk-Based Development`, a branching strategy for version control systems where all developers integrate their changes into a single main branch (often called "trunk" or "main") frequently, is followed. If a feature branch is necessary, create it off the main branch, work on it briefly, and merge it back into the main branch as soon as possible, ensuring Short-Lived Feature Branches.

Trunk-Based Development is a strategy that promotes collaboration, continuous integration, and high code quality, making it well-suited for modern agile and DevOps practices.

### Triggers
Triggers in a CI/CD pipeline are mechanisms that automatically start the pipeline's execution based on specific `events`. Properly configuring these triggers ensures that your pipeline responds appropriately to code changes and other relevant actions.

**Events:**
The followings are some important events often used.
- `push:` Trigger the pipeline when code is pushed to a specific branch.
- `pull_request:` Trigger the pipeline when a pull request is opened, updated, or merged.
- `Scheduled Triggers(Cron Jobs):` Run the pipeline at specified intervals (e.g., nightly builds).
- `Manual Triggers:` Allow users to manually trigger the pipeline from the CI/CD tool’s interface.
- `Webhook Triggers:` Trigger the pipeline based on events from `external systems`, such as GitHub, GitLab, or Bitbucket.
  
In this project `Webhook` will be used to trigger the pipeline whenever a PR is opened againt `main` branch.

To achieve this, I will be installing The `GitHub Generic Webhook Plugin` that allows you to trigger Jenkins jobs or pipelines based on events from GitHub webhooks. This plugin is especially useful when you want to create more customized and flexible CI/CD workflows that are not limited to the default triggers provided by other GitHub integration plugins.

#### Setup Webhook

**1. Install the GitHub Generic Webhook Plugin:**

In Jenkins, go to Manage Jenkins -> Manage Plugins -. Available tab -> search for "GitHub Generic Webhook Plugin".

**2. Configure the GitHub Webhook:**

- Go to your `GitHub repository` settings.
- Navigate to Webhooks and click on Add webhook.
- Set the payload URL to your Jenkins URL followed by /generic-webhook-trigger/invoke `(e.g., http://your-jenkins-server/generic-webhook-trigger/invoke?tpken=your-token)`.
- Choose application/json as the content type.
- Select the `event` you want to trigger the webhook (e.g. `pull_request`).
- Click Add webhook

**3. Configure the Webhook Trigger:**

In the Generic Webhook Trigger section in Jenkins UI:
- **Token:** Specify a secure token. This token will be used in the webhook URL to authenticate requests.
- **Post Content Parameters:** Define the parameters you expect from the webhook payload. For a GitHub PR webhook, you might specify parameters like action, pull_request, and other relevant fields.
- Add the following key-value pair to capture the `action`, `source_branch` and `target_branch`:
    - `variable: action, expression: $.action`
    - `variable: source_branch, expression: $.pull_request.head.ref`
    - `variable: target_branch, expression: $.pull_request.base.ref`
    
- **Optional Filter:** Use Groovy expressions or JSONPath to filter events. For example, you can filter events to trigger only when the action is opened for pull requests.

    - `"Text"`: $action $source_branch $target_branch
    - `"Expression"`: ^opened\.*\main$

**Groovy Script** for triggers:

```groovy
triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'action', value: '$.action'],
                [key: 'source_branch', value: '$.pull_request.head.ref'],
                [key: 'target_branch', value: '$.pull_request.base.ref']
            ],
            causeString: 'Triggered by PR from ${source_branch} to ${target_branch}',
            token: 'demo', // Ensure this token matches the one configured in the webhook
            printContributedVariables: true,  // Prints Contributing Variables in console output
            printPostContent: false,  // Prints payload details in console output
            silentResponse: true,
            regexpFilterText: '''${action}\n${source_branch}\n${target_branch}''',
            regexpFilterExpression: '^opened\n.*\nmain$'
        )
    }
```

This ensures that the job is triggered only when a `pull request` is opened against the `main` branch.

### Environment Variables
This section outlines the `environment variables` used in the CI/CD pipeline configuration. These variables are crucial for managing and automating the build, test, and deployment processes.

**Modularity:** Using environment variables makes the pipeline script modular and easier to maintain. Changes to configurations like URLs, credentials, or version numbers can be made in one place.

**Security:** Sensitive information such as credentials should be stored securely and referenced using environment variables.

**Flexibility:** Environment variables allow for easy adjustments and scalability, such as changing repository URLs or deploying to different environments.

```groovy
environment{
    NEXUS_VERSION = "nexus3"
    NEXUS_URL = "http://IP:PORT"
    NEXUS_REPOSITORY = "petclinic-war-repo"
    NEXUS_HELM_REPOSITORY = "petclinic-chart-repo"
    NEXUS_CREDENTIAL_ID = "CREDS-ID"
    DOCKERHUB_USERNAME = "USERNAME"
    APP_NAME = "petclinic"
    IMAGE_NAME = "${DOCKERHUB_USERNAME}/${APP_NAME}"
    IMAGE_TAG = "${BUILD_NUMBER}"
    CHART_DIR = "petclinic-chart"
    CHART_VERSION = "0.${BUILD_NUMBER}.0"
	APP_VERSION = "0.${BUILD_NUMBER}.0"
    RELEASE_NAME = "petclinic"
  }
```

## Pipeline Stages
In software development, especially within CI/CD environments, a pipeline represents an automated series of steps that code changes undergo to be tested, validated, and prepared for deployment. The typical stages of a CI/CD pipeline vary from project to project and organization to organization. I am designing a robust pipeline
that's why it may include multiples stages.

***Agent Template***:
```bash
kubernetes {
    yaml '''
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            app: test
        spec:
          containers:
          - name: <container_name>
            image: <docker_image>
            command:
            - cat
            tty: true
          - name: <container_name>
            image: <docker_image>
            command:
            - cat
            tty: true
    '''
}
```

### Stage 1. Checkout SCM
The Jenkins retrieves the latest code from the repository to the local build environment where the pipeline will run.
- Checkout the `source branch` of the repository.
- Uses a git container to fetch the latest code from the GitHub repository.
- Sends a success status notification upon successful checkout.
- 
**Docker Container:**
```bash
- name: git
  image: bitnami/git:latest
  command:
  - cat
  tty: true
```
**Stage:**
```bash
stages {
    stage('Checkout SCM') {
      steps {
        container('git') {
          script {
            checkout([
                $class: 'GitSCM', 
                //branches: [[name: "*/${source_branch}"]], 
                branches: [[name: '**']],
                extensions: [], 
                userRemoteConfigs: [[url: 'your-repo-url']]
            ])
          }
        }
      }
      post {
        success {
          sendStatus(Checkout SCM","success")
        }
        failure {
          sendStatus(Checkout SCM","failure")
        }
      }
    }
```

### Stage 2. Maven Build 
The code is built using Maven, with Jenkins automating the build process.

**Docker Container:**
```bash
- name: maven
    image: maven:3.8.3-adoptopenjdk-11
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: "/root/.m2/repository"
      name: cache
volumes:
  - name: cache
    persistentVolumeClaim:
      claimName: maven-cache
```
**Caching Maven Dependencies:** By mounting a persistent volume at `/root/.m2/repository`, the Maven dependencies downloaded during the build process are cached. This cache is stored outside the container and persists across multiple builds, significantly reducing the time required for dependency resolution in subsequent builds. Next time Maven Build will not take the longer time since dependencies are alreaady persisted. Create a file with `.yaml` extension and paste below content.
```yaml
# maven-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: maven-cache
spec: 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**Groovy Script:**
```groovy
stage('Maven Build') {
    steps {
        container('maven') {
            sh 'mvn -Dmaven.test.failure.ignore=true clean package'
            }
        }
        post {
            success {
                junit '**/target/surefire-reports/*.xml'
                sendStatus("Maven Build", "success")
            }
            failure {
                sendStatus("Maven Build", "failure")
            }
        }
    }
```

### Stage 3. SonarQube Scan
In this stage, the application code is analyzed using SonarQube. The SonarQube scan helps to maintain code quality by identifying bugs, vulnerabilities, and code smells in the codebase.

**Docker Container:**
```bash
- name: sonarcli
  image: sonarsource/sonar-scanner-cli:latest
  command:
  - cat
  tty: true
```

**Groovy Script:**
```groovy
    stage('Sonar Scan'){
      steps{
        container('sonarcli'){
          withSonarQubeEnv(credentialsId: '<sonar-creds>', installationName: '<server-name>') { 
            sh '''/opt/sonar-scanner/bin/sonar-scanner \
              -Dsonar.projectKey=petclinic \
              -Dsonar.projectName=petclinic \
              -Dsonar.projectVersion=1.0 \
              -Dsonar.sources=src/main \
              -Dsonar.tests=src/test \
              -Dsonar.java.binaries=target/classes  \
              -Dsonar.language=java \
              -Dsonar.sourceEncoding=UTF-8 \
              -Dsonar.java.libraries=target/classes
            '''
          }
        }
      }
      post {
        success {
          sendStatus("Sonar Scan","success")
        }
      }
    }

```

### Stage 4. Quality Gate 
- A `Quality Gate` is a crucial stage that evaluates the overall health and quality of the codebase. It is applied to assess the code against `predefined criteria`.
- The Jenkins job `waits` for the Quality Gate to provide its verdict on the code quality. This waiting period ensures that the evaluation completes before proceeding further with other stages in the pipeline.
- If the Quality Gate criteria are met (indicating good code quality), the pipeline proceeds to the next stages. However, if the Quality Gate criteria are not met, the pipeline may be configured to abort, preventing potentially flawed code from progressing further in the deployment process.

**Docker Container:**
```bash
- name: sonarcli
  image: sonarsource/sonar-scanner-cli:latest
  command:
  - cat
  tty: true
```
**Groovy Script:**
```groovy
stage('Quality Gate'){
      steps{
        container('sonarcli'){
          timeout(time: 1, unit: 'HOURS') {
            waitForQualityGate abortPipeline: true
          }
        }
      }
      post {
        success {
          sendStatus("QG Check","success")
        }
      }
    }
```


### Stage 5. Publish Artifacts to Nexus
Publishing Maven artifacts (such as JAR files) to a centralized Nexus repository ensures consistent and controlled artifact management throughout the CI/CD pipeline.

**Docker Agent:** ll be using default container `jnlp`.

**Groovy Script:**
```groovy
stage('Publish Artifacts to Nexus'){
      steps {
        container('jnlp'){
          script {
            pom = readMavenPom file: "pom.xml";
            filesByGlob = findFiles(glob: "target/*.${pom.packaging}"); 
            echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
            artifactPath = filesByGlob[0].path;
            artifactExists = fileExists artifactPath;
            if(artifactExists) {
                echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                nexusArtifactUploader(
                    nexusVersion: NEXUS_VERSION,
                    protocol: NEXUS_PROTOCOL,
                    nexusUrl: NEXUS_URL,
                    groupId: pom.groupId,
                    version: pom.version,
                    repository: NEXUS_REPOSITORY,
                    credentialsId: NEXUS_CREDENTIAL_ID,
                    artifacts: [
                        [artifactId: pom.artifactId,
                        classifier: '',
                        file: artifactPath,
                        type: pom.packaging],

                        [artifactId: pom.artifactId,
                        classifier: '',
                        file: "pom.xml",
                        type: "pom"]
                    ]
                );

            } else {
                error "*** File: ${artifactPath}, could not be found";
            }
          }
        }
      }
      post {
        success {
          sendStatus("Push to Nexus","success")
        }
        failure {
          sendStatus("Push to Nexus","failure")
        }
      }
    }
```

### Stage 6. Build and Push Docker Image
Building and pushing Docker images is a critical step to ensure that your application can be deployed consistently across different environments. I'm here using `Docker Hub` as regestry. You can use as per your needs like private container registry.

**Docker Container:**
```yaml
- name: docker
  image: docker:latest
  command:
  - cat
  tty: true
  volumeMounts:
  - mountPath: /var/run/docker.sock
    name: docker-sock
volumes:
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
```

**Access to Docker Daemon:** By mounting `/var/run/docker.sock` from the host to the Docker container, this configuration grants the container direct access to the Docker daemon running on the host machine. This allows the container to interact with Docker services and perform actions such as building, pushing, or managing Docker images and containers.

**Groovy Script:**
```groovy
stage('Build and Push Docker Image'){
      steps{
        container('docker'){
          sh "docker build -t $IMAGE_NAME:$IMAGE_TAG ."
          sh "docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest"
          withCredentials([usernamePassword(credentialsId: 'CREDS', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
             sh "docker login -u $USER -p $PASS"
             sh "docker push $IMAGE_NAME:$IMAGE_TAG"
             sh "docker push $IMAGE_NAME:latest"
          }
          sh "docker rmi $IMAGE_NAME:$IMAGE_TAG"
          sh "docker rmi $IMAGE_NAME:latest"
        }
      }
      post {
        success {
          sendStatus("Push to Dockerhub","success")
        }
        failure {
          sendStatus("Push to Dockerhub","failure")
        }
      }
    }

```

### Stage 7. Update Chart Version
you can automatically update the `version` and `appVersion` in your Helm chart's Chart.yaml, commit and push these changes back to your GitHub repository, and then package and push the Helm chart to your Helm repository. This approach ensures that your Kubernetes application is consistently `versioned` and `deployed`.

**Prerequisites**
`Helm Installed:` Ensure Helm is installed and configured on the Jenkins agent.
`Helm Chart:` A Helm chart should be available in your repository.
`Credentials:` If pushing the Helm chart to a repository, ensure credentials are stored securely in Jenkins.

**Docker Container:**
```bash
- name: kubectl_helm_cli
  image: kunchalavikram/kubectl_helm_cli:latest
  command:
  - cat
  tty: true
```

**Groovy Script:**
```groovy
stage('Update Helm Chart Version') {
  steps {
    container('git') {
      script {
        // Configure Git to treat the workspace directory as safe
        sh 'git config --global --add safe.directory "${WORKSPACE}"'

        // Update Chart.yaml with desired version and appVersion
        sh "sed -i 's/^version: .*/version: ${CHART_VERSION}/' ${CHART_DIR}/Chart.yaml"
        sh "sed -i 's/^appVersion: .*/appVersion: ${APP_VERSION}/' ${CHART_DIR}/Chart.yaml"
        sh "cat ${CHART_DIR}/Chart.yaml | grep -i version"  // Optional: Verify the updated versions

        // Configure Git user information
        sh '''
          git config --global user.name "your-name"
          git config --global user.email "your@example.com"
        '''

        // Add and commit the changes
        sh "git add ${CHART_DIR}/Chart.yaml"
        sh 'git commit -m "Update Helm chart version to ${CHART_VERSION} and appVersion to ${APP_VERSION}"'

        // Push the changes to the GitHub repository
        withCredentials([usernamePassword(credentialsId: 'github-creds', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
          sh "git push https://${USER}:${PASS}@github.com/username/repo-name.git main"
        }
      }
    }
  }
}

```

### Stage 8. Package Helm Chart
Helm charts allow you to define, install, and upgrade complex Kubernetes applications using a single command, Packages the Helm chart using the `helm package` command.

**Docker Container:**
```bash
- name: kubectl_helm_cli
  image: asa-96/kubectl_helm_cli:latest
  command:
  - cat
  tty: true
```

**Groovy Script:**
```groovy
stage('Package Helm Chart') {
      steps {
        container('kubectl-helm-cli') {
          // Package Helm chart
          sh "helm package ${CHART_DIR} -d ."
        }
      }
    }
```


### Stage 9. Publish Chart to Nexus
you can automate the process of `packaging` and `publishing` your Helm chart to Nexus, ensuring it's available for `deployment` and distribution within your organization.

**Docker Container:**
```bash
- name: curl
  image: alpine/curl:latest
  command:
  - cat
  tty: true
```

**Groovy Script:**
```groovy
stage('Publish Chart to Nexus') {
      steps {
        container('curl') {
          script {
            def helmChart = findFiles(glob: "${CHART_DIR}-*.tgz")[0]
            withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIAL_ID}", usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')]) {
              sh """
                curl -v --user "${NEXUS_USER}:${NEXUS_PASSWORD}" --upload-file "${helmChart.path}" "${NEXUS_URL}/repository/${NEXUS_REPO}/${helmChart.name}"
              """
            }
          }
        }
      }
    }
```

### Stage 10. Release Helm Chart
you can automate the deployment of your Helm chart from `Nexus` to your `Kubernetes cluster` using Jenkins pipeline, ensuring a smooth CI/CD process

**Docker Container:**
```bash
- name: kubectl_helm_cli
  image: asa-96/kubectl_helm_cli:latest
  command:
  - cat
  tty: true
```

**Groovy Script:**
```groovy
stages {
    stage('Deploy to Kubernetes') {
        steps {
            container('kubectl-helm-cli') {
                withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: "${K8S_CREDENTIALS_ID}", namespace: '', serverUrl: '') {
                    script {
                        sh "helm repo add ${NEXUS_HELM_REPOSITORY} ${NEXUS_URL}"
                        sh "helm repo update"
                        sh "helm upgrade --install ${RELEASE_NAME} ${NEXUS_HELM_REPOSITORY}/${CHART_NAME}"
                    }
                }
            }
        }
    }
```

## Monitoring 

### Monitoring Tools
Details on monitoring tools like Prometheus and Grafana, including configuration steps.
