In this project, we will implement continuous integration (CI) and Continous Delivery (CD) for a multi-microservice application provided by the development team. The application consists of:

- **Microservices**:
  - One written in Python (Voting Microservice)
  - One written in Node.js (Results Microservice)
  - One written in .NET (Worker Microservice)

- **Databases**:
  - An in-memory data store (Redis)
  - A PostgreSQL database

The source code is available in a [GitHub repository](https://github.com/dockersamples/example-voting-app.git) from Docker samples. Our task as the DevOps engineer is to set up the CI system using Azure DevOps and Azure Pipelines.

To accomplish this, we will:
1. Clone the GitHub repository to run the application locally and understand its architecture.
2. Write Docker files and configure the CI system for the different microservices, which use various programming languages.

**Application Overview**:
- **Voting Microservice**: Allows users to vote between two options (e.g., cats vs. dogs).
- **Results Microservice**: Displays the voting results.
- **Worker Microservice**: Manages data transfer between the in-memory data store (Redis) and the PostgreSQL database.

Each microservice can be independently deployed. We will create separate CI pipelines for each:
- Voting Microservice (Python)
- Worker Microservice (.NET)
- Results Microservice (Node.js)

This approach allows independent updates and deployments without affecting other services.

We will begin by running the application locally using `docker compose` to understand its functionality and then proceed to implement CI for each microservice. This practical exercise will enhance our understanding of continuous integration and delivery.

**CI/CD Overview**:
- **CI/CD** stands for Continuous Integration and Continuous Delivery (not Deployment). For example, if a developer makes a change in the `server.js` file of the "results" microservice, the CI process ensures the change is validated before merging.
- **CI (Continous Integration)** : Automates the integration of code changes from multiple developers into a shared repository. For example, when a developer pushes code to a shared repository, CI tools automatically run tests and checks to ensure the new code integrates well with the existing codebase.
- **CD (Continuous Delivery)** : Automates the process of delivering code changes to production, but doesn't necessarily deploy them immediately. For example, it ensures that code is always in a deployable state, performing automated testing, and creating production-ready builds so that deployment can be done manually when needed.

 
- DevOps engineers set up CI to automatically:
  - Run unit tests and static code analysis on changes.
  - Build the application and run end-to-end tests.
  - Create a Docker image of the application.
  - Push the Docker image to a registry like Docker Hub or Azure Container Registry.

This automation saves time and provides confidence that changes won’t break the application. Manual testing would be time-consuming and prone to errors.

# Continous Integration

- We will create CI pipelines using Azure Pipelines.
- CI can be triggered by pull requests or commits.

## Creating Dockerfiles for Microservices
Let's start by understanding how to create Dockerfiles for our three microservices. This is crucial since each stage—build, test, and image creation—depends on it. In modern containerized applications, the build is part of the Docker image creation. We’ll explore how to write Dockerfiles for each microservice by examining the repository structure and the dependencies.

#### Writing Dockerfiles for Microservices

1. **Results Microservice**:
   - Navigate to the repository and observe the presence of `server.js` and `package.json`, indicating it's a Node.js application.
   - To containerize this application:
     1. **Base Image**: Use a Node.js base image (`node`) because the application is written in Node.js. This eliminates the need to install Node.js manually.
     2. **Working Directory**: Set up a working directory in the Dockerfile.
     3. **Copy Dependencies**: Copy `package.json` and run `npm install` to install dependencies.
     4. **Build and Run**: Copy the application code, set up the entry point using `CMD` or `ENTRYPOINT`, and expose necessary ports.

    Here’s a simplified Dockerfile for the results microservice:
    
      ```dockerfile
      FROM node:18-slim

      # add curl for healthcheck
      RUN apt-get update && \
          apt-get install -y --no-install-recommends curl tini && \
          rm -rf /var/lib/apt/lists/*

      WORKDIR /usr/local/app

      # have nodemon available for local dev use (file watching)
      RUN npm install -g nodemon

      COPY package*.json ./

      RUN npm ci && \
      npm cache clean --force && \
      mv /usr/local/app/node_modules /node_modules

      COPY . .

      ENV PORT 80
      EXPOSE 80

      ENTRYPOINT ["/usr/bin/tini", "--"]
      CMD ["node", "server.js"]
      ```


2. **Voting Microservice**:
   - Identify it as a Python application by files like `app.py` and `requirements.txt`.
   - Dockerfile steps will be similar to Node.js, with adaptations for Python:
     1. **Base Image**: Use a Python base image (`python`).
     2. **Working Directory**: Define a working directory.
     3. **Copy Dependencies**: Copy `requirements.txt` and run `pip install`.
     4. **Build and Run**: Copy the application code, set up the entry point using `CMD` or `ENTRYPOINT`, and expose necessary ports.

    Example Dockerfile for the voting microservice:
      ```Dockerfile
      # Define a base stage that uses the official python runtime base image
      FROM python:3.11-slim AS base

      # Add curl for healthcheck
      RUN apt-get update && \
          apt-get install -y --no-install-recommends curl && \
          rm -rf /var/lib/apt/lists/*

      # Set the application directory
      WORKDIR /usr/local/app

      # Install our requirements.txt
      COPY requirements.txt ./requirements.txt
      RUN pip install --no-cache-dir -r requirements.txt

      # Define a stage specifically for development, where it'll watch for
      # filesystem changes
      FROM base AS dev
      RUN pip install watchdog
      ENV FLASK_ENV=development
      CMD ["python", "app.py"]

      # Define the final stage that will bundle the application for production
      FROM base AS final

      # Copy our code from the current folder to the working directory inside the container
      COPY . .

      # Make port 80 available for links and/or publish
      EXPOSE 80

      # Define our command to be run when launching the container
      CMD ["gunicorn", "app:app", "-b", "0.0.0.0:80", "--log-file", "-", "--access-logfile", "-", "--workers", "4", "--keep-alive", "0"]
      ```

3. **Worker Microservice**:
   - Recognize it as a .NET application.
   - Use a .NET base image and follow a similar process:
     1. **Base Image**: Use a .NET SDK image.
     2. **Working Directory**: Set up a working directory.
     3. **Copy Dependencies**: Copy project files and restore dependencies.
     4. **Build and Run**: Build the application, set the entry point, and expose necessary ports.

    Simplified Dockerfile for the worker microservice:
    ```dockerfile
      FROM --platform=linux mcr.microsoft.com/dotnet/sdk:7.0 as build
      ARG TARGETPLATFORM
      ARG TARGETARCH
      ARG BUILDPLATFORM
      RUN echo "I am running on $BUILDPLATFORM, building for $TARGETPLATFORM"

      WORKDIR /source
      COPY *.csproj .
      RUN dotnet restore

      COPY . .
      RUN dotnet publish -c release -o /app --self-contained false --no-restore

      # app image
      FROM mcr.microsoft.com/dotnet/runtime:7.0
      WORKDIR /app
      COPY --from=build /app .
      ENTRYPOINT ["dotnet", "Worker.dll"]
    ```

### Transition to Azure DevOps

- We will use Azure DevOps to build CI pipelines for these Dockerfiles.
- Make sure your Azure DevOps and Microsoft Azure subscriptions are set up, as we’ll use Azure Container Registry (ACR) and Azure Pipelines.
- The Docker images will be pushed to ACR, but Docker Hub can be used as well if preferred.

1. **Access Azure DevOps**:
   - Log in and create an account if you haven’t already.

2. **Project Setup**:
   - Create a new project for your CI pipelines.
   - We will configure Azure Pipelines to build, test, and push Docker images for each microservice to ACR.

This approach provides a seamless integration and deployment process, ensuring each microservice is properly containerized and managed through Azure DevOps.

Let's start from scratch to create and set up our voting application in Azure DevOps. I will guide you through setting up a project, importing a repository, and creating CI pipelines to build Docker images for our microservices.

### Setting Up Your Azure DevOps Project

1. **Create a New Project**:
   - Navigate to Azure DevOps and click on "New Project."
   - Name your project "Voting Application" or something relevant. 
   - Choose to keep it private if you’re working in an organizational setting.
   - Click "Create Project."

2. **Use Azure Repos**:
   - You can use Azure Repos instead of GitHub to keep everything within Azure DevOps.
   - Go to the **Repos** section and select "Import a Repository."
   - Choose "Git" for the import type.
   - Enter the URL of your GitHub repository and click "Import."
   - Ensure that no authentication is required if the repository is public.

3. **Set the Default Branch**:
   - By default, Azure DevOps might set a branch alphabetically. Make sure the default branch is `main`.
   - Go to the **Branches** section, find the `main` branch, and set it as the default by clicking the three dots next to the branch name and selecting "Set as default branch."

### Setting Up Azure Container Registry (ACR)

1. **Create a Resource Group**:
   - Go to the Azure portal and search for "Resource Groups."
   - Click "Create" and name it something like "AzureCICD."
   - Select a region (e.g., East US) and click "Review + Create."

2. **Create a Container Registry**:
   - In the Azure portal, search for "Container Registry" and click "Create."
   - Select the resource group you just created.
   - Name the container registry something like "AzureCICD."
   - Choose the "Standard" plan (or "Basic" if you prefer).
   - Click "Review + Create."

    > **Note**:If you are using Azure free trial, you'll need to create a VM in azure portal to run the pipelines. This VM will act as the pool (agent) for the pipeline.

3. **Setting Up the Agent (Create a Virtual Machine (VM) for Azure Pipelines)**:
   - In Azure portal, search for "Virtual Machine" and click "Create"
   - Select the resource group you just created.
   - Name the VM "azureagent"
   - Make sure to select SSH as authenticaation and Allow SSH access.
   - Click "Review + Create."

      > SSH to VM and Install following:
        > Docker: 
        > ```bash
        > sudo apt install docker.io
        > sudo usermod -aG docker $USER
        > sudo systemctl start docker
        > sudo systemctl enable docker
        >```
        > Configure to connect Vm to Azure DevOps
        > a. Go to Azure Devops Project Settings
        > b. Click Agent Pools and Select 'azureagent'
        > c. Select 'Agents' and click 'New Agent'
        > d. Follow the Azure DevOps documentation to configure the agent and link it to your Azure DevOps project.

### Creating CI Pipelines

1. **Create a Pipeline**:
   - Go back to Azure DevOps and click on the **Pipelines** section.
   - Click "New Pipeline" and select "Azure Repos Git" since we’re using Azure Repos.
   - Select your repository (e.g., "Voting Application").
   - Choose "Docker" as the template.

2. **Configure the Pipeline**:
   - Select the template "Build and Push Docker Image to Azure Container Registry."
   - Choose the Azure Container Registry you set up earlier.
   - Click "Validate and configure."

3. **Pipeline YAML Configuration**:
   - The template will generate a basic `azure-pipelines.yml` file for building and pushing Docker images.
   - Understand the main components of the pipeline:
     - **Trigger**: Specifies which branches trigger the pipeline.
     - **Resources**: Defines external resources (like repositories).
     - **Variables**: Defines variables used in the pipeline.
     - **Stages**: Contains jobs and steps to execute.

      Here’s a simple `azure-pipelines.yml` example for building and pushing a Docker image:

      ```yaml
        # Docker
        # Build and push an image to Azure Container Registry
        # https://docs.microsoft.com/azure/devops/pipelines/languages/docker

        trigger:
          paths:
            include:
              - result/*

        resources:
        - repo: self

        variables:
          # Container registry service connection established during pipeline creation
          dockerRegistryServiceConnection: 'xxx'
          imageRepository: 'resultapp'
          containerRegistry: 'chetanazurecicd.azurecr.io'
          dockerfilePath: '$(Build.SourcesDirectory)/result/Dockerfile'
          tag: '$(Build.BuildId)'

          # Agent VM image name
        pool:
          name: 'azureagent' 

        stages:
        - stage: Build
          displayName: Build
          jobs:
          - job: Build
            displayName: Build
            steps:
            - task: Docker@2
              displayName: Build Image
              inputs:
                containerRegistry: '$(dockerRegistryServiceConnection)'
                repository: '$(imageRepository)'
                command: 'build'
                Dockerfile: 'result/Dockerfile'
                tags: '$(tag)'
        - stage: Push
          displayName: Push
          jobs:
          - job: Push
            displayName: Push
            steps:
            - task: Docker@2
              displayName: Push an image to container registry
              inputs:
                containerRegistry: '$(dockerRegistryServiceConnection)'
                repository: '$(imageRepository)'
                command: 'push'
                tags: '$(tag)'

      ```

   - Replace `'<service-connection-id>'` with your actual service connection ID.
   - Since free trial doesn't support agent so we are using VM.
   - Replace trigger with following:
     ```
      - paths:
          include:
            - result/* 
      ```
    - Replace Agent VM image name with:
       ``` 
        pool:
        name: 'azureagent'
        ```

4. **Save and Run**:
   - Save the pipeline configuration and run it.
   - Monitor the pipeline to ensure it builds the Docker image and pushes it to the Azure Container Registry.

5. **Create Pipelines for Other Microservices**:
   - Repeat the pipeline creation process for each microservice (e.g., `voting`, `results`, `worker`).
   - Ensure each pipeline is configured to build and push Docker images for the respective microservice.

## Notes

- The CI pipeline setup will automate building Docker images and pushing them to ACR whenever changes are made.
- You can view pipeline runs in the Azure DevOps portal to monitor progress and diagnose issues.

This setup will streamline your CI process, making it easier to manage and deploy your microservices using Azure DevOps and Azure Container Registry.

To set up an Azure CI pipeline, follow these four key components: triggers, stages, jobs, and steps. Here’s a breakdown of each component and how to configure them.

#### Triggers
Triggers determine when a pipeline should be automatically triggered. For instance, you can set triggers based on changes to specific paths in a repository. This eliminates the need for manual triggering. An advanced example involves using path-based triggers to initiate pipelines for specific microservices within a project.

#### Stages
Stages represent different phases in the CI/CD pipeline, such as build, test, and deployment. Each stage can contain multiple jobs, allowing for parallel execution and efficient resource utilization.

#### Jobs
Jobs are smaller units of work within a stage. They can be assigned to different agents or runners to distribute the load. This is useful for complex builds that can be split into independent tasks.

#### Steps
Steps are individual tasks within a job, such as running a script or building a Docker image. Variables can be used to pass configuration details, credentials, or other necessary data between steps.

### Example CI Pipeline Configuration

1. **Triggers:**
   Use path-based triggers to ensure that only relevant pipelines are triggered based on changes in specific paths.
   ```yaml
   trigger:
     paths:
       include:
         - 'result/**'
   ```

2. **Variables:**
   Define variables for Docker registry credentials and other necessary configurations.
   ```yaml
   variables:
     DOCKER_REGISTRY_SERVICE_CONNECTION: $(dockerRegistryConnection)
     IMAGE_REPOSITORY: 'resultapp'
   ```

3. **Stages, Jobs, and Steps:**
   Define build and push stages, with respective jobs and steps to build and push Docker images.
   ```yaml
    stages:
    - stage: Build
      displayName: Build
      jobs:
      - job: Build
        displayName: Build
        steps:
        - task: Docker@2
          displayName: Build Image
          inputs:
            containerRegistry: '$(dockerRegistryServiceConnection)'
            repository: '$(imageRepository)'
            command: 'build'
            Dockerfile: 'result/Dockerfile'
            tags: '$(tag)'
    - stage: Push
      displayName: Push
      jobs:
      - job: Push
        displayName: Push
        steps:
        - task: Docker@2
          displayName: Push an image to container registry
          inputs:
            containerRegistry: '$(dockerRegistryServiceConnection)'
            repository: '$(imageRepository)'
            command: 'push'
            tags: '$(tag)'
   ```

### Testing the Pipeline
1. **Commit Changes:**
   Make a change in the specified path (e.g., `result/server.js`).
2. **Verify:**
   Ensure the pipeline triggers automatically and runs the defined stages successfully.

By structuring your pipeline with these components, you can efficiently manage CI/CD processes, ensuring that only relevant parts of your application are built and deployed based on specific triggers.

# CD Pipeline
Sure, let's dive into setting up the continuous delivery (CD) part for your Azure DevOps project using the GitOps approach. Here’s how you can configure it and explain the setup:

### Setting Up Continuous Delivery with GitOps

No we will set up continuous delivery (CD) using the GitOps approach, which leverages Git repositories as the source of truth for declarative infrastructure and application configurations.

#### GitOps Overview

GitOps is a modern approach to continuous delivery that uses Git repositories for managing infrastructure and application configuration. It is a practice where Git repositories are used as the single source of truth for infrastructure and application configurations. The key components typically include:

- **Git Repository**: Stores all configuration files (like Kubernetes YAML manifests).
- **Continuous Deployment Tool (Argo CD)**: Watches Git repositories for changes and automatically applies them to the Kubernetes cluster.

### Understanding Continuous Delivery (CD) Workflow

#### Overview of CD with GitOps Approach

In Continuous Integration (CI), where we built Docker images and pushed them to Azure Container Registry (ACR). Now we will focus on Continuous Delivery (CD), which involves deploying these newly created container images to our Kubernetes cluster.

#### CD Workflow Components

1. **Developer Action**:
   - A developer makes changes to the source code stored in Azure Repos (e.g., modifying voting options in `app.py`).
   - Commits and pushes the changes to Azure Repos.

2. **Azure Pipelines (CI/CD)**:
   - **Build Stage**: Automatically triggered upon a new commit.
     - Builds the Docker image for the updated application.
   - **Push Stage**: Pushes the Docker image to Azure Container Registry (ACR).

3. **Update Stage (New Addition)**:
   - **Purpose**: To update Kubernetes manifests (`deployment.yaml`) in the Azure Repos with the newly created Docker image tag.
   - **Script**: A shell script (`updateKmanifest.sh`) is introduced to automate the update process.
     - The script finds and updates the image tag in the appropriate Kubernetes manifest file (`deployment.yaml`).

4. **GitOps with Argo CD**:
   - **Purpose**: Monitors the Azure Repos for changes in Kubernetes manifests.
   - **Deployment**: Automatically deploys updated manifests to the AKS cluster.
   - **Architecture Diagram**: A clear diagram is prepared and uploaded to the designated folder (`day15`), illustrating the GitOps workflow.

#### Detailed Workflow

- **CI Phase**: Handles the build and push stages to ACR.
  - Developer commits changes → Build triggers → Docker image created → Push to ACR.
  
- **Update Stage**: Automates the update of Kubernetes manifests in Azure Repos.
  - Shell script updates `deployment.yaml` with the new Docker image tag.
  
- **GitOps (Argo CD)**:
  - Listens for changes in Azure Repos, specifically in Kubernetes manifests.
  - Detects updates made by the update stage and deploys them to AKS.
  
- **End-to-End CI/CD Pipeline**:
  - Azure Repos acts as a central repository.
  - CI handles image creation and ACR push.
  - Update stage bridges CI with CD by updating Kubernetes manifests.
  - GitOps (Argo CD) ensures automatic deployment of changes to the AKS cluster based on updated manifests.

Let's proceed with setting up the Azure Kubernetes Service (AKS) and configuring Argo CD for GitOps.


### Setting Up Kubernetes Service

#### Creating Azure Kubernetes Service (AKS)

1. **Creating Kubernetes Cluster**:
   - Navigate to Azure Portal and click on "Create a resource".
   - Search for "Kubernetes Service (AKS)" and select it.
   - Click on "Create" to start configuring the AKS.

2. **Configuration Details**:
   - **Basics**:
     - Choose a subscription and resource group (`azurecicd`).
     - Specify a cluster name (`azure-devops-aks`).
     - Select a region (e.g., `West US 2` for this demo).
     - Choose Kubernetes version and networking configuration.

3. **Node Pools**:
   - **Agent Pool**:
     - Configure the node pool settings:
       - Set minimum and maximum node counts (`1` to `2` for autoscaling).
       - Specify other details like node size, disk type, etc.

4. **Scaling Options**:
   - Enable autoscaling for the node pool to handle varying workload demands automatically.

5. **Networking**:
   - Configure network settings and optionally enable public IP per node for easy service access.

6. **Review + Create**:
   - Review the configuration details.
   - Click on "Create" to start provisioning the AKS cluster.
   - Azure will validate the configuration and deploy the cluster. This process may take several minutes.



### Configuring Argo CD for GitOps

#### Overview

Now that we have our AKS cluster ready, we'll proceed with setting up Argo CD to manage deployments using GitOps principles.

1. **Accessing Azure Kubernetes Cluster**:
   - Use Azure CLI to configure kubectl to connect to your AKS cluster:
     ```bash
     az aks get-credentials --resource-group az-devops --name azuredevops --overwrite-existing
     ```

2. **Installing Argo CD**:
   - Follow the official Argo CD documentation for installation. Here’s a summarized approach:
     - Visit the [Argo CD installation guide](https://argoproj.github.io/argo-cd/getting_started/#1-install-argo-cd).
     - Execute the installation commands:
       ```bash
       kubectl create namespace argocd
       kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
       ```

3. **Accessing Argo CD Dashboard**:
   - Once installed, get Argocd initial password for login:
      ```bash
      kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}'
      ```
   - Now decode the output of above command:
      ```bash
        echo -n $PASSWORD | base64 -d
      ```
   - Now to access UI of Argocd, 1st change the type of Argocd Server from CLusterIP to Nodeport:
      ```bash
      kubectl edit svc/argocd-server -n argocd --type=NodePort
      ```
   - Copy the NodeIP of cluster and HTTP NodePort:
      ```bash
      kubectl get nodes -o wide
      kubectl get svc -n argocd
      ```
   - Open the port in Network Security group of AKS cluster:
     - Go to Azure portal and search for "Virtual machine scale sets" and click of existing Node pool.
     - Go to Instance -> Networking -> Inbound Traffic and add destination port as copied above. 
   - Access the Argo CD UI in your browser at `https://NodeIp:NodePort/argocd` (accept any security warnings). (Username: admin, Password: decoded above)

4. **Configuring Argo CD with Azure DevOps Git Repository**:
   - **Access Token Creation**:
     - Generate a personal access token (PAT) in Azure DevOps with read permissions for repository access.
   
   - **Connecting Repository**:
     - Go to Argo CD UI → `Settings` → `Repositories`.
     - Add a new repository:
       - Connection method: `via HTTPS`
       - Repository URL: Paste your Azure DevOps Git repository HTTPS URL and replace the User/Org Name with access token. (Example XYZ with PAT)
         ```bash
         https://XYZ@dev.azure.com/XYZ/ArgocdDeployment/_git/ArgocdDeployment
         https://PAT@dev.azure.com/XYZ/ArgocdDeployment/_git/ArgocdDeployment
         ```
5. **Install Argo CD on your Kubernetes cluster**
   - Create an application in Argo CD for your voting application.
   - Specify the Git repository URL and path to the Kubernetes manifests (e.g., `/k8-specification`).
   - 
6. **Syncing and Deploying Applications**:
   - Argo CD watches for changes in the Git repository (e.g., when a new Docker image is pushed to ACR).
   - Automatically applies these changes to the Kubernetes cluster, ensuring the latest version of the application is deployed.

### Creating a New Stage for Updating and Writing Shell Script in Pipelines

Now that we have the Argo CD setup, we’ll create a new stage in our CI/CD pipeline to automatically update the Kubernetes manifest with the latest image from the Azure Container Registry (ACR). We will write a shell script that identifies the new image and updates the Kubernetes YAML file accordingly.

**Steps to Create the New Update Stage and Shell Script**:

1. **Create New Stage in Pipeline**:
   - Add a new stage named `update` in your Azure Pipeline configuration.
   - Set the `displayName` to `Update`.

2. **Write the Update Shell Script**:
   - The script will fetch the latest image from ACR and update the corresponding tag in the Kubernetes deployment YAML file.

3. **Shell Script Details**:
   - **Identify Image Pattern**: The repository and registry names are constant (`azure-cicd` and `votingapp`). The only changing part is the image tag, which corresponds to the build number.
   - **Shell Script Tasks**:
     - Fetch the latest build ID from the Azure Pipeline.
     - Clone the Git repository where the deployment files are stored.
     - Update the image tag in the Kubernetes deployment YAML file.

      ```bash
         #!/bin/bash

         set -x

         # Set the repository URL
         REPO_URL="https://<ACCESS-TOKEN>@dev.azure.com/<AZURE-DEVOPS-ORG-NAME>/voting-app/_git/voting-app"

         # Clone the git repository into the /tmp directory
         git clone "$REPO_URL" /tmp/temp_repo

         # Navigate into the cloned repository directory
         cd /tmp/temp_repo

         # Make changes to the Kubernetes manifest file(s)
         sed -i "s|image:.*|image: <ACR-REGISTRY-NAME>/$2:$3|g" k8s-specifications/$1-deployment.yaml

         # Add the modified files
         git add .

         # Commit the changes
         git commit -m "Update Kubernetes manifest"

         # Push the changes back to the repository
         git push

         # Cleanup: remove the temporary directory
         rm -rf /tmp/temp_repo
      ```

4. **Add the Script to the Azure Repository**:
   - Navigate to your Azure DevOps repository.
   - Create a new folder named `scripts`.
   - Add the shell script `update_k8s_manifest.sh` to the `scripts` folder.

5. **Configure the New Update Stage in Azure Pipeline**:
   - In your pipeline YAML, add a task to run the `update_k8s_manifest.sh` script.

      ```yaml
      - stage: Update
      displayName: Update
      jobs:
      - job: Update
         displayName: Update
         steps:
         - task: ShellScript@2
            inputs:
            scriptPath: 'scripts/updateK8sManifests.sh'
            args: 'vote $(imageRepository) $(tag)'
      ```

6. **Run the Pipeline**:
   - Save and run your pipeline. It should trigger the `update` stage, which will:
     - Clone the repository.
     - Update the image tag in the deployment YAML.
     - Commit and push the changes.

This setup ensures that every new image build updates the Kubernetes deployment automatically with the latest image, maintaining continuous delivery without manual intervention.

## Demo: Changing the Voting Options

Let's demonstrate how a developer can make a change (e.g., modifying the voting options) and trigger the end-to-end CI/CD process:

1. **Developer Action**:
   - Navigate to the voting application code (e.g., `voting-app`).
   - Edit `app.py` to modify the voting options (e.g., from cats vs dogs to summer vs winter).
   - Commit and push the changes to the Azure DevOps repository.

   ```python
   # Before
   options = ['cats', 'dogs']

   # After
   options = ['summer', 'winter']
   ```

2. **CI/CD Pipeline Triggers**:
   - The commit triggers the CI pipeline defined earlier, which builds the Docker image and pushes it to ACR.

3. **CI Pipeline (Azure DevOps)**:
   - **Build Stage**: Builds the Docker image.
   - **Push Stage**: Pushes the Docker image to ACR.
   - **Update Stage**: Updates the Kubernetes deployment with the latest image.

4. **CD Pipeline (GitOps Approach)**:
   - Uses Argo CD to manage deployments based on Git repository changes.
   - Monitors the Git repository for changes to Kubernetes YAML manifests (deployment, service, etc.).

#### Monitoring Deployment

- **Argo CD UI**: Monitor the Argo CD dashboard to see deployments and synchronization status.
- **Kubernetes Dashboard**: Optionally, use the Kubernetes dashboard to view pod status, service endpoints, etc.

#### Final Verification

- Once Argo CD synchronizes the deployment, verify the changes in your voting application.
- Access the application URL and confirm that the new voting options (summer vs winter) are visible and functional.


