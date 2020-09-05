# Setup of a vanilla Kubernetes (k8s) cluster - on bare metal

## Pre-Requisites

1. Download Virtual Box - *https://www.virtualbox.org/wiki/Downloads*
2. Install Vagrant - *brew cask install vagrant*. This would help provisioning the VM infrastructure
3. Git CLI client - *https://git-scm.com/downloads* - for easy reference to sample repos

## Cluster Setup

1. Setup Vagrant - *Check Status, Install plugins etc. (if needed)*
2. Provision VMs for k8s cluster - *1 Master Node, 2 Worker Nodes*
3. Install *Kubeadm* on each *Node*
4. Install *Docker CE* on each *Node* 
5. Install kubelet and kubectl on each *Node* 
6. Initialize and Setup K8s Cluster on *Master Node*
7. Deploy *Weavenet* Pod Network to the Cluster
8. Join *Worker Nodes* into that cluster

## Application Deployment - Simple

1. Deploy sample *Nginx* service from Docker Hub and deploy with 1 replica

2. Check *Deployment* status; create a k8s *Service* for connecting for outside te *Pod*.

   (*Identify the issue in connecting to the Nginx application from out side the cluster*)

## Application Deployment - Advanced

- Create *Namespaces* for *DEV* and *QA* environments

- Deploy Microservivces using YAML file rather than command line

- Scale (out/in) Applications - *Manually*

- Install Helm package manager

- Install *Nginx* *Ingress* *Controller* along with *MetalLB*, providing Load Balancer solution for bare metsal scenario

- Create k8s Ingress object and route to Nginx service from outside

- Use Nginx Ingress Controller to route to appropriate services

- Network Policy to restrict traffic from/to desrired applications only

- Resource Quota to manage resourcer allocation avcross namespaces

- *Rolling Updates* and *Rollbacks*

  

## Hands-On Lab

### Cluster Setup

#### Setup Vagrant

- vagrant status
- vagrant plugin install vagrant-scp

#### Provision VMs for k8s cluster 

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

  - Deploy *Weavenet* Pod Network to the Cluster. Please note that you can deploy another supported Pod Network as well e.g. *Calico* (*https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises*)

    ```bash
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
    ```

  - Join *Worker Nodes* into that cluster

    ```bash
    Follow On-Screen Instructions
    ```

  #### Application Deployment - Simple

  - Deploy sample microservice

    ```bash
    kubectl create deploy nginx-deploy --image=nginx:latest --port=80 --replicas=1
    ```

  - Expose the microservice

    ```bash
    kubectl expose deploy nginx-deploy --name=nginx-svc --port=80 --type=NodePort
    ```

     

  ####  Application Deployment - Advanced

  - Create Namespaces for resource level seggregration

    ```bash
    kubectl create ns k8s-dev
    kubectl create ns k8s-qa
    ```

  - Deploy first microservice using YAML - *nginx-deploy-1.yaml*

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: nginx-deploy-1
      name: nginx-deploy-1
      namespace: k8s-dev
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx-deploy-1
      strategy:
       type: RollingUpdate
       rollingUpdate:
        maxSurge: 20%
        maxUnavailable: 1
      template:
        metadata:      
          labels:
            app: nginx-deploy-1
        spec:
          containers:
          - image: nginx:latest
            name: nginx
            ports:
            - containerPort: 80
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
              limits:
                cpu: 200m
                memory: 200Mi
    
    ```

  - Deploy Service object for first microservice using YAML - *nginx-svc-1.yaml*

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: nginx-deploy-1
      name: nginx-svc-1
      namespace: k8s-dev
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        app: nginx-deploy-1
      type: ClusterIP
    ```

  - Deploy second microservice using YAML - *nginx-deploy-2.yaml*

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: nginx-deploy-2
      name: nginx-deploy-2
      namespace: k8s-dev
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx-deploy-2
      strategy: {}
      template:
        metadata:      
          labels:
            app: nginx-deploy-2
        spec:
          containers:
          - image: nginx:latest
            name: nginx
            ports:
            - containerPort: 80
            resources:
              requests:
                cpu: 150m
                memory: 150Mi
              limits:
                cpu: 300m
                memory: 300Mi
    ```

  - Deploy Service object for first microservice using YAML - *nginx-svc-2.yaml*

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: nginx-deploy-2
      name: nginx-svc-2
      namespace: k8s-dev
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        app: nginx-deploy-2
      type: ClusterIP
    ```

  

  - Install *Helm*

    ```bash
    brew install helm - MacOS
    choco install kubernetes-helm - Windows
    sudo snap install helm --classic - Ubuntu
    
    https://helm.sh/docs/intro/install/#helm - Refer this for more options
    
    
    ```

  - Install *Nginx* *Ingress* *Controller* 

    ```bash
    # Create a namespace for your ingress resources
    kubectl create namespace ingress-basic
    
    # Add the ingress-nginx repository
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    
    # Use Helm to deploy an NGINX ingress controller
    helm install nginx-ingress ingress-nginx/ingress-nginx \
        --namespace ingress-basic \
        --set controller.replicaCount=2 \
        --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
        --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
    ```

  - Install *MetalB* as *Load Balancer*

    ```bash
    # Reference: https://metallb.universe.tf/installation/
    
    kubectl edit configmap -n kube-system kube-proxy
    
    Set the following as -
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    mode: "ipvs"
    ipvs:
      strictARP: true
      
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
    
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
    
    # On first install only
    kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
    ```

  - Prepare *MetalB* to select IP pool for load balancing and then privide the same for *Nginx Ingress Controller*

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      namespace: metallb-system
      name: config
    data:
      config: |
        address-pools:
        - name: metalb-pools
          protocol: layer2
          addresses:
          - 192.168.56.100-192.168.56.120
    ```

  - Create Ingress object for Routing

    ```YAML
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: k8s-ingress 
      namespace: k8s-dev 
      annotations:    
        kubernetes.io/ingress.class: nginx    
        nginx.ingress.kubernetes.io/rewrite-target: /$1
        nginx.ingress.kubernetes.io/ssl-redirect: "false"
        nginx.ingress.kubernetes.io/enable-cors: "true"    
    spec:  
      rules:    
      - http:
          paths:
          - path: /n1/?(.*)
            backend:
              serviceName: nginx-svc-1
              servicePort: 80
          - path: /n2/?(.*)
            backend:
              serviceName: nginx-svc-2
              servicePort: 80
    ```

  - Route to both microservices

    ```bash
    kubectl get ing -n k8s-dev
    
    NAME          CLASS    HOSTS   ADDRESS          PORTS   AGE
    k8s-ingress   <none>   *       192.168.56.100   80      5h17m
    
    curl http://192.168.56.100/n1
    curl http://192.168.56.100/n2
    ```

  - Expected Response

  ```html
  <!DOCTYPE html>
  <html>
  <head>
  <title>Welcome to nginx!</title>
  <style>
      body {
          width: 35em;
          margin: 0 auto;
          font-family: Tahoma, Verdana, Arial, sans-serif;
      }
  </style>
  </head>
  <body>
  <h1>Welcome to nginx!</h1>
  <p>If you see this page, the nginx web server is successfully installed and
  working. Further configuration is required.</p>
  
  <p>For online documentation and support please refer to
  <a href="http://nginx.org/">nginx.org</a>.<br/>
  Commercial support is available at
  <a href="http://nginx.com/">nginx.com</a>.</p>
  
  <p><em>Thank you for using nginx.</em></p>
  </body>
  </html>
  ```

-  *Network Policy* - Restrict traffic from intended workload only with the Cluster - *k8s-nwp.yaml*

  ```bash
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: k8s-network-policy
    namespace: k8s-dev
  spec:
    podSelector:
      matchLabels:
        app: nginx-deploy-2
    policyTypes:
    - Ingress
    ingress:
    - from:    
      - namespaceSelector:
          matchLabels:
            name: k8s-dev
      - podSelector:
          matchLabels:
            app: nginx-deploy-1
      ports:
      - protocol: TCP
        port: 80
  ```

  Try Previous URLs now and see the differnce

  ```bash
  curl http://192.168.56.100/n1 - Should still return correct response
  curl http://192.168.56.100/n2 - Should actually timeout as blocked by Network Policy
  ```

- Resouerce Quota - For restricting resource usage

  - Restricting No. of Objects created within Cluster - *k8s-dev-rq-oc.yaml*

    ```yaml
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: object-counts
      namespace: k8s-dev
    spec:
      hard:    
        pods: "4" 
        services: "2"
    ```

  - Restricting amount of Compute resources created within Cluster - *k8s-dev-rq-cr.yaml*

    ```yaml
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: compute-resources
      namespace: k8s-dev
    spec:
      hard:    
        requests.cpu: 300m
        requests.memory: 250Mi
        limits.cpu: 500m
        limits.memory: 500Mi
    ```

    

  





