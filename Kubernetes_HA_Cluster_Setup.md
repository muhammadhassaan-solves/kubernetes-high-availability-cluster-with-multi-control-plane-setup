# Kubernetes HA Cluster with Multi-Control Plane Nodes, External etcd & API Load Balancing

## Overview
This guide demonstrates how to deploy a resilient Kubernetes cluster with multiple control plane nodes and an external etcd cluster to ensure high availability, fault tolerance, and enterprise-grade operations. 

## Step 1: Prepare Environment and Install Dependencies

```bash
# Add EC2 private IPs and hostnames (run on all nodes)
sudo tee -a /etc/hosts <<EOF
172.31.95.247 controlplane1
172.31.90.30 controlplane2
172.31.80.174 controlplane3
172.31.80.239 worker1
172.31.95.49 worker2
172.31.80.20 etcd1
172.31.94.69 etcd2
172.31.84.38 etcd3
172.31.83.131 lb1
172.31.83.131 k8s-api.local
EOF
```

```bash
# Update packages and install dependencies (all nodes) 
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Disable swap for Kubernetes
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load kernel modules
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl for Kubernetes networking
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

```

## Step 2: Set Up External etcd Cluster

```bash
# Install etcd binaries (all etcd nodes)
ETCD_VERSION="v3.5.9"
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz -o etcd-${ETCD_VERSION}-linux-amd64.tar.gz
tar xzf etcd-${ETCD_VERSION}-linux-amd64.tar.gz
sudo mv etcd-${ETCD_VERSION}-linux-amd64/etcd* /usr/local/bin/
sudo chmod +x /usr/local/bin/etcd*

# Create etcd user and directories
sudo useradd -r -s /bin/false etcd
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chown etcd:etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd

```

```bash
# Generate CA for etcd (run on etcd1 only)
mkdir -p ~/etcd-certs && cd ~/etcd-certs

openssl genrsa -out ca-key.pem 2048
openssl req -new -x509 -key ca-key.pem -out ca.pem -days 365 -subj "/CN=etcd-ca"

```

```bash
# Generate server certificates (run on etcd1 only)
cat > etcd-csr.conf <<EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = etcd

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = etcd1
DNS.2 = etcd2
DNS.3 = etcd3
DNS.4 = localhost
IP.1 = 172.31.80.20
IP.2 = 172.31.94.69
IP.3 = 172.31.84.38
IP.4 = 127.0.0.1
EOF

```

```bash
openssl genrsa -out etcd-key.pem 2048
openssl req -new -key etcd-key.pem -out etcd.csr -config etcd-csr.conf
openssl x509 -req -in etcd.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out etcd.pem -days 365 -extensions v3_req -extfile etcd-csr.conf
```

```bash
# Copy certificates to etcd2 and etcd3 (run from etcd1)
scp -i /home/ubuntu/my_key.pem ca.pem etcd.pem etcd-key.pem ubuntu@etcdnode2publicip:~/
ssh -i /home/ubuntu/my_key.pem ubuntu@etcdnode2publicip "sudo mkdir -p /etc/etcd && sudo mv ~/ca.pem ~/etcd.pem ~/etcd-key.pem /etc/etcd/ && sudo chown etcd:etcd /etc/etcd/*.pem"

scp -i /home/ubuntu/my_key.pem ca.pem etcd.pem etcd-key.pem ubuntu@etcdnode3publicip:~/
ssh -i /home/ubuntu/my_key.pem ubuntu@etcdnode3publicip "sudo mkdir -p /etc/etcd && sudo mv ~/ca.pem ~/etcd.pem ~/etcd-key.pem /etc/etcd/ && sudo chown etcd:etcd /etc/etcd/*.pem"

# Move etcd1 certificates to /etc/etcd just like etcd2 and etcd3.
sudo mv *.pem /etc/etcd/
sudo chown etcd:etcd /etc/etcd/*.pem

```

```bash
# Configure etcd systemd service (run on all etcd nodes) 
# For etcd2 and etcd3, just swap ETCD_NAME and the corresponding IPs for LISTEN and ADVERTISE fields.
sudo tee /etc/etcd/etcd.conf <<EOF
ETCD_NAME=etcd1
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_PEER_URLS=https://172.31.80.20:2380
ETCD_LISTEN_CLIENT_URLS=https://172.31.80.20:2379,https://127.0.0.1:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://172.31.80.20:2380
ETCD_ADVERTISE_CLIENT_URLS=https://172.31.80.20:2379
ETCD_INITIAL_CLUSTER=etcd1=https://172.31.80.20:2380,etcd2=https://172.31.94.69:2380,etcd3=https://172.31.84.38:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
ETCD_CERT_FILE=/etc/etcd/etcd.pem
ETCD_KEY_FILE=/etc/etcd/etcd-key.pem
ETCD_TRUSTED_CA_FILE=/etc/etcd/ca.pem
ETCD_PEER_CERT_FILE=/etc/etcd/etcd.pem
ETCD_PEER_KEY_FILE=/etc/etcd/etcd-key.pem
ETCD_PEER_TRUSTED_CA_FILE=/etc/etcd/ca.pem
ETCD_PEER_CLIENT_CERT_AUTH=true
ETCD_CLIENT_CERT_AUTH=true
EOF

```

```bash
# Create systemd service for etcd (all etcd nodes)
sudo tee /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
Type=notify
User=etcd
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
sudo systemctl status etcd

```

```bash
# Verify etcd cluster health (all etcd nodes)
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.pem --cert=/etc/etcd/etcd.pem --key=/etc/etcd/etcd-key.pem endpoint health
```


## Step 3: Set Up Load Balancer (HAProxy)

```bash
# Install HAProxy (on load balancer node)
sudo apt update
sudo apt install -y haproxy
```

```bash
# Configure HAProxy for API server load balancing
sudo tee /etc/haproxy/haproxy.cfg <<EOF
global
    log stdout local0
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    mode http
    log global
    option httplog
    option dontlognull
    option log-health-checks
    option forwardfor except 127.0.0.0/8
    option redispatch
    retries 3
    timeout http-request 10s
    timeout queue 20s
    timeout connect 10s
    timeout client 20s
    timeout server 20s
    timeout http-keep-alive 10s
    timeout check 10s

frontend k8s-api-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server master1 172.31.95.247:6443 check
    server master2 172.31.90.30:6443 check 
    server master3 172.31.80.174:6443 check

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE
EOF

```


```bash
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```

```bash
# Verify HAProxy stats
curl http://lb1:8404/stats

```

## Step 4: Install Container Runtime & Kubernetes Components


```bash
# Install containerd (all nodes)
sudo apt-get install -y containerd

# Generate default containerd config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Use systemd for cgroups
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```
```bash
# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

```bash
# Install kubeadm, kubelet, kubectl (all nodes)
sudo apt update
sudo apt install -y curl apt-transport-https ca-certificates gnupg

sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update

sudo apt-get install -y --allow-change-held-packages kubelet kubeadm kubectl

# Hold packages to prevent automatic upgrade
sudo apt-mark hold kubelet kubeadm kubectl

sudo tee /etc/default/kubelet <<EOF
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd
EOF

sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl restart kubelet
sudo systemctl status kubelet
```
```bash
kubeadm version
kubectl version --client
systemctl status kubelet
```

## Step 5: Initialize the First Control Plane Node

```bash
# Copy etcd certificates to all control plane nodes
scp /etc/etcd/ca.pem /etc/etcd/etcd.pem /etc/etcd/etcd-key.pem ubuntu@controlplane1:~/
scp /etc/etcd/ca.pem /etc/etcd/etcd.pem /etc/etcd/etcd-key.pem ubuntu@controlplane2:~/
scp /etc/etcd/ca.pem /etc/etcd/etcd.pem /etc/etcd/etcd-key.pem ubuntu@controlplane3:~/
```

```bash
# Move certificates to kube pki directory (all control planes)
sudo mkdir -p /etc/kubernetes/pki/etcd
sudo mv ~/ca.pem /etc/kubernetes/pki/etcd/
sudo mv ~/etcd.pem /etc/kubernetes/pki/etcd/
sudo mv ~/etcd-key.pem /etc/kubernetes/pki/etcd/
```

```bash
# Create kubeadm configuration for cluster initialization (run on controlplane1)
sudo tee /etc/kubernetes/kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.31.95.247   # master1 private IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.28.2
controlPlaneEndpoint: "k8s-api.local:6443"
etcd:
  external:
    endpoints:
    - https://172.31.80.20:2379   # etcd1
    - https://172.31.94.69:2379   # etcd2
    - https://172.31.84.38:2379   # etcd3
    caFile: /etc/kubernetes/pki/etcd/ca.pem
    certFile: /etc/kubernetes/pki/etcd/etcd.pem
    keyFile: /etc/kubernetes/pki/etcd/etcd-key.pem
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.244.0.0/16"
apiServer:
  certSANs:
  - "k8s-api.local"
  - "172.31.83.131"   # load balancer
  - "172.31.95.247"   # controlplane1
  - "172.31.90.30"    # controlplane2
  - "172.31.80.174"   # controlplane3
  - "127.0.0.1"
  - "localhost"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: KubeletConfiguration
cgroupDriver: systemd
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
EOF

```

```bash
# Initialize first control plane
sudo kubeadm init --config=/etc/kubernetes/kubeadm-config.yaml --upload-certs
```

```bash
# Set up kubectl for ubuntu user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
# Install Flannel network
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
# Verify
kubectl get nodes
kubectl get pods -n kube-system
```

## Step 6: Join Additional Control Plane Nodes

```bash
# Run on controlplane2 and controlplane3 (get token/cert key from controlplane1)
sudo kubeadm join k8s-api.local:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <certificate-key>
```

```bash
# Set up kubectl on both
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
# Verify cluster from any control plane
kubectl get nodes
kubectl get pods -n kube-system -o wide
```

## Step 7: Join Worker Nodes
```bash
# Run on worker1 & worker2
sudo kubeadm join k8s-api.local:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```
```bash
# Verify cluster from any control plane
kubectl get nodes
kubectl get pods -n kube-system
```

```bash
# Label worker nodes
kubectl label node worker1 node-role.kubernetes.io/worker=worker
kubectl label node worker2 node-role.kubernetes.io/worker=worker
```


