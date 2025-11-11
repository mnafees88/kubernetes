# UBUNTU
step 1 to disabled swap memory (because Kubernetes relies on predictable memory usage so that Kubernetes should have actual memory )
sudo swapoff -a
sed -i '/swap/d' /etc/fstab 

# Update Ubuntu 

1. sudo apt-get update
2. sudo apt install apt-transport-https curl
3. sudo mkdir -p /etc/apt/keyrings
4. curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
5. echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

6.sudo apt update
7.sudo apt install -y containerd.io


# configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# ensure Systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

# verify container service running fine
systemctl status containerd.service
  
# Install kubeadm, kubelet, kubectl (same on all nodes)
1.sudo apt update
2.sudo apt install -y apt-transport-https ca-certificates curl
3.curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg  https://packages.cloud.google.com/apt/doc/apt-key.gpg
4.echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
5.sudo apt install -y kubelet kubeadm kubectl
6.sudo apt-mark hold kubelet kubeadm kubectl
7.modprob br_netfilter
# start kubeadmin if you want to create more than 255 node than use cidr as per your requirement
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
# after above command completes succesfuly Output will give you kubeadm join ... command for worker (copy it). If you missed it, on control-plane run below command any time
kubeadm token create --print-join-command

# Configure kubectl for your user on all 
1.mkdir -p $HOME/.kube
2.sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
3.sudo chown $(id -u):$(id -g) $HOME/.kube/config
4.export kubeconfig=/etc/kubernetes/admin.conf

# verify all service of master node running state
kubectl get pods --all-namespaces

# Install CNI for virtual network for kubernetes only for master nodes
# Calico manifest (example Calico or Flannel)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml (full-featured networking + network security polices for 1000+ nodes)
Or for 
Flannel: kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml (lightweight for local use for 100 nodes)
# wait until CNI pods are Running
kubectl get pods -n kube-system -w
# Now check nodes status
kubectl get nodes
kubectl get pods -A

# join Nodes Repeat above steps except kubeadm init because only Master will use kubeadm init worker will use kubeadm join after installing all packege use kubejoin
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>


