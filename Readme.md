# ArgoCD Playground on AKS
This repo is ment to be a playground for GitOps using ArgoCD. There are several ways to implement GitOps practices depending on the requirements and the context. In this repo we will use Azure Kubernetes Service, Traefik as an ingress controller, and show some specific way to do Continuous Deployments of Helm charts on Kubernetes. 


## Prerequisites

* AKS with Traefik ingress (Can be any other ingress controller, but we use Traefik in this example)
* Azure dns zone, or some other DNS entry that points to your Traefik public IP
* `cp argo-ingress-route.yaml.example argo-ingress-route.yaml` and fill the CHANGEME parts with your DNS entry
* Install argo CLI. On mac using `brew install argocd`


## ArgoCD Installation

* `kubectl create namespace argocd`
* `kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`
* `kubectl -n argocd get all` to check if everything is up
* `kubectl apply -f argo-ingress-route.yaml`
* If your Traefik does TLS termination you might end up with too many redirects issue, because the communication between your ingress and ArgoCD might be using http while Argo requires https. So either set your ingress backend communication to https or use this: `k -n argocd patch cm/argocd-cmd-params-cm -p '{"data":{"server.insecure":"true"}}'`. This allows a http connection between your Traefik and ArgoCD.


## Configuration

* `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo` to get the password for user `admin`
* `argocd login <Your DNS Entry>`
* `argocd account update-password`
* `kubectl -n argocd delete secret argocd-initial-admin-secret`
* `kubectl config get-contexts -o name`
* `argocd cluster add <context name>`


## Use Cases
There are different ways and use cases for deploying applications with ArgoCD. We will focus on the following
* Microservice that gets deployed via helm 
* Microservice has to be promoted through multiple environments (dev, prod)
* Changes on the Helm Chart, The application code and value files has to result in a new deployment/update.


## Concept 
* We create an ArgoCD application for each service and each environment. See the [argo](./argo/) directory for details. The application is a k8s manifest file that has to applied using `kubectl`.  
* The ArgoCD application "listens" on the git repo where the helm chart lives. 
* Considering that every change (app code, helm chart, values) results in a new commit on the repo where the chat lives.
  * App code changes result in a new Docker image and an update of the appversion (and maybe the chart version)
  * Changes to the chart results in an update of the chart version (maybe also appversion)
  * Config changes result in the update of value files
* Every commit to the repo of the helm chart results in an update of the deployment.
* We could change the app code without changing the appversion and or chart version, by just changing the value file and update the `image.tag`. This could be helpfull for deploying PRs for example without having to make a commit to the Chart.yaml. 


## Getting Started
* `kubectl config set-context --current --namespace=argocd`
* `kubectl apply -f ./argo/argo-app-dev.yaml`
* `kubecl apply -f ./argo/argo-app-prod.yaml`

## Updating deployed application
* Building a Docker image that contains a new version of an application
* Either: 
  * Update image.tag in the values file of the environment that should be updated
  * Update appversion (and chartversion) of the helm chart


## Additional Information
* https://argo-cd.readthedocs.io/en/stable/getting_started/
* https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#traefik-v22
* https://argo-cd.readthedocs.io/en/stable/operator-manual/server-commands/additional-configuration-method/
* https://codefresh.io/blog/simplify-kubernetes-helm-deployments/ 
* https://codefresh.io/blog/stop-using-branches-deploying-different-gitops-environments/ 
* https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/application.yaml 


## Conclusions
* Pipelines that build Docker images still have to be there.
* Pipelines that build Helm Charts still have to be there. Though as the argo application listens to the repo where the helm chart lives in, instead of a helm chart repository, we still would need to build a helm chart (preferably after a deployment and tests). 
* All value files for a chart reside inside the chart itself.

## TODO
* Maybe Helm Application Set is something that works good