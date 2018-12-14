# Helm Testing

This repository will go through the steps for installing Helm and deploying a sample Helm Chart. 

## Introduction
Helm is a package manager for Kubernetes maintained by the Cloud Native Computing Foundation (CNCF). Helm uses a packaging format called Charts -- a collection of files that describe a set of Kubernetes resources. A single Chart can deploy an application, and may depend on other Charts e.g., a Wordpress Chart may require the MySQL Chart. Charts use templates which are dynamic YAML files containing logic and variables. Helm uses the templates to create YAML files that Kubernetes can understand. Helm has two parts: a client (`helm`) and a server (`tiller`). Tiller runs inside of the Kubernetes cluster and manages Chart installations. Helm runs outside of the cluster e.g., on your machine, CI/CD server, etc. 

## Local (Unsecured) Installation
Installation instructions:
1. Download the latest binary (https://github.com/helm/helm/releases/tag/v2.11.0)
2. Move `helm` binary to PATH
3. Initialize Helm and install Tiller: `helm init`
    * Creates `$home/.helm` where Helm-related files are stored
    * Adds local repository at `http://127.0.0.1:8879/charts`
    * Installs Tiller on Kubernetes cluster specified by `kubectl config current-context`
    * By default, Tiller is deployed into the `kube-system` namespace
    * To verify Tiller is running: `kubectl get pods -n kube-system`
4. Install an example Chart:
    * Get latest list of Charts: `helm repo update`
    * Install an official `stable` Chart: `helm install stable/mysql`
    * `helm inspect stable/mysql` to get details about release
    * `helm list` to show deployed releases
    * `helm delete <release-name>` to uninstall release

## Production (Secured) Installation
Helm and Tiller are designed to install, remove and modify logical applications which requires cluster-wide operations. Therefore, care must be taken in a multi-tenant cluster. The default installation of Tiller (`helm init`) is into the `kube-system` namespace without RBAC rules. There are two common architectural design:
  1. Run a Tiller instance in each namespace where Helm applications will be deployed, and
  2. Run a Tiller instance in its own namespace and deploy to other namespaces

The first approach seems beneficial for multi-tenant clusters as users can deploy Helm applications only if they have access to the namespace, and if they have credentials (token and certificate) to the service account bound to the Tiller deployment. The second approach seems beneficial for clusters where only a few people need to deploy Helm applications (our case for now). These people will be given access to the Tiller namespace and the credentials to the service account bound to the Tiller deployment. The latter setup is easier than the former; however, it should be noted that access to Tiller in the latter approach will allow one to deploy Helm applications to all namespaces! 

The procedure for implementing the second approach is described below:

1. Make sure kubectl is interacting with Docker EE cluster:
  * Download the CLI bundle from the UCP UI
  * `cd /path/to/ucp-bundle`
  * `eval "$(<env.sh)"`
  * `kubectl get nodes` to make sure nodes are referring to Docker EE nodes
2. Create a namespace where only Tiller will live:
  * Defined in `tiller-ns.yaml`
  * `kubectl create -f tiller-ns.yaml`
3. Create a Service Account for Tiller
  * Defined in `tiller-sa.yaml`
  * `kubectl create -f tiller-sa.yaml`
4. Create Roles and RoleBindings for Tiller Service Account to deploy Helm applications into the `default` namespace:
  * Defined in `tiller-rbac.yaml`
  * `kubectl create -f tiller-rbac.yaml`
  * Can easily extend `tiller-rbac.yaml` to include other namespaces as needed
5. Install Tiller using Helm
  * `helm init --skip-refresh --service-account tiller-sa --tiller-namespace tiller-ns`
6. Extract the Service Account's Secret: `kubectl describe sa tiller-sa -n tiller-ns`
7. Extract the Service Account's Token: `kubectl get secret tiller-sa-token-xxxx -n tiller-ns -o "jsonpath={.data.token}" | base64 -d`
8. Extract the Service Account's Certificate: `kubectl get secret tiller-sa-token-xxxx -n tiller-ns -o "jsonpath={.data['ca\.crt']}" | base64 -d`
9. Modify kube config file with the previous information:
  * Defined in `kube-config.yaml`
10. Give kube config file to each user that needs access to Tiller
  * They need to change `current-context` to `helm-tiller-account` when they are deploying Helm applications

## Creating a Helm Chart
- `helm init <project-name>` creates a Helm Chart template
- To debug Helm Chart: `helm install --tiller-namespace tiller-ns --debug --dry-run /path/to/<project-name>`
- To install Helm Chart: `helm install --tiller-namespace tiller-ns --namespace <namespace-to-deploy-to> /path/to/<project-name>`

## Sample Helm Chart
The `elastic-demo` directory contains a sample Helm Chart which deploys the Elastic Stack centralized logging for a dummy microservice. See https://github.com/cheuklau/elastic-stack-logging for more information. `elastic-demo` is made up of the following files:

* `Charts.yaml` - contains general chart information e.g., version number, project name, etc.
* `values.yaml` - contains values that are used to populate variables inside of `templates/`
* `templates/` - directory that contains the deployment and service yaml files for all of the components in the application e.g., microservice, logstash, elasticsearch and kibana

To install the application using Helm
  * Go to `kube-config.yaml` and make sure you are in the `helm-tiller-context` context
  * `kubectl config current-context` to verify current context
  * `helm install --tiller-namespace tiller-ns --namespace default /path/to/elastic-demo`

## Helm Commands
Some common Helm commands are:
1. `helm init` to install Tiller on the cluster
2. `helm reset` to remove Tiller from the cluster
3. `helm install` to install a Helm chart
4. `helm search` to search for a chart
5. `helm list` to list releases
6. `helm upgrade` to upgrade a release
7. `helm rollback` to rollback a release

## Resources

* https://docs.helm.sh/using_helm/
* https://jeremievallee.com/2018/05/28/kubernetes-rbac-namespace-user.html