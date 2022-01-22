# Build Kubernetes Lab
The simple way to build kubernetes cluster. 

## Step0: Pre-require
* GCP Account
* gcloud SDK

## Step1: Project setup
```
gloud init
```

```
gcloud config set compute/region us-west1
```

```
gcloud config set compute/zone us-west1-c
```

## Step2: Network 
### VPC
```
gcloud compute networks create k8s-lab --subnet-mode custom
```
### Subset
```
gcloud compute networks subnets create k8s-nodes \
  --network k8s-lab \
  --range 10.240.0.0/24
```
### Internal
```
gcloud compute firewall-rules create k8s-lab-allow-internal \
  --allow tcp,udp,icmp,ipip \
  --network k8s-lab \
  --source-ranges 10.240.0.0/24
```

### External
```
gcloud compute firewall-rules create k8s-lab-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network k8s-lab \
  --source-ranges 0.0.0.0/0
```

## Step3: VM
### Create Controller
```
gcloud compute instances create controller \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n2-standard-16 \
    --private-network-ip 10.240.0.11 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-nodes \
    --zone us-west1-c \
    --tags k8s-lab,controller
```
### Create Worker
```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n2-standard-16 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-nodes \
    --zone us-west1-c \
    --tags k9s-lab,worker
done

```

## Step4: Kubernetes
### Install Docker
```
sudo apt update
sudo apt install -y docker.io 
sudo systemctl enable docker.service
sudo apt install -y apt-transport-https curl
```

### Install kubelet, kubeadm, kubectl
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Reference
[General-purpose machine family](https://cloud.google.com/compute/docs/general-purpose-machines#n2_machines)
[Self-managed Kubernetes in Google Compute Engine (GCE)](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-public-cloud/gce)
[Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)    
    



