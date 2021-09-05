# NVIDIA operator on Kubernetes with Kerberos Vault

Machine learning using the NVIDIA GPU operator with Kerberos Vault on Kubernetes. Integrate and scale your Machine learning using Kerberos Vault and  a GPU node-based Kubernetes Cluster.

In this example we will show you how you can use [Kerberos Enterprise Agents](https://doc.kerberos.io/enterprise/first-things-first/) and [Kerberos Vault](https://doc.kerberos.io/vault/first-things-first/) to scale your machine learning and video surveillance or analytics landscape. By decoupling your cameras and GPU's using the Kubernetes platform and Kerberos Enterprise you bring real scale into the picture.

![NVIDIA operator Kerberos Vault](https://user-images.githubusercontent.com/1546779/132137679-33fc02df-085f-47cf-8587-301bd3448e63.png)

The following example will show you how to setup a node with one or more GPU's in a Kubernetes Cluster. Afterwards we will deploy a machine learning workload that can recognise pedestrians in one or more recordings. To handle that execution we have a couple of cameras in place (called Kerberos agents) and our open/extensible storage platform called [Kerberos Vault](https://doc.kerberos.io/vault/first-things-first/).

![NVIDIA integration with kafka](https://user-images.githubusercontent.com/1546779/132138400-61f6b2a3-04e2-461f-aee7-18d8bc28a3e8.png)

[Kerberos Vault](https://doc.kerberos.io/vault/first-things-first/) receives recordings from one or more (or thousands) of Kerberos agents, and will trigger events through integrations such as Kafka, SQS, etc. Everytime a recording is stored in [Kerberos Vault](https://doc.kerberos.io/vault/first-things-first/), a real-time message is generated, and a consumer (the workload we have deployed in our cluster) will download the recording and start the interference on one of you GPU based Kubernetes nodes (using the NVIDIA operator). 

# Prepare a node to run GPU based deployments

![NV-GPU-Operator-1](https://user-images.githubusercontent.com/1546779/132136901-44d90617-a80a-4933-9eca-bf965622d237.png)

To provision GPU worker nodes in a Kubernetes cluster, the following NVIDIA software components are required – the driver, container runtime, device plugin and monitoring. As shown in Figure 1, these components need to be manually provisioned before GPU resources are available to the cluster and also need to be managed during the operation of the cluster. The GPU Operator simplifies both the initial deployment and management of the components by containerizing all the components and using standard Kubernetes APIs for automating and managing these components including versioning and upgrades. The GPU operator is fully open-source and is available [at the NVIDIA GitHub repo](https://github.com/NVIDIA/gpu-operator).

![GPU-Operator-Manual-Install-Figure](https://user-images.githubusercontent.com/1546779/132136925-7f7a2c88-7d58-41ba-8b8f-8f72b0af82de.png)


## NVidia Drivers
We are assuming an Ubuntu 20.4 system with a clean installation, and first go ahead with installating the NVIDIA drivers and CUDA drivers.

    sudo -s
    apt install nvidia-driver-455
    reboot
    
    sudo -s
    nvidia-smi
    apt install nvidia-cuda-toolkit

## Setup tools

First we have to install Kubernetes (we might also use K3S) on a base image. 

### Install Docker

Let's install Docker. We could also use `containerd` with Kubernetes.

    apt install docker.io -y    

### Install Kubernetes
  
Install the Kubernetes toolset.

    apt update -y
    apt install apt-transport-https curl -y
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add
    apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
    apt update -y && apt install kubeadm kubelet kubectl kubernetes-cni -y

Disable swap as this is required by Kubernetes.

    swapoff -a
    sudo sed -i.bak '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

Initiate the cluster.

    kubeadm init

This might take a couple of minutes but once finished you should see following message.

    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 192.168.1.103:6443 --token ej7ckt.uof7o2iplqf0r2up \
        --discovery-token-ca-cert-hash sha256:9cbcc00d34be2dbd605174802d9e52fbcdd617324c237bf58767b369fa586209
  
      
### Install Nivida Docker

To enable NVidia on Kubernetes a couple of things will need to be changes (more info here 
https://github.com/NVIDIA/k8s-device-plugin).

    distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
       && curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
       && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
       
       apt-get update
       apt-get install -y nvidia-docker2
       systemctl restart docker
       docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
   
Make an additional modification to the `daemon.json` of Docker.

        nano /etc/docker/daemon.json
        
Make sure the `.json` file is aligned with below config.

    {
        "default-runtime": "nvidia",
        "runtimes": {
            "nvidia": {
                "path": "/usr/bin/nvidia-container-runtime",
                "runtimeArgs": []
            }
        }
    }

## Enable NVidia k8s plugin
  
    kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.9.0/nvidia-device-plugin.yml

## Create a GPU workload and scale with Kerberos Vault

When creating a new pod or deployment, you assign a number of GPU's to the workload, this will make sure the workload is scheduled on a node which has one or more GPUs available.

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: vault-ml
      labels:
        app: vault-ml
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: vault-ml
      template:
        metadata:
          labels:
            app: vault-ml
        spec:
          containers:
            - name: kerberoshub-ml
              image: kerberos/vault-ml-gpu
              resources:
                limits:
                  nvidia.com/gpu: 1 # requesting a single GPU
              env:
                - name: QUEUE_SYSTEM
                  value: "KAFKA"
                - name: QUEUE_NAME
                  value: "kubeflow"
                - name: QUEUE_TARGET
                  value: "footages"
                - name: KAFKA_BROKER
                  value: "pkc-ldjyd.southamerica:9092"
                - name: KAFKA_GROUP
                  value: "group"
                - name: KAFKA_USERNAME
                  value: "xxx"
                - name: KAFKA_PASSWORD
                  value: "xxx"
                - name: KAFKA_MECHANISM
                  value: "PLAIN"
                - name: KAFKA_SECURITY
                  value: "SASL_SSL"
                - name: VAULT_API_URL
                  value: "https://staging.api.vault.kerberos.live"
                - name: VAULT_ACCESS_KEY
                  value: "xxx"
                - name: VAULT_SECRET_KEY
                  value: "xxx"



