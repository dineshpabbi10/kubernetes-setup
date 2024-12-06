# Kubernetes Setup
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
- [Apply time-slicing for GPU in dedicated GPU node cluster](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#apply-time-slicing-for-gpu-nodes-on-dedicated-gpu-cluster)
- [Deploying Ollama](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#deploying-ollama)
- [Deploying Ollama Web UI](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#ollama-web-ui)
- [Deploying RabbitMQ Operator](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#rabbit-mq-operator)
- [Setting up CloudnativePG](https://github.com/dineshpabbi10/kubernetes-setup/blob/main/README.md#cloudnativepg)

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
- READ : Hetzner Cloud Control Manager : https://github.com/hetznercloud/hcloud-cloud-controller-manager
- READ : Hetzner CSI Driver : https://github.com/hetznercloud/csi-driver
- READ : Hetzner Proxy Protocol : https://docs.hetzner.com/cloud/load-balancers/faq/#what-does-proxy-protocol-mean-and-should-i-enable-it
- READ : Nginx use proxy protocol : https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/
- READ : Why proxy protocol does not work with TCP connections : https://github.com/kubernetes/ingress-nginx/issues/5748
  
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

Also setup ingress configmap by adding :
Reference: https://docs.digitalocean.com/support/how-do-i-enable-proxy-protocol-when-my-load-balancer-sends-requests-to-the-nginx-ingress-controller/

When nginx is deploying, it also creates a config map. You can find it using : `kubectl get config -n ingress-nginx`. Once you get the configmap name, edit it using `kubectl edit configmap <config name> -n ingress-nginx` and add following section:

```
  config:
    use-proxy-protocol: "true"
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
 Ref : https://medium.com/sparque-labs/serving-ai-models-on-the-edge-using-nvidia-gpu-with-k3s-on-aws-part-4-dd48f8699116
```
helm install --wait nvidiagpu      -n gpu-operator --create-namespace     --set toolkit.env[0].name=CONTAINERD_CONFIG     --set toolkit.env[0].value=/var/lib/rancher/k3s/agent/etc/containerd/config.toml     --set toolkit.env[1].name=CONTAINERD_SOCKET     --set toolkit.env[1].value=/run/k3s/containerd/containerd.sock     --set toolkit.env[2].name=CONTAINERD_RUNTIME_CLASS     --set toolkit.env[2].value=nvidia     --set toolkit.env[3].name=CONTAINERD_SET_AS_DEFAULT     --set-string toolkit.env[3].value=true      nvidia/gpu-operator
```


### How to expose TCP services using nginx controller deployed with helm
Ref: https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/
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

### Apply time-slicing for GPU nodes on dedicated gpu cluster

##### What is time-slicing ?
The NVIDIA GPU Operator enables oversubscription of GPUs through a set of extended options for the NVIDIA Kubernetes Device Plugin. GPU time-slicing enables workloads that are scheduled on oversubscribed GPUs to interleave with one another.

This mechanism for enabling time-slicing of GPUs in Kubernetes enables a system administrator to define a set of replicas for a GPU, each of which can be handed out independently to a pod to run workloads on. Unlike Multi-Instance GPU (MIG), there is no memory or fault-isolation between replicas, but for some workloads this is better than not being able to share at all. Internally, GPU time-slicing is used to multiplex workloads from replicas of the same underlying GPU.

##### Steps
- create `time-slicing-config.yaml` file with following content:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config-all
data:
  any: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 4
```
- Make sure gpu-operator namespace exists. Run :
```
kubectl create -n gpu-operator -f time-slicing-config.yaml
```
- Configure nvidia device plugin to use time-slicing :
```
kubectl patch clusterpolicies.nvidia.com/cluster-policy \
    -n gpu-operator --type merge \
    -p '{"spec": {"devicePlugin": {"config": {"name": "time-slicing-config-all", "default": "any"}}}}'
```

Once applied, every gpu will publish 4 GPUs instead of one.

### Deploying Ollama
Reference : https://github.com/otwld/ollama-helm
Helm Values :
```yaml
runtimeClassName: "nvidia"

ollama:
  gpu:
    # -- Enable GPU integration
    enabled: true
    
    # -- GPU type: 'nvidia' or 'amd'
    type: 'nvidia'
    
    # -- Specify the number of GPU to 1
    number: 40
   
  # -- List of models to pull at container startup
  models: 
    - gemma2:2b
    - llama3.1:8b
    - command-r:35b-08-2024-q4_1
    - command-r:35b-08-2024-q6_K
    - llava:13b
    - nomic-embed-text
    - gemma2:9b-instruct-q8_0
    - gemma2:27b-instruct-q8_0
    - llama3.2-vision:11b
    - llama3.2:3b
    - gemma2:27b-instruct-q6_K
    - qwen2.5:32b
    - mistral-nemo:12b-instruct-2407-q8_0
    - phi3:14b-medium-128k-instruct-q8_0
    - phi3:14b-medium-4k-instruct-q8_0

extraEnv:
  - name: NVIDIA_VISIBLE_DEVICES
    value: all
  - name: NVARCH
    value: x86_64
  - name: NV_CUDA_CUDART_VERSION
    value: 12.3.2
  - name: NVIDIA_DRIVER_CAPABILITIES
    value: all
```

### Ollama web ui:
Reference : https://artifacthub.io/packages/helm/open-webui/open-webui
Helm Values:

```yaml
nameOverride: ""

ollama:
  # -- Automatically install Ollama Helm chart from https://otwld.github.io/ollama-helm/. Use [Helm Values](https://github.com/otwld/ollama-helm/#helm-values) to configure
  enabled: false
  # -- If enabling embedded Ollama, update fullnameOverride to your desired Ollama name value, or else it will use the default ollama.name value from the Ollama chart
  fullnameOverride: "open-webui-ollama"
  # -- Example Ollama configuration with nvidia GPU enabled, automatically downloading a model, and deploying a PVC for model persistence
  # ollama:
  #   gpu:
  #     enabled: true
  #     type: 'nvidia'
  #     number: 1
  #   models:
  #     - llama3
  # runtimeClassName: nvidia
  # persistentVolume:
  #   enabled: true

pipelines:
  # -- Automatically install Pipelines chart to extend Open WebUI functionality using Pipelines: https://github.com/open-webui/pipelines
  enabled: true
  # -- This section can be used to pass required environment variables to your pipelines (e.g. Langfuse hostname)
  extraEnvVars: []

# -- A list of Ollama API endpoints. These can be added in lieu of automatically installing the Ollama Helm chart, or in addition to it.
ollamaUrls: 
  - <ollama endpoint>

# -- Value of cluster domain
clusterDomain: cluster.local

annotations: {}
podAnnotations: {}
replicaCount: 1
# -- Open WebUI image tags can be found here: https://github.com/open-webui/open-webui/pkgs/container/open-webui
image:
  repository: ghcr.io/open-webui/open-webui
  tag: "latest"
  pullPolicy: "IfNotPresent"
resources: {}
ingress:
  enabled: false
  class: ""
  # -- Use appropriate annotations for your Ingress controller, e.g., for NGINX:
  # nginx.ingress.kubernetes.io/rewrite-target: /
  annotations: {}
  host: ""
  tls: false
  existingSecret: ""
persistence:
  enabled: true
  size: 200Gi
  # -- Use existingClaim if you want to re-use an existing Open WebUI PVC instead of creating a new one
  existingClaim: ""
  # -- If using multiple replicas, you must update accessModes to ReadWriteMany
  accessModes:
    - ReadWriteOnce
  storageClass: ""
  selector: {}
  annotations: {}

# -- Node labels for pod assignment.
nodeSelector: {}

# -- Tolerations for pod assignment
tolerations: []

# -- Affinity for pod assignment
affinity: {}

# -- Service values to expose Open WebUI pods to cluster
service:
  type: ClusterIP
  annotations: {}
  port: 80
  containerPort: 8080
  nodePort: ""
  labels: {}
  loadBalancerClass: ""

# -- OpenAI base API URL to use. Defaults to the Pipelines service endpoint when Pipelines are enabled, and "https://api.openai.com/v1" if Pipelines are not enabled and this value is blank
openaiBaseApiUrl: ""

# -- Additional environments variables on the output Deployment definition. Most up-to-date environment variables can be found here: https://docs.openwebui.com/getting-started/env-configuration/
extraEnvVars:
  # -- Default API key value for Pipelines. Should be updated in a production deployment, or be changed to the required API key if not using Pipelines
  - name: OPENAI_API_KEY
    value: "0p3n-w3bu!"
  # valueFrom:
  #   secretKeyRef:
  #     name: pipelines-api-key
  #     key: api-key
  # - name: OPENAI_API_KEY
  #   valueFrom:
  #     secretKeyRef:
  #       name: openai-api-key
  #       key: api-key
  # - name: OLLAMA_DEBUG
  #   value: "1"

# -- Configure pod security context
# ref: <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-containe>
podSecurityContext:
  {}
  # fsGroupChangePolicy: Always
  # sysctls: []
  # supplementalGroups: []
  # fsGroup: 1001

# -- Configure container security context
# ref: <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-containe>
containerSecurityContext:
  {}
  # runAsUser: 1001
  # runAsGroup: 1001
  # runAsNonRoot: true
  # privileged: false
  # allowPrivilegeEscalation: false
  # readOnlyRootFilesystem: false
  # capabilities:
  #   drop:
  #     - ALL
  # seccompProfile:
  #   type: "RuntimeDefault"
```

### Rabbit MQ Operator

#### Installation 
https://www.rabbitmq.com/kubernetes/operator/install-operator

#### Creating Rabbitmq clusters
https://www.rabbitmq.com/kubernetes/operator/using-operator#replicas

### CloudNativePG

#### Installation
https://cloudnative-pg.io/documentation/1.24/installation_upgrade/

#### Configuring Clusters
https://cloudnative-pg.io/documentation/1.24/quickstart/#part-3-deploy-a-postgresql-cluster

#### Setting Up Real Time Backups
1. https://cloudnative-pg.io/documentation/1.24/backup_barmanobjectstore/
2. Create Backup Schedule Resource : https://cloudnative-pg.io/documentation/1.24/backup/#scheduled-backups

