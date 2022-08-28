# Installation

## kubectl

- kubectl is kubernetes cli

```bash
VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
curl -LO https://storage.googleapis.com/kubernetes-release/release/$VERSION/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

## Krew

Krew is kubectl plugin manager: https://github.com/kubernetes-sigs/krew/
Installation instructions: https://krew.sigs.k8s.io/docs/user-guide/setup/install/

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

Then in .bashrc:

```
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

## kubectx and kubens

kubectx is a tool to switch between contexts (clusters) on kubectl faster.
kubens is a tool to switch between Kubernetes namespaces (and configure them for kubectl) easily.
https://github.com/ahmetb/kubectx

To install them with Krew:

```
kubectl krew install ctx
kubectl krew install ns
```

## multipass

IMPORTANT NOTE FOR WSL. When creating multipass machines and installing kubernetes on them,
kubectl is not able to access them because of network rules.
To solve this issue,

- we can launch this from powershell as admin: `Get-NetIPInterface | where {$_.InterfaceAlias -eq 'vEthernet (WSL)' -or $_.InterfaceAlias -eq 'vEthernet (Default Switch)'} | Set-NetIPInterface -Forwarding Enabled` (see https://stackoverflow.com/questions/65716797/cant-ping-ubuntu-vm-from-wsl2-ubuntu).
- we can also do some port forwarding: see https://pypi.org/project/WSL-Port-Forwarding/

NOTE: I am still unclear if we need both of them or just the first

- Install multipass: https://multipass.run/ : allows to easily launch a VM with either hyperV or virtual box
  To use it on WSL, it must be installed on windows, not on linux. And then, we can create
  an alias to use it directly on WSL: for ex: `alias multipass="/mnt/c/'Program Files'/Multipass/bin/multipass.exe"`
- set driver to hyper V: `multipass set local.driver=hyperv` or virtualbox: `multipass set local.driver=virtualbox`

- IMPORTANT: to use the mount features in WSL, you must be in a folder accessible by windows, such as `/mnt/c/...`
- enable mounting with `multipass set local.privileged-mounts=true`
- create a vm:

  - basic : `multipass launch node1`
  - with parameters: ` multipass launch -n node2 -c 2 -m 3G -d 10G` (2 cores, 3G ram, 10G disk size)

- start/stop/delete vm: `multipass start node1` / `multipass stop node1` / `multipass delete -p node1`
- get infos: `multipass infos node2`
- list vms: `multipass list`
- launch a shell: `multipass shell node1`
- launch a command on the node, for ex:
  - install docker: `multipass exec node1 -- /bin/bash -c "curl -sSL https://get.docker.com | sh"` (here, it launches the bash shell and execute the command between quotes. It is the same as getting into the sell and executing the command between quotes)
  - check that docker is correctly installed: `multipass exec node1 -- sudo docker version`
- copy file from local machine: `multipass transfer /tmp/test/hello node1:/tmp/hello` (both ways work)
- mount a folder: `multipass mount ./testfolder node1:/usr/share/test`.
  Note that if you want to use an absolute path: `multipass mount $(wslpath -w /mnt/d/projects/kubernetes-tuto/test2/) node1:/usr/share/test`
- unmount a folder: `multipass umount node1:/usr/share/test` or unmount all : `multipass umount node1`

# Create local clusters

Summarizes multiple ways to create local clusters

## Minikube

- Install minikube:

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/
```

- to start minikube: `minikube start` (docker desktop needs to be running for this to work on wsl)
- to start minikube with 3 nodes while specifying docker driver: `minikube start --driver docker --nodes 3`
- see nodes: `kubectl get no`
- destroy cluster: `minikube delete`

## Kind

Kind (Kubernetes in docker) allows to deploy a kubernetes cluster such that each node runs in a docker container

- Install with go: `go install sigs.k8s.io/kind@v0.14.0` (replace v0.14.0 with stable version)
- Create a cluster: `kind create cluster --name k8s`
- We can see a container was created when running `docker container ls`: this container runs kubernetes
- Note that kind automatically creates a context and sets it as current context: `kubectl config get-contexts`
- We can list the nodes of the cluster: `kubectl get nodes`

To create clusters with several nodes, we need a yaml config file:

```yaml
# kindconfig.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

We create the cluster with: `kind create cluster --name k8s-2 --config kindconfig.yaml`

- Get the clusters with `kind get clusters`
- Cleanup: `kind delete cluster --name k8s`

## MicroK8s

Easy to use solution very good for local dev. Can also be configured with several nodes and used in prod

Note: In this example, install is done on VM:

- create VM: `multipass launch --name microk8s --mem 4G`
- install microk8s on VM: `multipass exec microk8s -- sudo snap install microk8s --classic`
- get config in yaml on local machine: `multipass exec microk8s -- sudo microk8s.config > microk8s.yaml`
- set KUBECONFIG var: `export KUBECONFIG=$PWD/microk8s.yaml`
- see nodes: `kubectl get no` (notes: see multipass notes on wsl)
- run `microk8s status` inside the vm to see all enabled and disabled addons
- before using cluster and deploying app, we can install CodeDNS via dns addon: `microk8s enable dns`

# K3s

Very light kubernetes distribution. Same as above, installed on a VM

- lauch: `multipass launch --name k3s-1`
- get ip: `IP=$(multipass info k3s-1 | grep IP | awk '{print $2}')` (not that IP will change if you restart host machine)
- install : `multipass exec k3s-1 -- bash -c "curl -sfL https://get.k3s.io | sh -"`
- create temp cfg in local machine : `multipass exec k3s-1 sudo cat /etc/rancher/k3s/k3s.yaml > k3s.cfg.tmp`
- replace IP in config file: `cat k3s.cfg.tmp | sed "s/127.0.0.1/$IP/" > k3s.cfg`
- set kubeconfig var: `export KUBECONFIG=$PWD/k3s.cfg`
- get nodes: `kubectl get nodes` (notes: see multipass notes on wsl)

- create other vms:

```bash
for node in k3s-2 k3s-3;do
  multipass launch -n $node
done
```

- retrieve token from master node: `TOKEN=$(multipass exec k3s-1 sudo cat /var/lib/rancher/k3s/server/node-token)`
- Join the nodes:

```bash
# Join node2

$ multipass exec k3s-2 -- \
bash -c "curl -sfL https://get.k3s.io | K3S_URL=\"https://$IP:6443\" K3S_TOKEN=\"$TOKEN\" sh -"

# Join node3
$ multipass exec k3s-3 -- \
bash -c "curl -sfL https://get.k3s.io | K3S_URL=\"https://$IP:6443\" K3S_TOKEN=\"$TOKEN\" sh -"

```

# K3d

Can deploy K3s clusters such that each node runs in a docker container. Installed directly
on local maching. Docker desktop must be running on WSL

- install: `curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash`
- check version: `k3d version`
- creates a cluster named k3s with 2 workers and 1 server: `k3d cluster create k3s --agents 2`
- more infos: `k3d cluster create --help`
- list containers: `docker ps`: we should see servers and workers
- swith context: `kubectl config use-context k3d-k3s`
- list nodes: `kubectl get nodes`
- delete cluster: `k3d cluster delete k3s`

# Production clusters

## managed solutions

Lots of cloud providers propose solutions to manage clusters for a prod environment:

- GKE: google kubernetes engine
- AKS: Azure Container service
- EKS: Amazon Elastic Container service
- DigitalOcean
- OVH

Using them is straightforward. Creation can be done using the web interface, then we
download the config file and modify KUBECONFIG var to point to the config. Then, we
can see the nodes with `kubectl get nodes`

## install kubernetes without providers

lots of solutions:

- kubeadm:
  - https://kubernetes.io/fr/docs/setup/production-environment/tools/kubeadm/
  - nodes must be provisionned prior to setting up cluster
- kops:
  - https://github.com/kubernetes/kops
  - deals with full cluster lifecycle
- kubespray:
  - https://github.com/kubernetes-sigs/kubespray
  - uses ansible to set up kubernetes
- rancher:
  - https://rancher.com/
  - allows to handle app lifecycle with a catalog
- Docker EE (deploy Swarm and Kubernetes)
- terraform + ansible (provisionning + config)

TODO: add instructions to set up cluster with kubeadm

# Pods

## Introduction

- Smallest applicative unit in kubernetes
- Group of countainer in same isolation context
- Share network stack and volumes
- Dedicated ip address, no NAT (network address translation) for communication between pods

- An app consists of several specification of pods.
- Each specification corresponds to a microservice
- Horizontal scaling with nb of replica of a pod

## Lifecycle

```
# example pod specification
# www.yaml
apiVersion: v1
kind: Pod
metadata:
  name: www
spec:
  containers:
    - name: nginx
      image: nginx:1.12.2
```

- launch a pod: `kubectl create -f <pod-specification.yaml>`
- list pods: `kubectl get pods`
- describe a pod: `kubectl describe pod <pod_name>`
- logs of a pod: `kubectl logs <pod-name> [-c <container-name>]` (no need to specify container-name if pod has only one container)
- launch a command in a pod: `kubectl exec <pod-name> [-c <container-name>] -- <command>`
  - ex: interactive shell: `kubectl exec -it <pod-name> -- /bin/bash`
- forward port of www pod to host machine: `kubectl port-forward www 8080:80`
- delete a pod: `kubectl delete pod <pod-name>`

# Summary of useful concepts

- Container:

  - is a process
  - visible from host machine
  - with limited vision of the system
  - with limited accessible resources
  - combines two linux primitives: namespace and control groups (cgroups)

- Docker:

  - simplifies use of containers
  - brings concept of images = packaging of an app and its dependencies instantiated in a container that can be deployed in several environments
  - see http://hub.docker.com

- Microservices architecture:

  - application separated in several services
  - each service can be developed / updated independently
  - well defined interface is needed for communication between services
  - containers are particularly adapted for Microservices
  - complexity is moved to the orchestration of full application

- Cloud native app:

  - Microservices oriented application
  - packaged in containers
  - Dynamic orchestration
  - a lot of project supported by Cloud Native Computing Foundation: https://www.cncf.io (kubernetes for ex)

- Devops

  - goal: minimise time to deliver a feature
  - Frequent deployments
  - With tested code
  - With automated processes
  - Short improvement loop
  - Example:
    - infrastructure (aws, digitalOcean, azure)
    - provisionning (terraform, CloudFormation)
    - build (github, gitlab, docker)
    - test (selenium, jenkins, circleci)
    - deploy (Ansible, CHEF, registry docker)
    - run (kubernetes, Swarm)
    - mesure (prometheus, elastic)

- Kubernetes role:
  - deal with applications running in containers (deployment, scaling, self-healing)
  - ensures that application is running as intended (reconciliation loop)
  - can deal with stateless and stateful apps
  - can handle secrets and configs
  - can run long-running processes or batch jobs
  - RBAC (role based access control)
