# CaaS-GitOps

## Overview
Welcome to the CaaS-GitOps repository which gives an introduction to use OpenShift GitOps with the tenant concept. The repository is designed to showcase the use of ArgoCD to syncronise your tenant applications and infrastructures with a Git repository, ensuring Continuous Deployment (CD). Developers can define application and environment configurations in this Git repository, and ArgoCD will ensure that the OpenShift cluster matches the defined state, deploying and updating applications automatically.

A developer must configure in their tenant definition to use `user-defined` and/or `auto-defined` application-creation. When this is set the developer must set up their repository to ligne with their choice of GitOps method. Below is a picture to illustra the process on a high-level:
![Alt text](doc/pictures/GitOps.png)

### GitOps process
The GitOps process follows these steps:
1. Define the tenant defintion with the wanted GitOps method. Push the configuration to your OCP Admin. Follow the setup given in the section [Setting up a tenant](#setting-up-a-tenant).
2. When the changes is merged by the OCP admin the changes will be implemented in the OpenShift cluster.
3. Define Kubernetes resources in the Tenants Main Git repository. Follow the examples given in the section [Getting Started](#getting-started).
4. The ArgoCD resources will then sync resources from the Tenants Main Git repository. It is important to define the correct target path in the repository. 
5. When the resources are synced from the main Tenant repository the resources will be created in the cluster with ArgoCD. 


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

#### 2) Kustomization file
A Kustomization file can be used to define multiple Kubernetes resoucres. It will only deploy the Kubernetes resources listed in the kustomization file.

Folder `applicationsets/dev/ex2-kustomize` illustrates an example of how to use a Kustomization file. In addition, this example show how to use the base folder to add the same deployment to both the `dev` and `test` environment. 

#### 3) Helm 
Helm is another effective tool for deploying and managing Kubernetes applications. It simplifies the process of defining, installing, and upgrading even the most complex Kubernetes applications. Here we will list different examples for deploying helm:

1) Local Helm chart: If you want to define a helm chart locally folder `applicationsets/dev/ex3-helm-1` gives an example for how this can be done. Here a simple service and serviceaccount is defined under the folder `templates/` and the application is created by reading the `values.yaml` and `Chart.yaml` file.
   
2) External Helm repository: If you want to use an external Helm repository folder `applicationsets/dev/ex3-helm-2` gives an example of how this can be done through a kustomization file. Kustomize uses a helmCharts field which has the ability to use the helm command line program in a subprocess to inflate a helm chart, generating YAML as part of (or as the entirety of) a kustomize base. 
   * This only works for public repositories. If you want to use a private repository it is recommended to create an ArgoCD application with the [user-defined method](#user-defined-argocd-applications). 
   
## User-defined ArgoCD applications

The user-defined method for creating ArgoCD applications is only recommended if the auto-defined methos don´t meet your applications requirements. The reason for this is that this method has some limitiations:
* Developers need to have more in-depth knowledge of ArgoCD
* It can be difficult to mange many ArgoCD applications in one Tenant
* A naming standard is required since the ArgoCD applications will have to be deployed in the namespace `gitops-developers`. This is a common namespace across all Tenants. Talk to your Tenant administrator regarding the naming standard.

When can it be useful to use user-defined ArgoCD applications:
* If you need to use a private helm repository
* If you want more freedom with your ArgoCD application definitions. 

The following file `poseidon1_main_repo/applications/dev/ex1-helm-app.yml` shows an example of how to deploy a application.



# Helpful Resources:
- https://kubectl.docs.kubernetes.io/references/kustomize/builtins/#_helmchartinflationgenerator_
- 