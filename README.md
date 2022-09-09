# kubectl tricks

## Basics

- install autocompletion for kubectl: `sudo apt install bash-completion` and `source <(kubectl completion bash)`

- Two approaches:

  - imperative approach: handle resources in cli:
    - create pod: `kubectl run <podname> --image <imagename> OPTIONS`
    - create any resource: `kubectl create <resourcetype> <resourcename> OPTIONS`
    - list resource: `kubectl get <resource>`
    - delete resource: `kubectl delete <resource>`
  - declarative: use a spec:
    - create resources: `kubectl apply -f <specification-file-or-folder>`
    - delete resources: `kubectl delete -f <specification-file-or-folder>`

- Possibility to add flag `--dry-run=client` or `--dry-run=server` to prevent creation of resource (if server is specified, request is passed to server API)
- See output with `-o yaml`, `-o json`, or `-o jsonpath` (can be used with `dry-run` to see resulting spec)

- describe a resource: `kubectl describe <resourcetype> <resourcename>`
- see all api resources and their abbreviation: `kubectl api-resources`
- infos about commands: `kubectl explain <resourcename>` (can be nested, for ex `kubectl explain pod.metadata.uid`)

## Cool tricks

- get image of pod www: `kubectl get po/www -o jsonpath='{.spec.containers[0].image}'`
- get image id of all pods in kube-system namespace: `kubectl get po -n kube-system -o jsonpath='{range.items[*]}{.status.containerStatuses[0].imageID}{"\n"}{end}'`
- custom columns: `kubectl get po -o custom-columns='NAME:metadata.name,IMAGES:spec.containers[*].image'`

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

- switch context:
- kubectl: `kubectl config use-context <context-name>`
- plugin: `kubectl ctx <context-name>` (replace cmd with `kubectx` if not installed with Krew)

- switch namespace:
  - kubectl: `kubectl config set-context $(kubectl config current-context) --namespace=<name>`
  - plugin: `kubectl ns <name>` (replace cmd with `kubens` if not installed with Krew)

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
- switch context: `kubectl config use-context k3d-k3s`
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
- launch a pod directly from an image: `kubect run <pod-name> --image <image-name>`
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
kubect run www --image nginx:1.16-alpine --restart=Never
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
    podAntiAffinity:
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

A pod inside a node can interact with the service and the service will forward
to relevant pods inside the cluster (even if not on same node)

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
- launch interactive shell and curl to port 80: `kubectl run debug -it --image alpine` and in shell: `wget -O- http://www`

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
  `kubectl run db --image mongo:4.2 --port=27017 --expose`

- same as pods, use `--dry-run=client -o yaml` to generate spec

# Deployment

## Creation

A deployment is a more high level resource that deals with an ensemble of identical pods
and makes sure that there is a specific number of nodes that run in the cluster
If one of the replicas crash, the deployment can restart one.
A deployment deals with pod lifecycle:

- creation / deletion
- scaling
- rollout / rollback

ReplicaSet = An intermediate resource between deployment and pod whose role is to make
sure pod is working as intended and takes corrective measures if needed
Note that we rarely manipulate ReplicaSet directly

example:

```yaml
# deployment

apiVersion: apps/v1 # version is apps/v1 and not just v1
kind: Deployment
metadata:
  name: www
spec:
  replicas: 5
  selector:
    matchLabels:
      app: www # selector must match pod label

  template: # contains the pod specification
    metadata:
      labels:
        app: www
    spec:
      containers:
        - name: www
          image: nginx:1.16
```

- run from spec: `kubectl apply -f <deploy-spec.yaml>`
- imperative cmd: `kubectl create deploy <name> --image <image>`
  Note: `--dry-run=client -o yaml` to have specification as usual

## Update

- update strategy. By default k8s uses a rolling update strategy. We can specify
  extra parameters in the spec.

```yaml
# deployment
..
spec:
  ..
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1 # how many extra replicas I can have
      maxUnavailable: 0 # how many replicas can be removed at most
..
```

In the example above, for the update, we start with 3 nodes at v1, then we add a node at v2,
then remove a v1, ... until we only have 3 v2s.
The rolling update will be triggered each time the pod specification in the deployment is
modified. So for example, we modify the spec and run a `kubectl apply -f <deploy_spec>`.
k8s will then run the rolling update process

- we can also update a deployment imperatively: `kubectl set image deploy/www www=nginx:1.18 --record`
  (here the deployment is called www and the container to modify is called www).

- list history of deployment: `kubectl rollout history deploy/www`:
  - CHANGE-CAUSE column contains the command that brought to current version if `--record` flag was set when doing the update
  - 10 revisions by default -> modified by property `.spec.revisionHistoryLimit` of deployment
- revert: `kubectl rollout undo deploy/www` with optional `--to-revision=X`

- we can also force an update with `kubectl rollout restart deploy/www`: the pods of the deployments are stopped and new ones are launched

## Scaling

- scale a deployment: `kubectl scale deploy <name> --replicas=3`

We can also use a HPA
A HorizontalPodAutoscaler (HPA for short) automatically updates a workload resource
(such as a Deployment or StatefulSet)

from a spec:

```yaml
# hpa-spec.yaml

apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: www
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: www
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

and `kubectl apply -f <hpa-spec.yaml>`

or imperatively: `kubectl autoscale deploy www --min=2 --max=10 --cpu-percent=50`

NOTE: we will have an error if the deployment does not exist

- we can access the hpa with `kubectl get hpa`

- Note that for HPA to work correctly:
  - a service exposing the pods must be running
  - resources must be specified in the container spec (not sure about that)
  - metrics server must be installed:
    - check that it is installed: `kubectl get po -n kube-system -l k8s-app=metrics-server`
    - install it if necessary: `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`
    - note that for minikube, you must do `minikube addons enable metrics-server` instead

# Namespaces

- Namespaces are scopes for pods, services, deployments,...
- allows to share a cluster (for ex team, project, client, ...)
- 3 namespaces by default: default, kube-public and kube-system
- resources are created in default if non specified
- see namespaces: `kubectl get namespace`
- create namespace directly: `kubectl create namespace <name>`
- create namespace from spec: `kubectl create -f <namespacename.yaml>`
- delete namespace: `kubectl delete namespace <name>`

example spec:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    name: development
```

- to put a pod in a namespace:

  - method 1: in the specification:

  ```yaml
  ..
  metadata:
    ..
    namespace: <namespace_name>
  ```

  - method 2: `kubectl run <pod_name> --namespace <namespace_name> --image <image_name>`

Now, when we list the pods, to see it: `kubectl get po --namespace=<namespace_name>` or `kubectl get-po --all-namespaces`
If namespace is not specified, default is used

Note that we can specify namespaces for other resources such as deployments.

- quotas: `kubectl apply -f <quota.yaml> --namespace <namespacename>`
- get the resource quotas: `kubectl get resourcequota`

```yaml
#quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
spec:
  hard:
    requests.cpu: "1" #cannot ask for more than 1 cpu
    requests.memory: 1Gi
    limits.cpu: "2" #cannot use more than 2 cpu
    limits.memory: 2Gi
```

- limitRange: `kubectl apply -f <limitrange.yaml> --namespace <namespacename>`

```yaml
#limitrange.yaml

apiVersion: v1
kind: LimitRange
metadata:
  name: memory-limit-range
spec:
  limits:
    - default:
        memory: 128M
      defaultRequest:
        memory: 64M
      max:
        memory: 256M
      type: Container
```

- Now when creating a pod,
  - if resource and requests are not specified, they will use the limitRange default
  - if specified and correct, will use, will use the ones specified
  - if specified but outside limitRange, will throw error

Trick with context:

- To get the current client context: `kubectl config current-context`
- to modify the default namespace in current context: `kubectl config set-context $(kubectl config current-context) --namespace=development`
- check changes: `kubectl config view`
  Now if we create a new resource without specifying the namespace, it will go to the namespace development

# ConfigMap

- Allows to decouple configuration from application specification

- Create from file: `kubectl create configmap proxy-config --from-file=./nginx.conf`
- in pod specification:

```yaml
spec:
  containers:
    - name: proxy
      image: nginx:1.20-alpine
      volumeMounts:
        - name: config
          mountPath: "/etc/nginx/"
  volumes:
    - name: config
      configMap:
        name: proxy-config
```

Here: the pod will use the config-map that was created from a file (can be seen when describing the pod)

- config map can also be created from env-file or from litteral values

config map can also be used directly in containers:

```yaml
# pod spec
..
spec:
  containers:
    - name: ..
      ..
      valueFrom:
        configMapKeyRef:
          name: app-config-list # name of configmap
          key: log_level # gets env variable from configmap key
```

or if you want to load the full config

```yaml
# in the container spec
envFrom:
  - configMapRef:
    name: <configmap-name>
```

- as usual, you can specify the specification and use `kubectl apply -f configmap.yaml`

NOTE: if a deployment uses a configmap and the file configmap spec is changed, it will NOT
automatically update the deployment when running `kubectl apply -f deploy.yaml`.
https://stackoverflow.com/questions/37317003/restart-pods-when-configmap-updates-in-kubernetes
A possibility is to add a config hash in the deployment spec:

```yaml
..
spec:
  ..
  template:
    ..
    spec:
      containers:
      - name: nginx
        image: nginx:1.14-alpine
        volumeMounts:
        - name: config
          mountPath: "/etc/nginx/"
        env:
        - name: CONFIG_HASH
          value: ${CONFIG_HASH}
```

- now when configmap is modified:
  - `export CONFIG_HASH=$(kubectl get cm -oyaml | sha256sum | cut -d' ' -f1)`
  - `envsubst '${CONFIG_HASH}' < deploy.yaml`
  - then when doing `kubectly apply -f deploy.yaml`, the change will be reflected

# Secrets

- almost the same as config map but used for sensitive information

## generic

- first possibility: from file

```bash
echo -n "admin" > ./username.txt
echo -n "password" > ./password.txt
kubectl create secret generic service-creds --from-file=./username.txt --from-file=./password.txt
```

- directly:`kubectl create secret generic service-creds --from-literal=mysecret=1234`
- list them with `kubectl get secrets`

- as usual, possible to specify a yaml instead (see output of dry-run)

Secret can then be used in pods similarly to config map (the key name change, for ex `secretKeyRef`)

## docker-registry

used to identify on a docker registry:
`kubectl create secret docker-registry registry-creds --docker-server=REGISTRY --docker-username=USERNAME --docker-password=PASSWORD --docker-email=EMAIL`

- when doing `kubectl get secret registry-creds -o yaml`, in the data key, we see `.dockerconfigjson` with creds encoded in b64

## tls

- use openssl to create a couple public private key: `openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out cert.pem`
- create secret: `kubectl create secret tls domain-pki --cert cert.pem --key key.pem`
- When we do `kubectel get secret domain-pki -o yaml`, we can see tls.crt and tls.key are here (encoded in b64)

- use the secret in a pod, in the spec of the pod:

```yaml
# use a secret as volume
..
spec:
  containers:
  - name: proxy
    image: nginx:1.12.2
    volumeMounts:
      - name: tls
        mountPath: "/etc/ssl/certs"
  volumes:
    - name: tls
      secret:
        secretName: domain-pki
```

```yaml
# use a secret by getting the key
..
spec:
  env:
    - name: MONGODB_URL
      valueFrom:
        secretKeyRef:
          name: mongo
          key: mongo_url
```

# RBAC (Role Based Access Control)

## Authentication

To authenticate a user, you must use several methods outside of kubernetes and then
approve them inside the cluster, for ex:

- client-certificate: `--client-ca-file=FILE`
- bearer tokens: `--token-auth-file=FILE`
- HTTP basic auth: `--basic-auth-file=FILE`

To authenticate a process inside the cluster, we use a ServiceAccount:

- gives rights to containers in a pod
- possibility to generate a JWT token (see section on Web interface)
- by default, pods use the ServiceAccount called default

## Authorization

- Being authenticated is not enough to be granted permissions.
- Each ServiceAccount/User must then be linked to a Role via a RoleBinding or a ClusterRole via a ClusterRoleBinding
- RoleBindings are limited to one namespace, ClusterRoleBindings apply to the whole custer
- Roles and ClusterRoles grant access to resources with high granularity.
  - For ex a ServiceAccount can be authorized to list pods but not delete them

Note :To see which clusterroles a user / service account has access to, you can list all the
clusterrolebindings and search for the name associated to the user / service account: `kubectl get customrolebindings -o yaml | less -p 'name: <name>'`
I did not find a better way

## Example

- Create a token associated to sa default: `kubectl create token default` (visualise it at https://jwt.io,)
- To retrieve the token, we must access it from a pod that uses this service account (see below)
- Run a debug container: `kubectl run debug -it --image alpine`
- Inside the container:

  - install curl `apk add --update curl`
  - try to access kubernetes server API: `curl https://kubernetes/api/v1 --insecure` => we get an error
  - retrieve TOKEN here: `TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)`
  - try again: `curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/ --insecure` => it works
  - try to access the list of pods: `curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/default/pods/ --insecure` => forbidden, we don't have enough permissions

- Create a new service account, a new role and a new role binding

```yaml
# demo-sa.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-sa

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: list-pods
  namespace: default
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - list

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: list-pods_demo-sa
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: list-pods
subjects:
  - kind: ServiceAccount
    name: demo-sa
    namespace: default
```

- run a basic alpine pod (specify serviceAccountName: demo-sa in spec) and do the
  same steps as above. Listing pods is now possible

# Web interface

If on minikube:

- enable dashboard: `minikube addons enable dashboard`
- launch dashboard: `minikube dashboard`

If not on minikube:

- `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml`
- `kubectl proxy` then go to `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`
  Then we need to login, either by uploading the kubeconfig, copy the token from kubeconfig or create an admin token with authentication rights
  To do so, we create a Service account and a CLusterRoleBinding to give it admin rights. We also create a secret

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kube-system
---
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: admin-user
type: kubernetes.io/service-account-token
```

and we get the token with: `echo $(kubectl -n kube-system get secret admin-user -o jsonpath='{.data.token}' | base64 --decode)`

- Note: instead of creating the secret (which creates a non expiring token), we can also do: `kubectl -n kube-system create token admin-user`
  and retrieve the token. If we use this method without binding the token to a secret, see prev section to retrieve the token

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
