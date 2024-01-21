# CaaS-GitOps
## Overview
Welcome to the CaaS-GitOps repository which gives you an introduction to use OpenShift GitOps with your tenant concept. The repository is designed to showcase the use of ArgoCD to syncronise your tenant applications and infrastructures with a Git repository, ensuring Continuous Deployment (CD). Developers can define application and environment configurations in this Git repository, and ArgoCD will ensure that the OpenShift cluster matches the defined state, deploying and updating applications automatically.

## Introduction
In this repository we will create a example tenant to showcase the different option for enabling a GitOps for your tenant. 

> Before you start, make sure you have a basic understanding of GitOps, ArgoCD, and OpenShift. This guide assumes familiarity with these technologies.

CaaS provides two primary methods for deploying an synchronising infrastructures:
1) **Auto-defined ArgoCD applications:** A ArgoCD applicationSet automatically creates applications when resources are defined in the target path.
   * Target path: `<basepath>applications/<environments[].name>/*`
2) **User-defined ArgoCD applications:** The user creates its own ArgoCD application definitions. Only recommended to use if more functionality needed than what is provided in the Auto-defined method. The ArgoCD application definition must be defined in the correct target path.
   * Target path: `<basepath>applicationsets/<environments[].name>/*`

## Pre-requisites
Before you can utilise the GitOps capabilties on OpenShift you need to setup your OpenShift Tenant correctly. In this repository, we will use a simple example tenant to enable the desired capabilties for both methods. the tenant definition is shown below. 
### Setting up a Tenant
Before we can start we must first define a Tenant and send it to our OCP administrators. Below is a minimal definition for setting up a tenant with two environments, `dev` and `test`, and enabling both GitOps methods above:
```yaml
appname: poseidon1
values: |
  namespace:
    name: poseidon1
    description: This is a test tenant named poseidon1 for showcasing the GitOps capabilities. 
    displayName: poseidon1, testing GitOps
    limits:
      memory: 2Gi
      cpu: 1

  environments:
    - name: dev
    - name: test

  rbac:
    ad_group_write_access: AZURE-AD-GROUP-WRITE-NAME
    ad_group_read_access: AZURE-AD-GROUP-READ-NAME

  argocd: 
    enable_auto_defined_apps: true
    enable_user_defined_apps: true
    main_git_repository:
      repourl: https://github.com/TeamPoseidonOCP/poseidon1_main_repo
      basepath: ""
      encrypted_url: <Encrypted repourl with sealedsecrets>
      encrypted_type: <value git encrypted>
      encrypted_password: <PAT always encrypted with sealedsecrets>
      encrypted_username: <Username used with PAT encrypted with sealedsecrets>
```
Setting the variable `enable_auto_defined_apps` to true gives you the option to use the **Auto-defined ArgoCD applications** method for creating applications. Similary, setting the `enable_user_defined_apps`to true gives you the option to use the **User-defined ArgoCD applications** method for creating applications. 

> More information on how to connect ArgoCD to your repository can be found [here](doc/conecting_to_a_repo.md).

# Getting started
With your tenant configuration successfully completed, the next crucial step is establishing the structure of your main Git repository. 

This section will illustrate both the **auto-defined** and **user-defined** methods for managing applications with ArgoCD. We will use the `poseidon1_main_repository` folder as our example to walk you through the process. 

Consider this folder as your repository blueprint, with the basepath set to an empty string, providing a clear and applicable framework for your GitOps setup. Let´s get started! 

## Auto-defined ArgoCD applications
This approach is the recommended approach for working with ArgoCD. It gives an intuative and easy way of working with multiple ArgoCD applications without implementing a nameingstandard across tenants.

### How it works in the background
The ArgoCD applicationsets looks in the target path `poseidon1_main_repo/applicationsets/<environment_name>`, so every folder created in this path will create a application in ArgoCD. The name of the application will have the following name  `<tenant_name>-<environment_name>-apps-<folder_name>`. For instance if we create a folder called `applicationsets/dev/demo-kubernetes` it will be called `poseidon1-dev-apps-demo-kubernetes` in ArgoCD. 

In addition to the environments a base folder is created in the repository to easier share definitions across environments. The structure of the applicationsets will look like this: 

```bash
├── applicationsets
│   ├── base
│   ├── dev
│   └── test
```

### Deployment Examples
Applications can be deployed with kustomization files, helm templates, helm repositories or plain kubernetes YAML definitions. The examples below show different methods for deploying resources to a cluster:

#### 1) Using YAML definitions of Kubernetes resources
Kubernetes resources can be deployed to the cluster by simpling adding files with YAML definition of Kubernetes resources. Folder `applicationsets/dev/ex1-kubernetes-resources` illustrates an example of how this is done for a service resource. 

#### 2) Using YAML definitions of Kubernetes resources



## User-defined ArgoCD applications



