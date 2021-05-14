# Admin GitOps Repository

This repository contains cluster and namespace configuration.  It also contains configuration to deploy Operator instances such as Argo CD and Bitnami Sealed Secrets.

Only "cluster admins" should have access to commit to main.  This will help limit who can modify the cluster(s) and core cluster resources such as namespaces, resource quotas, and sealed secrets.

## Repository Layout

This repository has three main directories:
* **argocd**: This contains Argo CD resources (`AppProject` and `Project`) to configure two clusters - `non-prod` and `prod`
* **base**: This contains *shared* configuration that can be referenced from multiple *overlays*.  For example, the configuration of `argocd`, and `sealed-secrets` is the same between clusters, so this configuration is kept in the *base* folder and referenced from the appropriate *overlay* directory.
* **overlays**: This directory is split into `non-prod` and `prod` sub directories and contains the definition of each cluster.  In each *overlay* directory you will find a `kustomization.yaml` file that lists the resources (including bases) that make up the overall group of resources that are to be created (and managed) in a cluster.

## Prerequisites

To run this demo you will need:
* The "oc" command line tool.  This can be downloaded directly from OpenShift by clicking on the **?** icon near the top-right corner of the console, selecting "Command Line Tools" and downloading the binary for your operating system.
    * It is recommended to add this cli to your *path*.


## Getting Started

Reproducing this in your own cluster is mostly automated, but since there is sensitive information that needs to be stored in a git repository (GitHub and Quay credentials), there are a few manual tasks required in order to first encrypt your secrets.  

### 1. Fork the Repositories

Fork each of the demo repositories into your own GitHub account.  These are:
* [Contoso University DotNet App](https://github.com/pittar-sandbox/dotnet-contoso-app)
* [Admin GitOps Repository](https://github.com/pittar-sandbox/dotnet-contoso-gitops-admins)
* [Developers GitOps Repository](https://github.com/pittar-sandbox/dotnet-contoso-gitops-developers)

### 2. Update Argo CD Applications With Your Git Repositories

This part is tedious, but since you will need to update the *Applications* in the `argocd` directory of both the "admins" and "developers" repositories.  These files can be found in:

```
dotnet-contoso-gitops-admins
|-- argocd
```
and
```
dotnet-contoso-gitops-developers
|-- argocd
```

These files end in `-app.yaml`.  You need to replace the `repoURL` with the link to your own git repository.

### 3. Install the OpenShift GitOps Operator in Each Cluster

Log into your OpenShift clusters and install the "OpenShift GitOps" Operator.
1. From the "Administrator" view, select `Operators -> Operator Hub` from the left menu.
2. Search for **OpenShift GitOps**
3. Select this operator and click on **Install**, accepting all the defaults.

This will install the OpenShift GitOps operator, as well as a default instance of Argo CD in the `openshift-gitops` namespace.  This instance of Argo CD is meant for *cluster admins* to use to configure the cluster.

### 4. Create and Configure Namespaces with Argo CD

First, log into your "non-prod" OpenShift cluster using the `oc` cli as a *cluster-admin* user.

From the root of the `dotnet-contoso-gitops-admins` repository, run:

```
oc apply -k argocd/non-prod/01-namespaces
```

Once the namespaces are configured, install an instance of Argo CD for "Developers" and an instance of "Sealed Secrets" into your cluster with:

```
oc apply -k argocd/non-prod/02-operator-instances/
```

**Note:** In OpenShift 4.6, the `argocd-for-developers-instance` will remain "OutOfSync".  This is fixed in Argo CD 2.0 / OpenShift 4.7.  For now, this is fine.  Everything will be running properly.

### 5. Generate Sealed Secrets for GitHub and Quay Credentials

Part of this demo pushed new container images into your Quay.io account and creates/pushes branches in your git registry.  For this, your pipeline will need access to credentials.  In your cluster, information like this is managed by Kubernetes "Secrets".  However, it is a bad idea to put a standard "Secret" in a Git repository, since the data is only base64 encoded.  How do you manage Secrets in a GitOps world, then?  One option is to use Bitnami Sealed Secrets.

In the previous step Sealed Secrets was installed into a new namespace called `sealed-secrets`.  Two helper pipelines have also been created in that namespace to make generated "sealed secrets" for your credentials easier.

1. From the Developer Console, go to the "sealed-secrets" project and click on "Pipelines" in the left menu.
2. Click on the "generate-git-repository-sealed-secret" Pipeline, then from the drop-down near the top-left of the screen select "Start".
3. Fill in the fom with the following values:
    * Keep the default value for SEALED_SECRETS_NAMESPACE
    * SECRET_NAME: `github-creds-secret`
    * SECRET_NAMESPACE: `contoso-cicd`
    * GIT_HOST - keep default
    * GIT_USERNAME: your github username
    * GIT_EMAIL: your github account email address
    * GIT_PASSWORD: If using GitHub, create an access token and use this instead of your password.
4. Click "Start"

If you watch the logs, you will see a multi-line JSON object printed at the end of the pipeline.  This is your encrypted git credentials secret!

Create a new file in your gitops admin repo for this file:
```
dotnet-contoso-gitops-admins/overlays/01-namespaces/contoso-cicd/github-creds-secret-sealedsecret.json
```
Copy the JSON from the log output into this file.

Similarly, we will create an encrypted Quay secret:
1. From the Developer Console, go to the "sealed-secrets" project and click on "Pipelines" in the left menu.
2. Click on the "generate-image-registry-sealed-secret" Pipeline, then from the drop-down near the top-left of the screen select "Start".
3. Fill in the fom with the following values:
    * Keep the default value for SEALED_SECRETS_NAMESPACE
    * SECRET_NAME: `quay-creds-secret`
    * SECRET_NAMESPACE: `contoso-cicd`
    * REGISTRY_HOST - keep default
    * REGISTRY_USERNAME: Your quay.io username
    * REGISTRY_EMAIL: Your quay.io account email address
    * REGISTRY_PASSWORD: Your quay.io password.
4. Click "Start"

If you watch the logs, you will see a multi-line JSON object printed at the end of the pipeline.  This is your encrypted quay.io credentials secret!

Create a new file in your gitops admin repo for this file:
```
dotnet-contoso-gitops-admins/overlays/01-namespaces/contoso-cicd/quay-creds-secret-sealedsecret.json
```
Copy the JSON from the log output into this file.

Finally, in the same directory where you created these new secrets, open the `kustomization.yaml` file and uncomment the two "resources" lines that reference these new files.
