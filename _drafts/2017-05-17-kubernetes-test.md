- Install kubectl
  - https://kubernetes.io/docs/tasks/tools/install-kubectl/
  - I did: `brew install kubectl`
- Install Minikube (for local testing)
  - https://kubernetes.io/docs/tasks/tools/install-minikube/
  - `curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.19.0/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/`
  - `minikube start`
  - To connect to cluster later: `kubectl config use-context minikube`


## Goals

- Automate creation of new nodes (not necessarily new env)
- N node cluster behind some kind of load balancer (or round-robin DNS w/ health check)
- Automated deploys
- Simple (small project)

Things that have to be set up on a new env:

- Load balancer
- Routing rules
- SSL cert

Things that have to be set up on a new node:

- Installation of correct version of nodejs
- Deploy correct software and configuration to that node
- Tag the node by env and role

Things that have to be done on each deploy:

- Take node out of load balancer
- Get new code out there
- Deliver configuration
- npm install
- npm prune
- Restart or otherwise apply new code
- Check that it is healthy
- Add node back into load balancer

## Option 1: Scripts

Set up new node:

- aws-cli to create a new instance
- bash over ssh to do the following:
- check if the correct version of node is installed. if not, install.

Deploy:

- Use a CI server to build assets

## Option 2: kubernetes/gcloud

Install gcloud https://cloud.google.com/sdk/docs/quickstart-mac-os-x
Install kubectl, either directly via `brew install kubectl` or through gcloud via `gcloud components install kubectl`
Might want this component later: `docker-credential-gcr`
Connect to cluster by logging into google cloud, clicking "Connect" by your cluster
Or use minikube and connect to it with "minikube start"

`kubectl create secret generic stage-mysql --from-literal=MYSQL_HOST=104.154.21.59 --from-literal=MYSQL_USER=root --from-literal="MYSQL_PASSWORD=dk4%rkg&kflghwlekfjfklkhqxvHyE8)" --from-literal=MYSQL_DATABASE=pacas_staging --from-literal=MYSQL_CONNECTION_LIMIT=2`

create a deployment.yaml file

create a deployment from that file: `kubectl create -f deployment.yaml`

update the deployment: `kubectl replace -f deployment.yaml`

check status of a deployment: `kubectl rollout status deployment/pacas-api-deployment`
