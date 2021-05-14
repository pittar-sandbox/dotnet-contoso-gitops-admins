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