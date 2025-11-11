UBUNTU
step 1 to disabled swap memory (because Kubernetes relies on predictable memory usage so that Kubernetes should have actual memory )
sudo swapoff -a
sed -i '/swap/d' /etc/fstab 

Update Ubuntu 

1. sudo apt-get update
2. sudo apt install apt-transport-https curl
3. sudo mkdir -p /etc/apt/keyrings
4. curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
5. echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y containerd.io


# configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# ensure Systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

#verify container service running fine
systemctl status containerd.service

#Install kubeadm, kubelet, kubectl (same on all nodes)
