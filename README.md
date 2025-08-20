<h1>Kubernetes HA Cluster with Multi-Control Plane Nodes, External etcd & API Load Balancing</h1>


<h2>Description</h2>
Implemented a resilient Kubernetes cluster with multiple control plane nodes and an external etcd cluster to ensure high availability and fault tolerance. Configured HAProxy for API server load balancing, deployed test application, and validated failover scenarios by simulating master node and etcd node failures. Developed hands-on expertise in enterprise-grade Kubernetes operations and high availability best practices. <br />


<h2>Tools and Technologies</h2>

- Kubernetes
- kubeadm
- etcd
- HAProxy
- Containerd
- Flannel CNI
- Linux (Ubuntu 24.04)
- SSH
- YAML
- OpenSSL
- kubectl
- Systemd

<h2>Project Walk-through</h2>

<p align="center">
1. Prepare Environment and Install Dependencies <br />
<img src="https://i.postimg.cc/QCC3rwDp/0.jpg"/>
<br />
<br />
<img src="https://i.postimg.cc/5yrM9g9D/1.jpg"/>
<br />
<br />
2. Set Up External etcd Cluster <br/>
<img src="https://i.postimg.cc/Y2D7xrN8/2.jpg" />
<br />
<br />
3. Set Up Load Balancer for API Servers (HAProxy) <br/>
<img src="https://i.postimg.cc/MGY2fqK7/3.jpg"/>
<br />
<br />
4. Install Container Runtime and Kubernetes Components <br/>
<img src="https://i.postimg.cc/8CNxS9Bj/4.jpg" />
<br />
<br />
5. Initialize the First Master Node <br/>
<img src="https://i.postimg.cc/MHMF5Sxr/5.jpg" />
<br />
<br />
6. Join Additional Master Nodes <br/>
<img src="https://i.postimg.cc/668Wg4pF/6.jpg" />
<br />
<br />
7. Join Worker Nodes <br/>
<img src="https://i.postimg.cc/ZRrc55bm/7.jpg" />
<br />
<br />
8. Deploy Test Application & Test Failover Scenarios  <br/>
<img src="https://i.postimg.cc/s2XWLPHb/8.jpg" />
<br />
<br />
<img src="https://i.postimg.cc/SNhM4bt6/9.jpg" />
<br />
<br />

</p>

