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

## Kubernetes
First we have to install Kubernetes (we might also use K3S) on a base image. 

Let's install Docker. We could also use `containerd` with Kubernetes.

    apt install docker.io -y
  
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
  
