<h1>Kubernetes HA Cluster with Multi-Control Plane Nodes, External etcd & API Load Balancing</h1>


<h2>Description</h2>
Implemented a resilient Kubernetes cluster with multiple control plane nodes and an external etcd cluster to ensure high availability and fault tolerance. Configured HAProxy for API server load balancing, validated failover scenarios, and deployed test applications to verify cluster stability. Developed hands-on expertise in enterprise-grade Kubernetes operations and high availability best practices. <br />


<h2>Tools and Technologies</h2>

- Kubernetes
- kubeadm
- etcd
- HAProxy
- Docker
- Flannel CNI
- Linux (Ubuntu 24.04)
- SSH
- YAML
- OpenSSL
- kubectl
- Systemd

<h2>Project Walk-through</h2>

<p align="center">
Prepare Environment and Install Dependencies <br />
<img src="https://i.postimg.cc/QCC3rwDp/0.jpg"/>
<br />
<br />
<img src="https://i.postimg.cc/5yrM9g9D/1.jpg"/>
<br />
<br />
Set Up External etcd Cluster <br/>
<img src="https://i.postimg.cc/Y2D7xrN8/2.jpg" />
<br />
<br />
Set Up Load Balancer for API Servers (HAProxy) <br/>
<img src="https://i.postimg.cc/MGY2fqK7/3.jpg"/>
<br />
<br />
Install Container Runtime and Kubernetes Components <br/>
<img src="https://i.postimg.cc/8CNxS9Bj/4.jpg" />
<br />
<br />
Initialize the First Master Node <br/>
<img src="https://i.postimg.cc/MHMF5Sxr/5.jpg" />
<br />
<br />
Join Additional Master Nodes <br/>
<img src="https://i.postimg.cc/668Wg4pF/6.jpg" />
<br />
<br />
Join Worker Nodes <br/>
<img src="https://i.postimg.cc/ZRrc55bm/7.jpg" />
<br />
<br />
Deploy Test Application & Test Failover Scenarios by simulating master node and etcd node failures <br/>
<img src="https://i.postimg.cc/s2XWLPHb/8.jpg" />
<br />
<br />
<img src="https://i.postimg.cc/SNhM4bt6/9.jpg" />
<br />
<br />

</p>

