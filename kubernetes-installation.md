# UBUNTU
step 1 to disabled swap memory (because Kubernetes relies on predictable memory usage so that Kubernetes should have actual memory )

1. sudo swapoff -a
2. sed -i '/swap/d' /etc/fstab 

# Update Ubuntu 
After that we need to run the following commands to install Containerd:

1. `sudo apt-get update`
2. `sudo apt install apt-transport-https curl`
3. `sudo mkdir -p /etc/apt/keyrings`
4. `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`
5. `echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null.`
6. `sudo apt-get update`
7. `sudo apt-get install containerd.io`
8. `sudo mkdir -p /etc/containerd`
9. `sudo containerd config default | sudo tee /etc/containerd/config.toml`
10. `sudo nano /etc/containerd/config.toml [set SystemdCgroup = true]`
11. `sudo systemctl restart containerd`


The purpose of setting SystemdCgroup = true in the config.toml file of containerd is to enable the use of systemd for managing cgroups (control groups) for containers.

Containered Should be Installed Now....


### Install Kubernetes [kubeadm, kubelet, kubectl]

13. `curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`
14. `echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list`
15. `sudo apt update`
16. `sudo apt install -y kubelet kubeadm kubectl`
17. `sudo apt-mark hold kubelet kubeadm kubectl`
18. `modprobe br_netfilter`
19. `cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf`

`net.bridge.bridge-nf-call-iptables  = 1`

`net.bridge.bridge-nf-call-ip6tables = 1`

`net.ipv4.ip_forward                 = 1`

`EOF`

21. `sysctl --system`
22.  `kubeadm init --pod-network-cidr=10.244.0.0/16` ==> Run this command only on Master node.
### IF all goes Ok you will get this status
<img width="977" height="327" alt="image" src="https://github.com/user-attachments/assets/76f47f9e-6391-4e65-8d6b-c59b9d24867c" />

### KubeConfig Setup

22. `mkdir -p $HOME/.kube`
23. `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
24. `sudo chown $(id -u):$(id -g) $HOME/.kube/config`
25. `export KUBECONFIG=/etc/kubernetes/admin.conf`

### Install Flannel Network Plugin

Flannel is an open-source Container Network Interface (CNI) plugin that creates a virtual network for containers.

26. `kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml`

# We are done with the Master Node

<img width="508" height="53" alt="image" src="https://github.com/user-attachments/assets/a2d90d47-e768-4626-b5e8-9318dcebe6ef" />

# All the name spaces status should be running 
<img width="773" height="177" alt="image" src="https://github.com/user-attachments/assets/24fef156-6046-4b6f-8b85-221327a074dc" />

# Print your Token to join worker node
kubeadm token create --print-join-command

### Worker Node
# use all the step from 1 to 21 and then use kube admin join

Just like we did install Containerd, Kubeadm, Kubelet and Kubectl on the master node, we have to install these packages on the worker node as well. There is an important point to remember that we don't have to run kubeadm init on the worker node, instead we have to run kubeadm join to join the worker node with the master node to create a Kubernetes cluster. The command should look like:

`kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>`
# Node after join Master
<img width="933" height="303" alt="image" src="https://github.com/user-attachments/assets/dc523fb2-effb-4bb7-85b3-fad99fa0964d" />

# we can also verify on Master

<img width="481" height="69" alt="image" src="https://github.com/user-attachments/assets/c9104135-4b81-4d2f-bd19-516401daedd3" />



# verify all service of master node running state
kubectl get pods --all-namespaces

# Install CNI for virtual network for kubernetes only for master nodes
# Calico manifest (example Calico or Flannel)
1. kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml (full-featured networking + network security polices for 1000+ nodes)
Or for 
2. Flannel: kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml (lightweight for local use for 100 nodes)
# wait until CNI pods are Running
kubectl get pods -n kube-system -w
# Now check nodes status
kubectl get nodes
kubectl get pods -A

# join Nodes Repeat above steps except kubeadm init because only Master will use kubeadm init worker will use kubeadm join after installing all packege use kubejoin
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

#Only in case of you need Multi Master Node
------------------------------------------------------------------------------------------------
# 🧠  1. Concept: What you want = “HA Control Plane”

In Kubernetes, “master node” = “control plane node”.

A Highly Available Control Plane means:

You have multiple control-plane nodes (e.g. 2 or 3)

They share the same API server endpoint (VIP or LoadBalancer)

If one master goes down, another continues serving requests automatically

When the primary comes back, it rejoins the cluster and syncs back normally

# 🧩 2. Components involved
Component	Function
kube-apiserver	Exposes Kubernetes API (6443 port). Needs to stay available.
etcd	Stores cluster state. In HA, runs on all control-plane nodes.
kube-controller-manager / scheduler	Run on all masters, only one active at a time.
Keepalived or LoadBalancer (HAProxy, Nginx, cloud LB)	Provides a single Virtual IP (VIP) shared by all masters.
kubeadm HA mode	Helps bootstrap multiple control planes.
🏗️ 3. Typical Architecture (Bare-metal)
        +-----------------------+
        |     Virtual IP (VIP)  |  <-- Provided by Keepalived / HAProxy
        |   192.168.1.100:6443  |
        +-----------+-----------+
                    |
   +----------------+----------------+
   |                |                |
+--+--+          +--+--+          +--+--+
| CP1 |          | CP2 |          | CP3 |
| (Master 1)     | (Master 2)     | (Master 3)
| kube-apiserver | kube-apiserver | kube-apiserver
| etcd           | etcd           | etcd
+----------------+----------------+----------------+


# All your workers connect to the VIP (192.168.1.100:6443) instead of an individual master.

# ⚙️ 4. Setup Steps (example: 2 control planes + 1 worker)

Let’s assume:

Node	Role	IP
node1	master-1	192.168.1.10
node2	master-2	192.168.1.11
node3	worker	192.168.1.12
VIP	shared via Keepalived	192.168.1.100
Step 1️⃣: Setup Keepalived + HAProxy (for VIP)

# Install on both node1 and node2:

sudo apt install -y keepalived haproxy

/etc/haproxy/haproxy.cfg
frontend kubernetes_api_frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes_api_backend

backend kubernetes_api_backend
    mode tcp
    option tcp-check
    balance roundrobin
    server master1 192.168.1.10:6443 check fall 3 rise 2
    server master2 192.168.1.11:6443 check fall 3 rise 2


Enable + restart HAProxy:

sudo systemctl enable haproxy
sudo systemctl restart haproxy

/etc/keepalived/keepalived.conf

On node1 (MASTER):

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.1.100/24
    }
}


On node2 (BACKUP):

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.1.100/24
    }
}


Restart:

sudo systemctl restart keepalived


# ✅ Now 192.168.1.100 will float between node1 and node2 automatically.

Step 2️⃣: Initialize first control-plane

On node1:

sudo kubeadm init \
  --control-plane-endpoint "192.168.1.100:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16


When it finishes, it will output a join command — copy both:

Worker join

Control-plane join (with --control-plane and --certificate-key)

Setup kubectl on node1:

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


# Install network plugin (Flannel or Calico):

1.kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
OR
2.kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Step 3️⃣: Join the 2nd control-plane (node2)

On node2, run the command printed from kubeadm init, e.g.:

sudo kubeadm join 192.168.1.100:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <key>

Step 4️⃣: Join worker nodes

On node3 (and others):

sudo kubeadm join 192.168.1.100:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

Step 5️⃣: Verify cluster

On node1:

kubectl get nodes
kubectl get pods -A


If node1 (master) fails:

Keepalived VIP automatically moves to node2

HAProxy on node2 keeps API running

Cluster keeps working

When node1 comes back:

It rejoins automatically as secondary control-plane.

✅ You now have an automatic failover master system.

⚡ 5. Summary
Feature	Behavior
Primary Master Down	Secondary takes VIP and control-plane continues
Primary Comes Back	Rejoins cluster, becomes secondary again
API Endpoint	Always reachable via VIP (192.168.1.100:6443)
etcd	Replicated automatically between masters
Scheduler & Controller	Run on both, leader elected automatically
