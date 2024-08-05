# Kubernetes Setup (In Progress)
Compute, storage and databases can be expensive on cloud platforms such as AWS, Azure etc. The goal of this setup is to allow anyone deploy kubernetes cluster with following features:

#### Features
- About 70-80% cheaper than AWS
- Deploy elastic clusters which can be resized and expanded easily
- Deploy database clusters (Postgres) with multiple replicas, continous backups and point in time recovery
- Ability to run GPU workloads with nvidia gpu operator on dedicated machines
- Exposing kubernetes services to public internet with nginx ingress
- Automatic provisioning and renewal of SSL certificates for all public facing services

## Pre-Requisites
- Hetzner cloud account : https://hetzner.cloud/?ref=VDxeuaU6cw7u [ Referral Link ]
- Install hetzner-k3s : https://github.com/vitobotta/hetzner-k3s
- A hetzner cloud project ( Click New Project on hetzner cloud dashboard )
- A hetzner cloud project token (Read and Write)
1.
![image](https://github.com/user-attachments/assets/4ee45ae0-97ee-4ce7-8846-9ac07a007ce4)

2.
![image](https://github.com/user-attachments/assets/a9213b77-894a-4524-ba6b-e0078273d2a2)

3.
![image](https://github.com/user-attachments/assets/5e7899ba-2dc0-49d3-ab9e-9f31ab33affa)

## Steps
### Deploying kubernetes cluster on hetzner using hetzner-k3s ( Self Managed )
- Create a cluster.yaml file as shown here : https://github.com/vitobotta/hetzner-k3s?tab=readme-ov-file#creating-a-cluster
- In cluster.yaml file , replace hetzner-token with the read-write project token
- Specify the instance and region of worker-nodes, master-nodes in cluster.yaml file
- Generate an ssh file WITHOUT any passphrase (This is important) and specify path of generated ssh files in cluster.yaml file
- Specify path of kubeconfig file in the cluster.yaml. This file is used to run commands against the k8s cluster later
- Once done run : `hetzner-k3s create --config cluster_config.yaml`
- Wait for everything to finish

### Install nginx controller for ingress
- Run :
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
- You will notice a public ip is pending for nginx on running : `kubectl get svc -n ingress-nginx`. That's because hetzner need additional configuration for lb to be provisioned (next step)
- Run following so hetzner can create hetzner loadbalancer:
```
kubectl -n ingress-nginx annotate services ingress-nginx-controller \
  load-balancer.hetzner.cloud/name=nginx-lb \
  load-balancer.hetzner.cloud/location="nbg1" \
  load-balancer.hetzner.cloud/use-private-ip="true" \
  load-balancer.hetzner.cloud/uses-proxyprotocol="true" \
  load-balancer.hetzner.cloud/hostname="test.com"
```

### Install certbot manager for automatic ssl certs for nginx ingress
- Apply hairpin proxy. Cert generation fails without this.
```
kubectl apply -f https://raw.githubusercontent.com/compumike/hairpin-proxy/v0.2.1/deploy.yml
```
- Run :
```
helm repo add jetstack https://charts.jetstack.io --force-update
```
&
```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.2 \
  --set crds.enabled=true
```

### Setting up k3s cluster on Hetzner GPU dedicated node
TO BE WRITTEN LATER

### Deploying GPU operator on dedicated GPU server
- On GPU server, install nvidia container toolkit
```
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```
&

```sudo apt-get update```

&

```sudo apt-get install -y nvidia-container-toolkit```

REF : https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt

- Configure containerd (docker image runtime) to use nvidia container toolkit using :

```sudo nvidia-ctk runtime configure --runtime=containerd```

&

```sudo systemctl restart containerd```

NOTE: This can change based on what docker runtime you have 

- On the GPU dedicated server, install nvidia-smi using :
```
sudo apt purge nvidia-*
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
sudo apt install nvidia-381
```
- Setup Helm repo using 
```
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia \
    && helm repo update
```
- Install gpu operator with following command
 
```
helm install --wait nvidiagpu      -n gpu-operator --create-namespace     --set toolkit.env[0].name=CONTAINERD_CONFIG     --set toolkit.env[0].value=/var/lib/rancher/k3s/agent/etc/containerd/config.toml     --set toolkit.env[1].name=CONTAINERD_SOCKET     --set toolkit.env[1].value=/run/k3s/containerd/containerd.sock     --set toolkit.env[2].name=CONTAINERD_RUNTIME_CLASS     --set toolkit.env[2].value=nvidia     --set toolkit.env[3].name=CONTAINERD_SET_AS_DEFAULT     --set-string toolkit.env[3].value=true      nvidia/gpu-operator
```
