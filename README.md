# Setup of a vanilla Kubernetes (k8s) cluster - on bare metal



## Pre-Requisites

1. Download Virtual Box - *https://www.virtualbox.org/wiki/Downloads*
2. Install Vagrant - *brew cask install vagrant*. This would help provisioning the VM infrastructure
3. Git CLI client - *https://git-scm.com/downloads* - for easy reference to sample repos



## Execution Steps

1. Setup Vagrant - *Check Status, Install plugins etc. (if needed)*

2. Provision VMs for k8s cluster - *1 Master Node, 2 Worker Nodes*

3. Install *Kubeadm* on each *Node*

4. Install *Docker CE* on each *Node* 

5. Install kubelet and kubectl on each *Node* 

6. Initialize and Setup K8s Cluster on *Master Node*

7. Join *Worker Nodes* into that cluster

   

### Setup Vagrant

- vagrant status
- vagrant plugin install vagrant-scp

### Provision VMs for k8s cluster 

- vagrant up

- vagrant ssh <machine_name> // kubemaster, kubenode01, kubenode02

- Install *Kubeadm* on each *Node* - 

  (Follow Instructions at - https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

  - Make sure that the *br_netfilter*  module is loaded - *lsmod | grep br_netfilter*

  - To load it explicitly call - *sudo modprobe br_netfilter*

  - Ensure *net.bridge.bridge-nf-call-iptables* is set to 1 in your *sysctl* config

    ```bash
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sudo sysctl --system
    ```

  - Switch to root privileges - *sudo -i*

  - Install *Docker CE* on each *Node* 

    (Follow Instructions at - *https://kubernetes.io/docs/setup/production-environment/container-runtimes/*)

    ```bash
    # (Install Docker CE)
    ## Set up the repository:
    ### Install packages to allow apt to use a repository over HTTPS
    apt-get update && apt-get install -y \
      apt-transport-https ca-certificates curl software-properties-common gnupg2
    ```

    ```bash
    # Add Docker's official GPG key:
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    ```

    ```bash
    # Add the Docker apt repository:
    add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) \
      stable"
    ```

    ```bash
    # Install Docker CE
    apt-get update && apt-get install -y \
      containerd.io=1.2.13-2 \
      docker-ce=5:19.03.11~3-0~ubuntu-$(lsb_release -cs) \
      docker-ce-cli=5:19.03.11~3-0~ubuntu-$(lsb_release -cs)
    ```

    ```bash
    # Set up the Docker daemon
    cat > /etc/docker/daemon.json <<EOF
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }
    EOF
    ```

    ```bash
    mkdir -p /etc/systemd/system/docker.service.d
    ```

    ```bash
    # Restart Docker
    systemctl daemon-reload
    systemctl restart docker
    
    If you want the docker service to start on boot, run the following command:
    
    sudo systemctl enable docker
    
    systemctl status docker
    docker images
    docker ps -a
    ```

  - Install *kubelet* and *kubectl* on each *Node*

    ```bash
    sudo apt-get update && sudo apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

    ```bash
    kubeadm version
    kubectl --version
    kubectl version
    ```

  - Initialize and Setup K8s Cluster on *Master Node*

    ```bash
    vagrant ssh kubemaster
    kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.56.2
    ```

  - Join *Worker Nodes* into that cluster

    ```bash
    Follow On-Screen Instructions
    ```

  - Deploy sample microservice

    ```bash
    kubectl create deploy nginx-deploy --image=nginx:latest --port=80 --replicas=1
    ```

  - Expose the microservice

    ```bash
    kubectl expose deploy nginx-deploy --name=nginx-svc --port=80
    ```

    





