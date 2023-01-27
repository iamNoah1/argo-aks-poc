# ArgoCD Playground on AKS

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
* We create an ArgoCD application for each service and each environment. See the [argo](./argo/) directory for details. 
* The ArgoCD application "listens" the helm repository. 
* The target version of the helm chart can be defined in the application manifest.
* Considering that every change (app code, helm chart, values) results in a new commit on the chart thus a new docker image and helm chart.
* To promote a that change to any system, we simply make a commit to the respective application manifest, where we update the targetversion of the helm chart


## Getting Started
* `kubectl config set-context --current --namespace=argocd`
* `kubectl apply -f ./argo/argo-app-dev.yaml`
* `kubecl apply -f ./argo/argo-app-prod.yaml`


## Additional Information
* https://argo-cd.readthedocs.io/en/stable/getting_started/
* https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#traefik-v22
* https://argo-cd.readthedocs.io/en/stable/operator-manual/server-commands/additional-configuration-method/
* https://codefresh.io/blog/simplify-kubernetes-helm-deployments/ 
* https://codefresh.io/blog/stop-using-branches-deploying-different-gitops-environments/ 
* https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/application.yaml 


## Conclusions
* Pipelines that build Docker images still have to be there
* Pipelines that build Helm Charts still have to be there
* All value files for a chart reside inside the chart itself.

## TODO
* Find if it is better to 
  * Reference helm repos and promote changes to argo app manifest changes or 
  * Let the argo app have the helm chart repo as source and having the changes on the values file  
* Let's try: 
  * Build another version of the app 
  * Update Helm Chart (because we do it like this at the moment -> new app version => new Chart version)
  * Work with changes of application manifest
* Maybe Helm Application Set is something that works good