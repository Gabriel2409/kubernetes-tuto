# Links

Certified Kubernetes Administrator: https://www.cncf.io/certification/cka/
Exam Curriculum (Topics): https://github.com/cncf/curriculum
Candidate Handbook: https://www.cncf.io/certification/candidate-handbook
Exam Tips: http://training.linuxfoundation.org/go//Important-Tips-CKA-CKAD

# Core concepts

## ETCD

Distributed, reliable key-value store that is Simple, Secure and Fast

install: https://etcd.io/docs/v3.5/install/

- start service with `etcd`: by default on port 2379
- use `etcdctl` to set and get values: `etcdctl put key1 value 1` and `etcdctl get key1`

## ETCD in kubernetes

- etcd datastore stores info regarding the cluster such as nodes, pods, configs, secrets, accounts, roles, bindings, ...
- each time we run `kubectl get ...`, the information comes to the etcd server
- each change we make in the cluster is updated in the etcd server.

If we manually setup the cluster from scratch:

- we must get the binaries ourselves
- we must then create the etcd service in the master node (there are a lot of options)

If we setup the cluster using kubeadm, kubeadm deploys the etcd-master pod in the kube-system namespace.
To explore the etcd database we can use the etcdctl utility from within the pod:
`kubectl exec etcd-master -n kube-system -- sh -c "ETCDCTL_API=3 etcdctl get / --prefix --keys-only --limit=10 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key"`
Note that we specified paths to certificates

## Kube-API server

primary management component in k8s

Example 1: `kubectl get nodes`

- the kubectl utility is reaching the kube-apiserver
- the kube-apiserver authenticates the request and validates it
- it then retrieves the data from etcd cluster
- and then it sends the response to the user

Example 2: `kubectl run mypod --image alpine`

- the kubectl utility is reaching the kube-apiserver
- the kube-apiserver authenticates the request and validates it
- the kube-apiser creates a pod object without assigning it to a node
- the kube-apiserver updates the etcd cluster
- the kube-apiserver updates the user that the pod has been created

- the kube-scheduler monitors the api-server and realises there is a new pod with no node assigned
- the kube-scheduler identifies the right node to place the pod on and communicates it to the kube-apiserver
- the kube-apiserver updates the information in the etcd cluster
- the kube-apiserver then passes this information to the kubelet in the appropriate node
- the kubelet creates the pod on the node and instructs the container runtime engine to deploy the image
- the kubelet then updates the status in the api-server
- the api-server then updates the etcd cluster

Under the hood, kubectl does an http request: you can see it when passing the verbose option to the kubectl cmd: `-v7`

To summarize, the kube-api server:

- authenticate users
- validate requests
- retrieve data
- update etcd (in fact only the api-server interacts with the etcd datastore)

Other components such as the scheduler or kubelet use the api-server to perform updates in the cluster

If we setup the cluster using kubeadm, kubeadm deploys the kube-apiserver-master as a pod in the kube-system namespace

## Controller manager

- A controller is a process that continuously monitors the state of elements in a system
- A controller works to remediate situations in case of problems
- For ex, the node controller controls the status of the nodes (by calling the api-server) and takes the necessary actions to keep the application running in case of problems on the nodes

  - Controls every 5s (Node monitor period)
  - In case of failure to reach node, it waits for 40s (Node Monitor Grace period) and then marks it as unreachable
  - After a node is marked as unreachable, it gives it 5 mins to come back up (POD Eviction Timeout)
  - If it is still unreachable, it removes the pods assigned to the node and provisions them on healthy ones if they are part of a replicaset

- Other ex: the replication controller

  - is responsible for controlling the state of the replicasets.
  - It ensures that the correct nb of pods is available as part of the replicaset. If a pod dies, it creates another one.

- There are a lot of controllers in kubernetes and the full behavior of the cluster is implemented through these controllers

All of the controllers are packaged in a single process: the `kube-controller-manager`

If we setup the cluster using kubeadm, kubeadm deploys the kube-controller-manager-master as a pod in the kube-system namespace
