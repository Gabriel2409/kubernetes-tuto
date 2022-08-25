# Installation

- Install kubectl = kubernetes cli

```bash
VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
curl -LO https://storage.googleapis.com/kubernetes-release/release/$VERSION/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

- Install multipass: https://multipass.run/ : allows to easily launch a VM with either hyperV or virtual box
  To use it on WSL, it must be installed on windows, not on linux. And then, we can create
  an alias to use it directly on WSL: for ex: `alias multipass="/mnt/c/'Program Files'/Multipass/bin/multipass.exe"`

- create a vm:
  - basic : `multipass launch node1`
  - with parameters: ` multipass launch -n node2 -c 2 -m 3G -d 10G` (2 cores, 3G ram, 10G disk size)
- get infos: `multipass infos node2`
- list vms: `multipass list`
- launch a shell: `multipass shell node1`
- install docker: `multipass exec node1 -- /bin/bash -c "curl -sSL https://get.docker.com | sh"` (here, it launches the bash shell and execute the command between quotes. It is the same as getting into the sell and executing the command between quotes)
- check that docker is correctly installed: `multipass exec node1 -- sudo docker version`

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
