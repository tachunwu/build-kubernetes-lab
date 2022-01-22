# ☸ Build Kubernetes Lab ☸
The simple way to build Kubernetes cluster. This is for dev-use if you want to build a minimum production environment at least use 3 masters 3 workers setup.

## ☸ Step 0: Pre-require
* GCP Account: Google gives user $300 to try their platform. In my opinion, it's awesome.
* gcloud SDK: You need this command-line tool to setup VM and network.
* Create a GCP Project

## ☸ Step 1: Project setup
### Init gcloud
```
gloud init
```
### Set region
We're in Taiwan! Choose our data center~
```
gcloud config set compute/region asia-east1
```
### Set zone
```
gcloud config set compute/zone asia-east1-a
```
### Set project
```
gcloud config set project <PROJECT_ID>
```

## ☸ Step 2: Network 
### Create VPC
```
gcloud compute networks create k8s-lab --subnet-mode custom
```
### Create Subset
```
gcloud compute networks subnets create k8s-nodes \
  --network k8s-lab \
  --range 10.240.0.0/24
```
### Create Internal Firewall
```
gcloud compute firewall-rules create k8s-lab-allow-internal \
  --allow tcp,udp,icmp,ipip \
  --network k8s-lab \
  --source-ranges 10.240.0.0/24
```

### Create External Firewall
```
gcloud compute firewall-rules create k8s-lab-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network k8s-lab \
  --source-ranges 0.0.0.0/0
```
### Verify Network
```
gcloud compute firewall-rules list --filter="network:k8s-lab"
```
### Verify result
```
NAME                    NETWORK  DIRECTION  PRIORITY  ALLOW                         DENY  DISABLED
default-allow-icmp      default  INGRESS    65534     icmp                                False
default-allow-internal  default  INGRESS    65534     tcp:0-65535,udp:0-65535,icmp        False
default-allow-rdp       default  INGRESS    65534     tcp:3389                            False
default-allow-ssh       default  INGRESS    65534     tcp:22                              False
k8s-lab-allow-external  k8s-lab  INGRESS    1000      tcp:22,tcp:6443,icmp                False
k8s-lab-allow-internal  k8s-lab  INGRESS    1000      tcp,udp,icmp,ipip                   False
```

### Create Publisc IP for Load Balanacer
```
gcloud compute addresses create k8s-lab \
  --region $(gcloud config get-value compute/region)
```
### Verify Publisc IP
```
gcloud compute addresses list --filter="name=('k8s-lab')"
```
### Verify result
```
NAME     ADDRESS/RANGE   TYPE      PURPOSE  NETWORK  REGION      SUBNET  STATUS
k8s-lab  35.236.182.103  EXTERNAL                    asia-east1          RESERVED
```
## ☸ Step 3: VM
### Create 1 Controller
```
gcloud compute instances create controller \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n2-standard-8 \
    --private-network-ip 10.240.0.11 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-nodes \
    --tags k8s-lab,controller
```
### Create 3 Worker
```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-8 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-nodes \
    --tags k8s-lab,worker
done

```
### Verify Instances
```
gcloud compute instances list --filter="tags.items=k8s-lab"
```
### Verify result
```
NAME        ZONE          MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller  asia-east1-a  n2-standard-8               10.240.0.11  35.234.18.211   RUNNING
worker-0    asia-east1-a  n1-standard-8               10.240.0.20  104.199.163.21  RUNNING
worker-1    asia-east1-a  n1-standard-8               10.240.0.21  35.236.159.39   RUNNING
worker-2    asia-east1-a  n1-standard-8               10.240.0.22  35.221.187.201  RUNNING
```

## ☸ Step 4: Kubernetes
### SSH
```
gcloud compute ssh controller
gcloud compute ssh worker-0
gcloud compute ssh worker-1
gcloud compute ssh worker-2
```

### Close swap
Important! MUST close swap!
```
swapoff -a
```
### Install Docker 
Need to do these steps on all instances!
```
{
sudo apt update
sudo apt install -y docker.io 
sudo systemctl enable docker.service
sudo apt install -y apt-transport-https curl
}
```
### Install kubelet kubeadm kubectl
```
{
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
}
```

### Error solving
If kubelet isn't running you need to fix that before using kubeadm!
```
sudo touch /etc/docker/daemon.json
sudo vim /etc/docker/daemon.json

{
  "exec-opts": ["native.cgroupdriver=systemd"]
}

{
sudo systemctl restart docker
sudo systemctl restart kubelet
}
```
### Create seed on controller node
Before using kubeadm docker, kubelet need alive!
```
sudo kubeadm init --pod-network-cidr 192.168.0.0/16
```

### kubeadm join workers
You'll get link that allow to to join workers to controller.
```
NAME         STATUS     ROLES                  AGE     VERSION
controller   NotReady   control-plane,master   22m     v1.23.2
worker-0     NotReady   <none>                 5m43s   v1.23.2
worker-1     NotReady   <none>                 31s     v1.23.2
worker-2     NotReady   <none>                 33s     v1.23.2
```

### Install Calico
```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```

### Final verify
```
NAME         STATUS   ROLES                  AGE     VERSION
controller   Ready    control-plane,master   27m     v1.23.2
worker-0     Ready    <none>                 11m     v1.23.2
worker-1     Ready    <none>                 5m48s   v1.23.2
worker-2     Ready    <none>                 5m50s   v1.23.2
```
## ☸ Step 5: Clean
Just delete the project.

## ☸ Reference
* [General-purpose machine family](https://cloud.google.com/compute/docs/general-purpose-machines#n2_machines)
* [Self-managed Kubernetes in Google Compute Engine (GCE)](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-public-cloud/gce)
* [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)    
    



