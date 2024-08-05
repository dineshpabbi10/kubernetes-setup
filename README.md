# Kubernetes Setup (In Progress)
Compute, storage and databases can be expensive on cloud platforms such as AWS, Azure etc. The goal of this setup is to allow anyone deploy kubernetes cluster with following features:

#### Index
- [Pre-Requisites](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#pre-requisites)
- [Deploying k8s cluster on hetzner using hetzner-k3s](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#deploying-kubernetes-cluster-on-hetzner-using-hetzner-k3s--self-managed-)
- [Install Nginx Controller](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#install-nginx-controller-for-ingress)
- [Install certbot](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#install-certbot-manager-for-automatic-ssl-certs-for-nginx-ingress)
- [Install ClougnativePG Postgres Operator](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#setup-cloudnativepg-postgres-operator)
- [Setting up ArgoCD](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#setup-argocd)
- [Setting up K3s cluster on dedicated GPU node on hetzner](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#setting-up-k3s-cluster-on-hetzner-gpu-dedicated-node)
- [Deploying GPU operator on dedicated GPU cluster](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#deploying-gpu-operator-on-dedicated-gpu-server)
- [Expose TCP services on nginx controller](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#how-to-expose-tcp-services-using-nginx-controller-deployed-with-helm)

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
- Install k3sup cli : https://github.com/alexellis/k3sup
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
- Once done run : 
```hetzner-k3s create --config cluster_config.yaml```
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

### Setup CloudnativePG (Postgres) operator
Run :
```
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  cnpg/cloudnative-pg
```

### Setup ArgoCD 
Run :
```
kubectl apply -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable
```

### Setting up k3s cluster on Hetzner GPU dedicated node

- Generate an ssh key without passphrase
- On GPU node firewall add following rules for incoming traffic :
![image](https://github.com/user-attachments/assets/0255710d-1c9b-4857-b590-ef6a0865a0f3)

Reference : https://docs.k3s.io/installation/requirements#networking

- Add ssh key for the user root if the dedicated node is setup with a password
- Run :
```
export IP=<IP address of your GPU node>
k3sup install --ip $IP --user root

# Or use a hostname and SSH key for EC2
export HOST="ec2-3-250-131-77.eu-west-1.compute.amazonaws.com"
k3sup install --host $HOST --user ubuntu \
  --ssh-key $HOME/ec2-key.pem
```
- Run 
```
export KUBECONFIG=<path of kubeconfig file generated by previous command>
```
- Run following commmand to add hetzner cloud controller manager for k8s so k8s cluster can create or remove hetzner cloud resources :
```
helm repo add hcloud https://charts.hetzner.cloud
helm repo update hcloud
helm install hccm hcloud/hcloud-cloud-controller-manager -n kube-system
```
Your k8s cluster is ready !

### Deploying GPU operator on dedicated GPU server
- On GPU server, install nvidia container toolkit
```
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```
&

```
sudo apt-get update
```

&

```
sudo apt-get install -y nvidia-container-toolkit
```

REF : https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt

- Configure containerd (docker image runtime) to use nvidia container toolkit using :

```
sudo nvidia-ctk runtime configure --runtime=containerd
```

&

```
sudo systemctl restart containerd
```

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


### How to expose TCP services using nginx controller deployed with helm
NOTE : the nginx controller must use `use-proxy-pass` as false in order for this to work. This is set in following command. So better create separate nginx controller.
```
kubectl -n ingress-nginx annotate services ingress-nginx-controller \
  load-balancer.hetzner.cloud/name=nginx-lb \
  load-balancer.hetzner.cloud/location="nbg1" \
  load-balancer.hetzner.cloud/use-private-ip="true" \
  load-balancer.hetzner.cloud/uses-proxyprotocol="true" \
  load-balancer.hetzner.cloud/hostname="test.com"
```
<b>Steps:</b>
- Create nginx-values.yaml file with default config . See reference : https://github.com/kubernetes/ingress-nginx/blob/main/charts/ingress-nginx/values.yaml
- In TCP section, example : https://github.com/kubernetes/ingress-nginx/blob/main/charts/ingress-nginx/values.yaml#L1184, Add the route
- run :
```
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --values nginx-values.yaml --namespace ingress-nginx --create-namespace
```
