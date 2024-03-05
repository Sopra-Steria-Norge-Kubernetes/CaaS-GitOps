---
Author: Simen Haga
Title: Continuous Integration and delivery with Tekton Pipelines
Version: 0.1.0
externally-exposed: false
---
## Introduction

This document describes the team's approach to delivering Continuous Integration and Continuous Delivery (CI/CD) workflow using Tekton Pipelines within an OpenShift environment. By integrating these powerful tools, we've established a streamlined workflow that automates the path from source code to production, ensuring consistency, reliability, and efficiency in our deployment processes.

This CI/CD pipeline serves as a template to showcase our streamlined development and deployment capabilities to potential customers. It's a glimpse into how we can enhance their project's efficiency and reliability with our automated solutions.

## Design overview

This is a general overview of the steps involved in the process. 

![](pictures/CI_CD%20-%20Tekton%20(1).png)

This workflow is a high-level representation of a CI/CD process where code changes are automatically fetched, built, and deployed, demonstrating the concept of a pipeline that progresses from code to deployment in a series of automated steps.

The `Pipeline` described in this workflow is our public Sopra Steria/Team Poseidon pipeline, here by called ***public-2S-pipeline***, which is located in a publicly accessible GitHub Repository. The repository defines the different pipelines our customer can choose from to build and deploy their source code. The repository can be found here: [OpenShift Pipelines](https://github.com/TeamPoseidonOCP/openshift-pipelines.git)

1. **Source Code**: The process begins with the source code, which is the starting point for the CI/CD pipeline.
2. **Pipeline Task (Fetch)**: A pipeline task is initiated to fetch the source code upon a new **merge** to the main branch. This is the first operational step in the pipeline where the code is retrieved from the source repository.
3. **Pipeline Task (Build)**: Subsequent to fetching the source code, the next pipeline task is to build the code. The build task compiles the source code and then creates a container image with the application.
4. **Container Registry**: Once the build is successful, the resulting container image is then pushed to a container registry. This registry could be a public or a private store where the image is held for deployment.
5. **Pipeline Task (Fetch from Container Registry)**: Another pipeline task is executed to fetch the **container image tag** from the build-task. This step ensures that the latest built image is retrieved for deployment.
6. **Pipeline Task (Update and Deploy)**: The final pipeline task is to update the deployment with the new container image and then deploy it. This step involves updating the image tag in the deployment configuration and then rolling out the deployment to a live environment.
7. **Deployment**: The deployment is the end state where the new version of the application is running in the production or test environment, ready for use.

## Repository overview

This is a overview and description of the different repositories used in the CI/CD workflow. As the whole workflow is a complex process it is necessary to explain where each component is located and how the different components and therefore repositories interact with each other. 

1. **Public-2S-pipeline-repo**: This repository defines the different pipelines our customer can choose from to build and deploy their source code. The repository contains one pipeline called `build-and-push-python.yml` which build and deploys a python application, before updating the image-tag in the deployment. With time we intend to extend this repository to contain pipelines for multiple programming languages such as java, C# and so on.  
	- https://github.com/TeamPoseidonOCP/openshift-pipelines.git
	  
2. **Tenant-application-repo**: This repository is where the Helm-chart, Helm-values and the resources for the ArgoCD Application is defined. In this repository we define all the resources needed for the Continuous Integration part of the workflow, such as all the Tekton Pipeline resources.
	- https://github.com/TeamPoseidonOCP/openshift-pipelines.git
	  
3. **Deployment-repo** (customer): This repository is where the customer defines the values and enables a CI/CD workflow from Sopra Steria and from these values there will be create an Application in ArgoCD with a deployment that contains the source-code-application image.
	* https://github.com/TeamPoseidonOCP/poseidon2_main_repo.git
	  
4. **Source-code-repo** (customer): This is the source code of the customers application. The customer will have their application in this repository and a merge-request to the main branch in this repository will trigger the CI/CD-process. 
	* https://SolidCloudv2@dev.azure.com/SolidCloudv2/TeamPoseidon/_git/source-to-image
	

> [!NOTE]
> In the parts that follow the repositories mentioned here will be referenced to the names described above. 


## In-depth overview
This section of the documentation aims to describe the CI/CD process in detail, and will provide the reader with a in-depth overview over the workflow. It will ble divided into two main sections; Continuous Integration (*Tekton pipeline part*) and Continuous Delivery.

![](pictures/CI_CD%20-%20Tekton.png)


### Continuous Integration with Tekton Pipelines

**1. Triggering the Pipeline:**

- The process begins with a **merge-to-main** from the source-code-repo, in the chart above called `Customer Application repo`.
- A webhook is set up to listen for code updates and will trigger on this update.
- Once an update is detected, the webhook triggers the Tekton Pipeline.

**2. Tekton Pipeline Execution:**

- The pipeline starts with the `TriggerRoute` which directs the webhook to the appropriate service.
- The `TriggerBinding` translates the webhook event into Tekton resource parameters.
- The `TriggerEventListener` responds to the event by initiating the `TriggerTemplate`.
- The `TriggerTemplate` defines a series of steps and resources that make up a `pipelineRun`.
- The `pipelineRun` is an instance of a `pipelineRef`, which references the predefined pipeline in the Public-2S-pipeline-repo.
- The pipeline then perform the task `fetch-repository`, `build` and `update-image-tag`, which is described in the numbered section here: [[#Design overview]]

**3. Image Creation:**

- After the integration steps are completed, a container image is created with a `image-tag` that is equal to the pull-request number on the merge that was completed in the source-code-repo.
- The image is then pushed to a public or private container registry, ready to be pulled for deployment.

### Continuous Delivery

**1. Deployment Preparation:**

- The new image pushed to the container registry is fetched as part of the deployment process.
- A image-tag is updated in the deployment-repo, which holds the deployment specifications.
- ArgoCD will after 3 minutes try to sync and then update the deployment with the new image-version.

**2. Deployment to OpenShift:**

- The deployment object in OpenShift pulls the new container image and deploys it to the cluster.
- The deployment is monitored, and health checks are performed to ensure successful deployment.
- The running deployment is verified to be using the correct image.

## Ordering a CI/CD pipeline

In this section will go through the process of setting the correct variables and Helm-values in the deployment-repo. This is the process our customer will have to go through when thay want to enable a CI/CD pipeline. First let's look at the `values.yaml` file and go through the variables needed for enabling a CI/CD workflow. 

### Prerequisites
* Have read and deployed a deployment as described in this document:   
	* [Tenant-application deployment](https://dev.azure.com/SolidCloudv2/SolidCloud/_wiki/wikis/SolidCloud.wiki/2350/Tenant-application)
	* Have defined a container limit range in your tenant

>[!NOTE]
>The values set in the example below is just for this documentation purpose and should be replaced with the correct values for your configuration. 

### values.yaml
```yaml
pipeline:
  enable_pipeline: true
  name: testing-pipeline
  container_registry:
    name: 'sourcetoimage.azurecr.io'
    image_name: 'test-image/test-image'
    secret_credentials:
      name: docker-credentials
      encrypted_dockerconfig: <ENCRYPTED_DOCKERCONFIG>
  ci:
    enable_ci: true
    git_application_repo: 
      url: 'https://github.com/application/main_repo.git'
      path_context: 'path/to/application/'
      ## programming_language: should be either python, java or c#
      programing_language: python
      revision: HEAD
      secret:
        custom_secret: false
        name: 'repo-credentials'
        encrypted_username: <ENCRYPTED_USERNAME>
        encrypted_password: <ENCRYPTED_PASSWORD>
  cd:
    enable_cd: true
    git_deployment_repo:
      url: 'https://github.com/deployment/main_repo.git'
      path_value_file: 'path/to/value_file/'
      secret:
        name: 'git-deployment-credentials'
        encrypted_username: <ENCRYPTED_USERNAME>
        encrypted_password: <ENCRYPTED_PASSWORD>
```

### Configuring the `values.yaml` file

After you've set up your tenant application deployment, the next step is to configure the CI/CD pipeline using the `values.yaml` file. This file contains the necessary parameters and settings that define how your CI/CD pipeline will operate. Here's a brief explanation of the variables you need to set:

#### Pipeline Configuration

- `enable_pipeline`: Set this to `true` to activate the pipeline process.
- `name`: Provide a name for your pipeline, such as `testing-pipeline`.

#### Container Registry Settings

- `container_registry.name`: Specify the name of the container registry, like `sourcetoimage.azurecr.io`.
- `container_registry.image_name`: Define the image name, for example, `test-image/test-image`.
- `container_registry.secret_credentials.name`: Enter the name of your docker credentials secret.
- `container_registry.secret_credentials.encrypted_dockerconfig`: Insert your encrypted docker configuration credentials.

#### Continuous Integration Settings

- `ci.enable_ci`: Set to `true` to enable continuous integration.
- `ci.git_application_repo.url`: Provide the URL to your application's Git repository.
- `ci.git_application_repo.path_context`: Indicate the path context to your application within the repository.
- `ci.git_application_repo.programing_language`: Specify the programming language used, choices being `python`, `java`, or `c#`.
- `ci.git_application_repo.revision`: Usually set to `HEAD` to indicate the latest commit in the specified branch.
- `ci.git_application_repo.secret.custom_secret`: Set to `false` if you are not using a custom secret.
- `ci.git_application_repo.secret.name`: Name your repository credentials secret.
- `ci.git_application_repo.secret.encrypted_username`: Your encrypted repository username.
- `ci.git_application_repo.secret.encrypted_password`: Your encrypted repository password.

#### Continuous Delivery Settings

- `cd.enable_cd`: Set to `true` to enable continuous delivery.
- `cd.git_deployment_repo.url`: Provide the URL to your deployment's Git repository.
- `cd.git_deployment_repo.path_value_file`: Specify the path to the value file within the deployment repository.
- `cd.git_deployment_repo.secret.name`: Name your Git deployment credentials secret.
- `cd.git_deployment_repo.secret.encrypted_username`: Your encrypted Git username for the deployment repository.
- `cd.git_deployment_repo.secret.encrypted_password`: Your encrypted Git password for the deployment repository.

Ensure that you replace the placeholder values with the actual data relevant to your environment. These configurations will direct the CI/CD pipeline on how to access your repositories, build and push images, and deploy your applications. Once set, your pipeline will be ready to run and manage the lifecycle of your applications automatically.
