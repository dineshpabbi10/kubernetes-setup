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
- Once done run : `hetzner-k3s create --config cluster_config.yaml`
