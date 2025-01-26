# NVIDIA operator on Kubernetes with Kerberos Vault

> This project is archived, and move to the new organisation and repository: [uug-ai/hub-pipeline-classifier](https://github.com/uug-ai/hub-pipeline-classifier)

Machine learning using the NVIDIA GPU operator with Kerberos Vault on Kubernetes. Integrate and scale your Machine learning using Kerberos Vault and  a GPU node-based Kubernetes Cluster.

In this example we will show you how you can use [Kerberos Agents](https://doc.kerberos.io/enterprise/first-things-first/) and [Kerberos Vault](https://doc.kerberos.io/vault/first-things-first/) to scale your machine learning and video surveillance or analytics landscape. By decoupling your cameras and GPU's using the Kubernetes platform and Kerberos Enterprise you bring real scale into the picture.

![NVIDIA operator Kerberos Vault](https://user-images.githubusercontent.com/1546779/132137679-33fc02df-085f-47cf-8587-301bd3448e63.png)

The following example will show you how to setup a node with one or more GPU's in a Kubernetes Cluster. Afterwards we will deploy a machine learning workload that can recognise pedestrians in one or more recordings. To handle that execution we have a couple of cameras in place (called Kerberos agents) and our open/extensible storage platform called [Kerberos Vault](https://doc.kerberos.io/vault/first-things-first/).

![NVIDIA integration with kafka](https://user-images.githubusercontent.com/1546779/132138400-61f6b2a3-04e2-461f-aee7-18d8bc28a3e8.png)

[Kerberos Vault](https://doc.kerberos.io/vault/first-things-first/) receives recordings from one or more (or thousands) of Kerberos agents, and will trigger events through integrations such as Kafka, SQS, etc. Everytime a recording is stored in [Kerberos Vault](https://doc.kerberos.io/vault/first-things-first/), a real-time message is generated, and a consumer (the workload we have deployed in our cluster) will download the recording and start the interference on one of you GPU based Kubernetes nodes (using the NVIDIA operator). 

# Prepare a node to run GPU based deployments

![NV-GPU-Operator-1](https://user-images.githubusercontent.com/1546779/132136901-44d90617-a80a-4933-9eca-bf965622d237.png)

To provision GPU worker nodes in a Kubernetes cluster, the following NVIDIA software components are required – the driver, container runtime, device plugin and monitoring. As shown in Figure 1, these components need to be manually provisioned before GPU resources are available to the cluster and also need to be managed during the operation of the cluster. The GPU Operator simplifies both the initial deployment and management of the components by containerizing all the components and using standard Kubernetes APIs for automating and managing these components including versioning and upgrades. The GPU operator is fully open-source and is available [at the NVIDIA GitHub repo](https://github.com/NVIDIA/gpu-operator).

![GPU-Operator-Manual-Install-Figure](https://user-images.githubusercontent.com/1546779/132136925-7f7a2c88-7d58-41ba-8b8f-8f72b0af82de.png)

## NVidia Drivers
We are assuming an Ubuntu 20.4 system with a clean installation. First things first, let's go ahead with installing the NVIDIA drivers and CUDA drivers.

    sudo -s
    apt install nvidia-driver-455
    reboot
    
    sudo -s
    apt install nvidia-cuda-toolkit
    apt install nvidia-utils-455
    nvidia-smi

## Setup tools

Once we have the NVIDIA drivers installed, we are ready to setup Docker and Kubernetes. Next to that we will enable NVIDIA for Docker and later on we will install the NVIDIA Kubernetes operator.

### Install Docker

Let's install Docker. We could also use `containerd` with Kubernetes.

    apt install docker.io -y    
    
Once installed modify the cgroup driver, so kubernetes will be using it correctly. By default Kubernetes cgroup driver was set to systems but docker was set to systemd.

    sudo mkdir /etc/docker
    cat <<EOF | sudo tee /etc/docker/daemon.json
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }
    EOF

    sudo systemctl enable docker
    sudo systemctl daemon-reload
    sudo systemctl restart docker


### Install Kubernetes
  
Install the Kubernetes toolset.

    apt update -y
    apt install apt-transport-https curl -y
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    apt update -y
    apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1

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
  
      
### Install NVIDIA for Docker

To enable NVIDIA for Docker, [a couple of things will need to be installed](https://github.com/NVIDIA/k8s-device-plugin).

    distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
       && curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
       && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
       
       apt-get update
       apt-get install -y nvidia-docker2
       systemctl restart docker
       docker run --rm --gpus all nvidia/cuda:11.1.1-base-ubi8 nvidia-smi
   
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
        },
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "100m"
        },
        "storage-driver": "overlay2"
    }

Restart the Docker daemon to complete the installation after setting the default runtime:

    sudo systemctl restart docker

### or Install NVIDIA for containerd

Update containerd to use nvidia as the default runtime and add nvidia runtime configuration. This can be done by adding below config to `/etc/containerd/config.toml` and restarting containerd service.

    version = 2
    [plugins]
      [plugins."io.containerd.grpc.v1.cri"]
        [plugins."io.containerd.grpc.v1.cri".containerd]
          default_runtime_name = "nvidia"
    
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
            [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
              privileged_without_host_devices = false
              runtime_engine = ""
              runtime_root = ""
              runtime_type = "io.containerd.runc.v2"
              [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
                BinaryName = "/usr/bin/nvidia-container-runtime"

Restart the Containerd daemon to complete the installation after setting the default runtime:

    sudo systemctl restart containerd

## Enable NVidia GPU operator

> Find the full tutorial on [the official NVIDIA docs page](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html).

Download the NVIDIA helm chart

    helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update

Install the NVIDIA helm chart in the `gpu-operator` namespace.

    helm install --wait --generate-name \
    -n gpu-operator --create-namespace \
    nvidia/gpu-operator

## Create a GPU workload and scale with Kerberos Vault

When creating a new pod or deployment, you assign a number of GPUs to the workload, this will make sure the workload is scheduled on a node which has one or more GPUs available. Magic, all done by the NVIDIA Kubernetes operator. So the conclusion is that you can add as much nodes and GPUs you want, and you can simply increase the `replicas: 1` parameter to the number of GPUs you have available.

![Integration in practice (Yolov3)](https://user-images.githubusercontent.com/1546779/133740877-a0bc2261-57ad-47b7-ab01-a929329b729a.png)

Once you have created below deployment in your Kubernetes cluster, you will have one or more machine learning workloads integrated with your Kerberos Vault and Kafka broker. Due to the nature of Kafka, and how we designed the Kerberos Enterprise suite, it will also loadbalance or divide and concur the request over your different GPU's. Have some fun ;)

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
              image: kerberos/vault-ml:nvidia
              resources:
                limits:
                  nvidia.com/gpu: 1 # requesting a single GPU
              env:
                - name: QUEUE_SYSTEM
                  value: "KAFKA"
                - name: QUEUE_NAME
                  value: "source_topic" # This is the topic of kafka we will read messages from.
                - name: QUEUE_TARGET
                  value: "target_topic" # Once we processed the recording with ML, we will send results/metadata to a target topic of Kafka.
                - name: KAFKA_BROKER
                  value: "xxx-your-kafka-xxx:9092"
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
                  value: "https://xxx.api.vault.kerberos.live"
                - name: VAULT_ACCESS_KEY
                  value: "xxx"
                - name: VAULT_SECRET_KEY
                  value: "xxx"
                - name: NUMBER_OF_PREDICTIONS
                  value: "5"


The results you will show when inspect the logs of `vault-ml` is:

    {"date": 1630643468, "data": {"probabilities": [[0.5907418131828308], [0.7311708927154541], [0.555280864238739], [0.5144052505493164]], "labels": [["car"], ["truck"], ["truck"], ["car"]], "boxes": [[[298, 36, 398, 79]], [[514, 53, 656, 129]], [[514, 53, 656, 127]], [[315, 101, 351, 125]]]}, "operation": "classification", "events": ["monitor", "sequence", "analysis", "throttler", "notification"], "provider": "kstorage", "request": "persist", "payload": {"key": "youruser/1630643468_6-967003_highway4_200-200-400-400_24_769.mp4", "fileSize": 4545863, "is_fragmented": false, "metadata": {"uploadtime": "1630643468", "event-instancename": "highway4", "event-timestamp": "1630643468", "productid": "Bfuk14xm40eMSxwEEyrd908yzmDIwKp5", "event-numberofchanges": "24", "event-microseconds": "0", "event-regioncoordinates": "200-200-400-400", "capture": "IPCamera", "event-token": "0", "publickey": "ABCDEFGHI!@#$%12345"}, "bytes_ranges": "", "bytes_range_on_time": null}, "source": "storj"}
    next..
    checking..
    [{'date': 1630643550, 'events': ['monitor', 'sequence', 'analysis', 'throttler', 'notification'], 'provider': 'kstorage', 'request': 'persist', 'payload': {'key': 'youruser/1630643550_6-967003_highway4_200-200-400-400_24_769.mp4', 'fileSize': 7589031, 'is_fragmented': False, 'metadata': {'uploadtime': '1630643550', 'event-instancename': 'highway4', 'event-timestamp': '1630643550', 'productid': 'Bfuk14xm40eMSxwEEyrd908yzmDIwKp5', 'event-numberofchanges': '24', 'event-microseconds': '0', 'event-regioncoordinates': '200-200-400-400', 'capture': 'IPCamera', 'event-token': '0', 'publickey': 'ABCDEFGHI!@#$%12345'}, 'bytes_ranges': '', 'bytes_range_on_time': None}, 'source': 'storj'}]
    {'date': 1630643550, 'events': ['monitor', 'sequence', 'analysis', 'throttler', 'notification'], 'provider': 'kstorage', 'request': 'persist', 'payload': {'key': 'youruser/1630643550_6-967003_highway4_200-200-400-400_24_769.mp4', 'fileSize': 7589031, 'is_fragmented': False, 'metadata': {'uploadtime': '1630643550', 'event-instancename': 'highway4', 'event-timestamp': '1630643550', 'productid': 'Bfuk14xm40eMSxwEEyrd908yzmDIwKp5', 'event-numberofchanges': '24', 'event-microseconds': '0', 'event-regioncoordinates': '200-200-400-400', 'capture': 'IPCamera', 'event-token': '0', 'publickey': 'ABCDEFGHI!@#$%12345'}, 'bytes_ranges': '', 'bytes_range_on_time': None}, 'source': 'storj'}
    {"date": 1630643550, "data": {"probabilities": [[0.9019190669059753], [0.8251644968986511], [0.8919550776481628, 0.5001923441886902], [0.8414549231529236], [0.8807628750801086, 0.5700141787528992], [0.8745995759963989]], "labels": [["traffic light"], ["traffic light"], ["traffic light", "car"], ["traffic light"], ["traffic light", "train"], ["traffic light"]], "boxes": [[[489, 375, 525, 455]], [[488, 373, 525, 456]], [[488, 376, 525, 455], [682, 191, 752, 234]], [[489, 376, 525, 454]], [[488, 375, 525, 455], [18, 64, 419, 496]], [[489, 376, 525, 455]]]}, "operation": "classification", "events": ["monitor", "sequence", "analysis", "throttler", "notification"], "provider": "kstorage", "request": "persist", "payload": {"key": "youruser/1630643550_6-967003_highway4_200-200-400-400_24_769.mp4", "fileSize": 7589031, "is_fragmented": false, "metadata": {"uploadtime": "1630643550", "event-instancename": "highway4", "event-timestamp": "1630643550", "productid": "Bfuk14xm40eMSxwEEyrd908yzmDIwKp5", "event-numberofchanges": "24", "event-microseconds": "0", "event-regioncoordinates": "200-200-400-400", "capture": "IPCamera", "event-token": "0", "publickey": "ABCDEFGHI!@#$%12345"}, "bytes_ranges": "", "bytes_range_on_time": null}, "source": "storj"}
    next..
    checking..
