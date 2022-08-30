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
- get ip: `IP=$(multipass info k3s-1 | grep IP | sed "s/\r//" | awk '{print $2}')` (not that IP will change if you restart host machine)
- install : `multipass exec k3s-1 -- bash -c "curl -sfL https://get.k3s.io | sh -"`
- create config with correct IP: `multipass exec k3s-1 sudo cat /etc/rancher/k3s/k3s.yaml | sed "s/127.0.0.1/$IP/" > k3s.cfg`
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

- Pods can be created manually but are usually created in a ReplicaSet inside a Deployment
- They are exposed in the cluster or to the outside via a Service

## Lifecycle

```yaml
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

- launch a pod from a file: `kubectl apply -f <pod-specification.yaml>`
- launch a pod directly from an image: `kubect run <pod-name> --image=<image-name>`
- list pods: `kubectl get pods`
- describe a pod: `kubectl describe pod <pod_name>`
- logs of a pod: `kubectl logs <pod-name> [-c <container-name>]` (no need to specify container-name if pod has only one container)
- launch a command in a pod: `kubectl exec <pod-name> [-c <container-name>] -- <command>`
  - ex: interactive shell: `kubectl exec -it <pod-name> -- /bin/bash`
- forward port of www pod to host machine: `kubectl port-forward www 8080:80`
- delete a pod: `kubectl delete pod <pod-name>`

Note: generate a pod specification: `kubectl run db --image mongo:4.0 --dry-run=client -o yaml`
--dry-run simulates the resource creation with 2 possible options:

- client: resource not sent to server API
- server: resource set but not persisted to server API

## Scheduling

- selection of the node where a pod is deployed
- done by kube-scheduler component

Example:

```
kubect run www --image=nginx:1.16-alpine --restart=Never
kubectl describe pod www
# the output shows that the default scheduler assigned default/www to a node
```

Here the scheduler was able to select a node in our cluster. It is also possible to
add constraints to help the scheduler select a node

### nodeSelector: schedule a pod on a node with a specific label

```bash
# add label on a node
kubectl label nodes <node-name> disktype=ssd
# see node as yaml
kubectl get node/<node-name> -o yaml
# shows the labels in the outputs
```

- then in the file specification:

```yaml

..
kind: Pod
spec:
  containers:
    - ...
  nodeSelector:
    disktype: ssd
```

Now when creating the pod, the scheduler will only use nodes that have the correct label.
If no node correspond, it will fail

### nodeAffinity

- allows to schedule pods on certain nodes only
- applied on node labels
- more granular than nodeSelector
- List of Operators: `In`, `NotIn`, `Exists`, `DoesNotExit`, `Gt`, `Lt`
- different rules:
  - hard constraint, not deployed on failure: `requiredDuringSchedulingIgnoredDuringExecution`
  - soft constraint, lets selector decide which pod to use on failure: `preferredDuringSchedulingIgnoredDuringExecution`

File specification example:

```yaml
..
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/e2e-az-name
                operator: In
                values:
                  - e2e-az1
                  - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
        - preferencee:
          matchExpressions:
            - key: disktype
              operator: In
              values:
                - ssd
```

### podAffinity / podAntiAffinity

- allows to schedule pods based on labels of other pods
- different rules: `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution` (same as nodeAffinity)
- key `topologyKey` can be used to specify hostname, region, az, ... and specifies where the constraint must be applied

File specification example:

```yaml
..
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
         matchExpressions:
            - key: security
              operator: In
              values:
                - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity
      preferredDuringSchedulingIgnoredDuringExecution:
        - podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: security
                  operator: In
                  values:
                    - S2
            topologyKey: kubernetes.io/hostname
```

### resource allocation

in the pod specification

```yaml
..
spec:
  containers:
    - name: ..
      ..
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

### taints and toleration

A taint is added on a node and is repulsive. For a pod to be scheduled on this node,
it must tolerate this taint

```
# example: on a master node, there is a taint to prevent scheduling
$ kubectl get no master -o yaml
...
spec:
  taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
```

Now if I want to deploy a pod on this node, in the pod specification file, i put the
same key and same effect as the taint.

```yaml
..
spec:
  tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
```

# Services

- expose pods via network rules
- use labels to group pods
- persistant IP address (virtual IP address)
- kube-proxy in charge of laod balancing on pods
- different types:
  - ClusterIP: exposition inside cluster
  - NodePort: exposition to outside
  - LoadBalancer: integration with cloud provider
  - ExternalName: associates service to DNS name

## ClusterIP

allows to have interaction inside the cluster

```yaml
# pod.yaml - pod specification file
apiVersion: v1
kind: Pod
metadata:
  name: www
  labels:
    app: www
spec:
  containers:
    - name: nginx
      image: nginx:1.20-alpine
```

```yaml
# service.yaml - specification of the www service
apiVersion: v1
kind: Service
metadata:
  name: www
spec:
  selector:
    app: www
  type: ClusterIP
  ports:
    - port: 80 # service exposes port 80 in the cluster
      targetPort: 80 # requests forwarded to port 80 of pods
```

- creation of the pod: `kubectl apply -f pod.yaml`
- creation of the service: `kubectl apply -f service.yaml`
- see pods and service: `kubectl get po,svc`
- launch interactive shell and curl to port 80: `kubectl run -it --image:alpine` and in shell: `wget -O- http://www`

If we want to TEMPORARILY expose a port to host machine (outside the cluster):

- `kubectl port-forward svc/www 8080:80` then go to localhost: 8080
- OR `kubectl proxy`: then go to localhost:8001/api/v1/namespaces/default/services/www:80/proxy

## NodePort

Contrary to ClusterIP, exposes ports outside of cluster. Note that it also exposes inside the cluster
In fact, it can do what a ClusterIP can but can also open a port on each machine of the cluster so
that outside world can access the service
Note: port must be in a specified range (32000 - 32767 by default)

Example:

```yaml
# service.yaml - exactly same as above except type and ports
..
spec:
  ..
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 31000 # service is accessible at port 31000 for each node of the cluster
```

Now we can create the service (same as before)
To retrieve the external IP of one of the nodes, `kubectl get no -o wide`
We can then go to our browser to `<IP>:31000`

NOTE: with multipass, if external IP is not visible, IP can be retrieved with a `multipass list`

## LoadBalancer

Same as NodePort but allows to create a LoadBalancer which will be the entrypoint to
access the infrastructure. Instead of accessing directly the port of a given machine,
the traffic accesses the load balancer which is outside of the custer and will redirect
the request to one of the machines of the cluster.

```yaml
# service.yaml - exactly same as above except type and ports
..
spec:
  ..
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      # nodePort: 31000 # if not specified, will be set by k8s
```

Usually we use LoadBalancer with a cloud provider.
If I create a load balancer service, it will expose an external IP. Then I can go to port
80 of this IP and it will redirect to one of the machine of the cluster in the port
specified by nodePort

## Useful cmds

- launch from a speficiation file: `kubectl apply -f <service-spec.yaml>`

- create pod and expose it:

```bash
# create pod
kubectl run whoami --image containous/whoami
# Expose NodePort Service
kubectl expose pod whoami --type=NodePort --port=8080 --target-port=80
# service will have the same selector as pod
```

- create service directly: `kubectl create service nodeport whoami --tcp 8080:80` (selector will be app:whoami here)

- create pod and service which exposes it at once:
  `kubectl run db --image=mongo:4.2 --port=27017 --expose`

- same as pods, use `--dry-run=client -o yaml` to generate spec

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
