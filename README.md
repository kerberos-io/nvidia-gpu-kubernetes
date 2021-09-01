# Kubeflow

Machine learning with Kerberos on Kubernetes. A Kubeflow example that integrates with Kerberos Vault.

## NVidia Drivers
We are assuming an Ubuntu 20.4 system with a clean installation, and first go ahead with installating the NVIDIA drivers and CUDA drivers. Check with @Pedro, but do not believe we need to have the CUDA drivers on the host system, they should go into the kubeflow deployments.

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

## Create a GPU workload (pod/deployment)

When creating a new pod or deployment, you assign a number of GPU's to the workload, this will make sure the workload is scheduled on a node which has one or more GPUs available.

    apiVersion: v1
    kind: Pod
    metadata:
      name: kerberoshub-ml
    spec:
      containers:
        - name: kerberoshub-ml
          image: kerberos/yolo34py-gpu
          resources:
            limits:
              nvidia.com/gpu: 1 # requesting 2 GPUs



