# K8s-Installation
This repository gives a reference of installation of Kubernetes cluster on Openstack cluster using kubespray

## Setting up the environment

- Install the required packages on the host
  
```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt-get install terraform
```

- Clone the kubespray repository
  
```bash
cd ~
git clone https://github.com/kubernetes-sigs/kubespray.git
```

- We need to create an python virtual environment which contains the required packages for kubespray. Create a virtual environment using the below command.
  
```bash
cd ~
python3 -m venv ./kubespray-venv
source ~/kubespray-venv/bin/activate
```

- Install the required packages using the below command.

```bash
pip install -r ~/kubespray/requirements.txt 
```

## Creating Infrastructure

Kubespray already has automation developed to create the required infrastructure for the kubernetes cluster. Follow the below steps to create the base infrastructure.

- Copy the required files to a directory to hold the terraform state files.
  
```bash
#Name of the cluster
export CLUSTER=openstack-cluster
export VENV=~/kubespray-venv/
sudo mkdir -p /kube-spray-clusters/inventory/$CLUSTER
cd ~/kubespray
cp -LRp ./contrib/terraform/openstack/sample-inventory  /kube-spray-clusters/inventory/$CLUSTER
cp -LRp ./contrib/terraform/openstack /kube-spray-clusters/inventory/$CLUSTER
cd /kube-spray-clusters/inventory/$CLUSTER
ln -s /kube-spray-clusters/inventory/$CLUSTER/openstack/hosts
sudo cp /etc/kolla/clouds.yaml /kube-spray-clusters/inventory/$CLUSTER/openstack/
sudo chown $USER:$USER clouds.yaml
export OS_CLOUD=kolla-admin
```

- Create the **cluster.tfvars** file describing how the infrastructure looks like.
  
```bash
export CLUSTER=openstack-cluster
cd /kube-spray-clusters/inventory/$CLUSTER/openstack/
cat <<EOF > cluster.tfvars
# Name of the kubernetes cluster, all the nodes created in the openstack will be prefixed by this name.
cluster_name = "kcs-cluster"
# list of availability zones available in your OpenStack cluster
#az_list = ["nova"]
# SSH key to use for access to nodes
# Path of the ssh public key that should be used to create a key-pair in Openstack and inject into the nodes.
public_key_path = "~/.ssh/id_rsa.pub"
# Image to use for bastion, masters, standalone etcd instances, and nodes
image = "ubuntu-server-22.04-password-enabled"
# User on the node (ex. core on Container Linux, ubuntu on Ubuntu, etc.)
ssh_user = "ubuntu"
# 0|1 bastion nodes
number_of_bastions = 0
#flavor_bastion = "<UUID>"
# standalone etcds
number_of_etcd = 0
# masters
number_of_k8s_masters = 0
number_of_k8s_masters_no_etcd = 0
number_of_k8s_masters_no_floating_ip = 0
number_of_k8s_masters_no_floating_ip_no_etcd = 0
flavor_k8s_master = "fff7279a-29a9-4aac-935d-6bf307f8392c"
# If defining k8s_masters, ignore the above configuration.
k8s_masters = {
   "master-1" = {
     "az"          = "nova"
     "flavor"      = "fce53855-3668-4f9c-ab45-70180868a06c"
     "floating_ip" = true
     "etcd" = true
   },
   "master-2" = {
     "az"          = "nova"
     "flavor"      = "fce53855-3668-4f9c-ab45-70180868a06c"
     "floating_ip" = true
     "etcd" = true
   },
   "master-3" = {
     "az"          = "nova"
     "flavor"      = "fce53855-3668-4f9c-ab45-70180868a06c"
     "floating_ip" = true
     "etcd" = true
   },
}
# nodes
number_of_k8s_nodes = 3
number_of_k8s_nodes_no_floating_ip = 0
k8s_nodes = {
  "slave-1" = {
    "az" = "nova"
    "flavor" = "e5dfa33c-3b51-4f21-9f40-220efcca6afe"
    "floating_ip" = true
  },
  "slave-2" = {
    "az" = "nova"
    "flavor" = "e5dfa33c-3b51-4f21-9f40-220efcca6afe"
    "floating_ip" = true
  },
  "slave-3" = {
    "az" = "nova"
    "flavor" = "e5dfa33c-3b51-4f21-9f40-220efcca6afe"
    "floating_ip" = true
    "extra_groups" = "calico_rr"
  }
}
flavor_k8s_node = "e5dfa33c-3b51-4f21-9f40-220efcca6afe"
# GlusterFS
# either 0 or more than one
#number_of_gfs_nodes_no_floating_ip = 0
#gfs_volume_size_in_gb = 150
# Container Linux does not support GlusterFS
#image_gfs = "<image name>"
# May be different from other nodes
#ssh_user_gfs = "ubuntu"
#flavor_gfs_node = "<UUID>"
# Name of the internal network to create in openstack, to use an existing network specify  use_existing_network to true along with the network_name.
network_name = "k8s-network"
# Use a existing network with the name of network_name. Set to false to create a network with name of network_name.
# use_existing_network = true
# Kubespray downloads a lot of files from the internet for setting up K8s cluster, in order to enable internet access to the VM's specify the network that should #be used to create the router and attach it as the external gateway.
external_net = "2c8a8bd8-d768-4f28-ad93-d596ae087853"
# CIDR to use for the internal kubernetes network.
subnet_cidr = "192.168.25.0/24"
# If the instances are to be accessed outside of the Openstack environment, specify the floatingip pool.
floatingip_pool = "VPC-poc-Provider"
bastion_allowed_remote_ips = ["0.0.0.0/0"]
port_security_enabled =  true
use_access_ip= 0
EOF
```

- Create the base insfrastructure using the below.

```bash
cd /kube-spray-clusters/inventory/$CLUSTER/openstack/
terraform  init
terraform apply -var-file=cluster.tfvars --auto-approve
...output truncated
bastion_fips = tolist([])
floating_network_id = "2c8a8bd8-d768-4f28-ad93-d596ae087853"
k8s_master_fips = tolist([
  "10.232.189.48",
  "10.232.189.132",
  "10.232.189.142",
])
k8s_node_fips = tolist([
  "10.232.189.185",
  "10.232.189.105",
  "10.232.189.57",
])
private_subnet_id = "bab4d76e-2c85-44af-b23f-c2ee58747f8e"
router_id = " 5d2c782f-00b1-4607-8ef4-e95d2c998a22 "
```

- To view the output anytime after the deployment, use the below.

```bash
cd /kube-spray-clusters/inventory/$CLUSTER/openstack/
bastion_fips = tolist([])
floating_network_id = "2c8a8bd8-d768-4f28-ad93-d596ae087853"
k8s_master_fips = tolist([
  "10.232.189.48",
  "10.232.189.132",
  "10.232.189.142",
])
k8s_node_fips = tolist([
  "10.232.189.185",
  "10.232.189.105",
  "10.232.189.57",
])
private_subnet_id = "bab4d76e-2c85-44af-b23f-c2ee58747f8e"
router_id = " 5d2c782f-00b1-4607-8ef4-e95d2c998a22 "
```

- Create the customisations for the kubernetes cluster.
  
```bash
cat <<EOF > custom_vars.yml
external_openstack_lbaas_enabled: true
external_openstack_lbaas_floating_network_id: "2c8a8bd8-d768-4f28-ad93-d596ae087853"
#external_openstack_lbaas_floating_subnet_id: "4d192ad0-4e51-40c7-a3a7-7a8a73bc5370"
external_openstack_lbaas_method: ROUND_ROBIN
external_openstack_lbaas_provider: amphora
#external_openstack_lbaas_subnet_id: "396eb95e-0cf7-4ba4-af6e-20f57b242e2d"
#external_openstack_lbaas_network_id: "3dc92572-088b-413e-b29e-0a0af478e442"
external_openstack_lbaas_manage_security_groups: false
external_openstack_lbaas_create_monitor: false
external_openstack_lbaas_monitor_delay: 5s
remove_default_searchdomains: false
resolvconf_mode: none
helm_enabled: true
metrics_server_enabled: true
ingress_nginx_enabled: true
cert_manager_enabled: true
cloud_provider: external
external_cloud_provider: openstack
EOF
```

## Bootstrapping Cluster

- Bootstrap the Kubernetes cluster.

```bash
cd ~/kubespray
export CLUSTER=openstack-cluster
ansible-playbook --become -i /kube-spray-clusters/inventory/$CLUSTER/openstack/inventory/$CLUSTER/hosts cluster.yml -e "@custom_vars.yml"
```

- Upgrading the Cluster.
  
```bash
cd ~/kubespray
export CLUSTER=openstack-cluster
ansible-playbook --become -i  /kube-spray-clusters/inventory/$CLUSTER/hosts cluster.yml -e "@custom_vars.yml" -e upgrade_cluster_setup=true
```

- Deleting the cluster
  
```bash
cd ~/kubespray
export CLUSTER=openstack-cluster
ansible-playbook --become -i  /kube-spray-clusters/inventory/$CLUSTER/hosts reset.yml -e "@custom_vars.yml"
```

- Deleting the Infrastructure.

```bash
cd /kube-spray-clusters/inventory/openstack-cluster/openstack
terraform destroy -var-file=cluster.tfvars --auto-approve
```

```bash
cat <<EOF > nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test
  name: test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        lifecycle:
          postStart:
            exec:
              command:
                - /bin/sh
                - -c
                - >
                  echo "<h1>Welcome to NGINX on $HOSTNAME</h1>" > /usr/share/nginx/html/index.html
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  ports:
  - name: 5678-80
    port: 5678
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
status:
  loadBalancer: {}
EOF
kubectl apply -f nginx.yaml
```

```bash
sudo mount -o remount,rw /
mount | grep " / "
sudo fsck /dev/vda1
```

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin


cat /etc/docker/daemon.json
{
"bip": "192.26.0.1/16"
}
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
openssl genpkey -algorithm RSA -out /home/ubuntu/harbor/harbor/certs/harbor.key
# openssl req -new -key /home/ubuntu/harbor/harbor/certs/harbor.key -out /home/ubuntu/harbor/harbor/certs/harbor.csr
openssl req -new -newkey rsa:2048 -nodes -keyout /home/ubuntu/harbor/harbor/certs/harbor.key -out /home/ubuntu/harbor/harbor/certs/harbor.csr -config openssl-san.cnf
openssl x509 -req -in harbor.csr -signkey harbor.key -out harbor.crt -days 3650 -extensions v3_req -extfile openssl-san.cnf

# openssl x509 -req -in /home/ubuntu/harbor/harbor/certs/harbor.csr -signkey /home/ubuntu/harbor/harbor/certs/harbor.key -out /home/ubuntu/harbor/harbor/certs/harbor.crt

```
