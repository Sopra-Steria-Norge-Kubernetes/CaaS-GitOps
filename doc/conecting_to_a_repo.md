
# Connecting to a Git repository
The `argocd` feature can connect to a Git repository through a Personal Access Token (PAT) or a GitHub App. The table below shows a more detailed description of each variable in the `argocd` feature under the `main_git_repository`. The table is split into three categories: 
- **Default**: variables that both connection methods need
- **PAT**: variables needed to connect with a GitHub PAT token
- **GitHub app**: variables need to configure the GitHub app

| **Variable**         | **Description**                                                                                                     | **Example**                                | **Type**                  |
|----------------------|---------------------------------------------------------------------------------------------------------------------|--------------------------------------------|---------------------------|
| **Default**              |                                                                                                                     |                                            |                           |
| `repourl`            | The URL of the git repository which ArgoCD will use as its "source of truth"                                        | https://github.com/customer-repo/openshift | String                    |
| `basepath`           | The basepath of the git repository where ArgoCD manifests are stored                                                | poseidon1/                                 | String                    |
| `encrypted_url`      | The URL of the git repository encrypted with sealedsecrets                                                          | See description below                      | Kubeseal encrypted string |
| `encrypted_type`     | Type should always be "git" and encrypted with sealedsecrets                                                        | See description below                      | Kubeseal encrypted string |
| **PAT**                  |                                                                                                                     |                                            |                           |
| `encrypted_password` | The git Personal Access Token for the service account connecting to the git repository encrypted with sealedsecrets | See description below                      | Kubeseal encrypted string |
| `encrypted_username` | The username of the service account connecting to the git repository encrypted with sealedsecrets                   | See description below                      | Kubeseal encrypted string |
| **GitHub App**           |                                                                                                                     |                                            |                           |
| `enable_app`         | Whether or not to use GitHub App to authtenticate ArgoCD with your Git Repository. Default false.                   | True / False                               | Boolean                   |
| `id`                 | The app id for your Github App encrypted with sealedsecrets                                                         | See description below                      | Kubeseal encrypted string |
| `installation_id`    | The installation id for your GitHub App encrypted with sealedsecrets                                                | See description below                      | Kubeseal encrypted string |
| `private_key`        | Private key for your GitHub App encrypted with sealedsecrets                                                        | See description below                      | Kubeseal encrypted string |



## Encrypt and configure the Argo-specific information

# Encrypt PAT and GitHub App variables for ArgoCD

## Introduction
This user guide provides detailed instructions and templates for setting up encrypted variables for the ArgoCD feature within the OpenShift Tenant framework. It is divided into Two main sections:
1. **Installation of Kubeseal**: This section guides you through installing Kubeseal, a tool essential for encrypting the variables.
2. **Authentication Methods for ArgoCD with GitHub**: This part explains two distinct ways to authenticate with ArgoCD using GitHub:
   1. Using a Personal Access Token (PAT): It details the steps to use a Personal Access Token for authentication.
   2.  Using a GitHub App: This method outlines how to authenticate using a GitHub App.

## Install KubeSeal

If not already installed, you can install `kubeseal` on your machine with the following commands:

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/<release-tag>/kubeseal-<version>-linux-amd64.tar.gz
```
```bash
tar -xvzf kubeseal-<version>-linux-amd64.tar.gz kubeseal
```
```bash
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

Remember to replace `<release-tag>` and `<version>` with the latest release version. For example:

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.22.0/kubeseal-0.22.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.22.0-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

For Mac users you can use Homebrew package manager:

```bash
brew install kubeseal
```

## Authentication Methods for ArgoCD with GitHub

For both authentication methods, you must add the GitHub repository URL and the basepath for your applications:
- `argocd.main_git_repository.repourl`: Plaintext HTTPS Git URL for repository where your applications are stored 
-  `argocd.main_git_repository.basepath`: Basepath is where argoCD will look to find your application definions. ArgoCD will look in your git repository for path: `$basepath/$environment/applications/`

### Authenticate with PAT

#### 1. Create a secret for PAT

Create a secret for your git repository
Fill out the yaml below. 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <Name of your tenant>
  namespace: gitops-developers
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  url: <HTTPS url of git repository>
  type: git
  password: <Password of personal access token>
  username: <Git username associated with personal access token>
```
```
> [!ATTENTION]
> For the git url use the HTTPS url of the git repository.
```

#### 2. Encrypt the Secret with Kubeseal

To seal the secret with `kubeseal`, you can use the following command:

```bash
kubeseal --cert /path/to/pub.cert --scope namespace-wide -f secret.yaml -o yaml
```
Replace `/path/to/pub.cert` with the path to your public certificate. `secret.yaml` is the path to the Secret YAML file you created in the first step. The sealed secret will be outputted to `sealed_secret.yaml`.

#### 3. Fill out the argo-specific section in your tenant definition

Fill out the argo-specific section by copying the field `password` of the sealed secret to `argocd.main_git_repository.encrypted_password`. Likewise, copy these fields the same way: 

* `url` --> `encrypted_url`
* `type` --> `encrypted_type`
* `username` --> `encrypted_username`
* `password`--> `encrypted_password`

Remember to also add the required variables `argocd.main_git_repository.repourl` and `argocd.main_git_repository.basepath` to the tenant definition.

### Authenticate with a GitHub App


#### 1. Create a GitHub App for ArgoCD
Follow the steps below to create a GitHub App suited for ArgoCD:

1. Follow the steps in the [Github documentation](https://docs.github.com/en/enterprise-cloud@latest/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)

2. For connecting to ArgoCD, ensure that your application has at least `Read-only` permissions to the `Contents` and `metadata` of the repository.  This is the minimum requirement. See  [ArgoCD documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/#github-app-credential) for further explanation.

3. Generate a Private key inside the GitHub App. This will later be mapped and encrypted to the tenant variable `github_app.private_key`

4. Go to the tab Install App, install your new app, and choose the repositories you want to include. Finally, press install.

5. From the web URL inside your app extract the `githubAppInstallationID`. This will later be mapped to the variable `github_app.installation_id` in the tenant definition. 

6. Click on app settings and extract the App ID, which will later be mapped to the tenant variable `github_app.id`

   ![alt text](pictures/github_app.png "GitHub app Example")

#### 2. Create a secret for Github App

Create a secret for your Github App
Fill out the yaml below.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <Name of your tenant>
  namespace: gitops-developers
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: <HTTPS url of git repository>
  githubAppId: <GitHub App Id>
  githubAppInstallationID: <GitHub Installation Id>
  githubAppPrivateKey: |
    -----BEGIN RSA PRIVATE KEY-----
       ** secret key here **
    -----END RSA PRIVATE KEY-----
type: Opaque

```

#### 3. Encrypt the Secret with Kubeseal

To seal the secret with `kubeseal`, you can use the following command:

```bash
kubeseal --cert /path/to/pub.cert --scope namespace-wide -f secret.yaml -o yaml
```
Replace `/path/to/pub.cert` with the path to your public certificate. `secret.yaml` is the path to the Secret YAML file you created in the first step. The sealed secret will be outputted to `sealed_secret.yaml`.

#### 4. Fill out the argo-specific section in your tenant definition
Fill out the argo-specific section by copying the following fields under the `spec.encryptedData` in the Sealed secret to the tenant variable `argocd.main_git_repository` inside your tenant definition:

   * `true` --> `github_app.enable_app`
   * `type` --> `encrypted_type`
   * `url` --> `encrypted_url`
   * `githubAppId` -> `github_app.id`
   * `githubAppInstallationID` -> `github_app.installation_id`
   * `githubAppPrivateKey` -> `github_app.private_key`

Remember to also add the required variables `argocd.main_git_repository.repourl` and `argocd.main_git_repository.basepath` to the tenant definition.

This should result in a argo-specific section looking like this:
```yaml
argocd: 
    main_git_repository:
      repourl: 
      basepath:
      encrypted_url: 
      encrypted_type: 
      github_app: 
        enable_app: true
        id: 
        installation_id: 
        private_key: 
```




