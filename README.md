# How-to-Install-a-Kubernetes-HA-Cluster-with-HAProxy-Load-Balancer-Step-by-Step-Guide
How to Install a Kubernetes HA Cluster with HAProxy Load Balancer | Step-by-Step Guide

<pre>

  apt-get update
apt-get upgrade
apt-get install sudo -y
usermod -aG sudo Linux
vim /etc/hosts
10.0.0.1  prod-k8s-master01
10.0.0.2  prod-k8s-master01
10.0.0.3  prod-k8s-master03
10.0.0.4  prod-k8s-worker01
10.0.0.5  prod-k8s-worker02
10.0.0.6  prod-k8s-worker03
10.0.0.7  prod-k8s-lb01

sudo nano /etc/fstab
comment swap line
reboot

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call- iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4. ip_forward
EOF

sudo sysctl --system

lsmod | grep overlay
lsmod | br_netfilter

sudo timedatectl set-timezone Asia/Kolkata
sudo apt-get install chrony
sudo systemctl start chrony
sudo systemctl enable chrony

sudo timedatectl

sudo apt-get install wget curl -y
wget -q --show-progress --https-only \
https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.31.1/crictl-v1.31.1-linux-amd64.tar.gz \
https://github.com/opencontainers/runc/releases/download/v1.1.14/runc.amd64 \
https://github.com/containerd/containerd/releases/download/v1.7.22/containerd-1.7.22-linux-amd64.tar.gz \
https://github.com/containernetworking/plugins/releases/download/v1.5.1/cni-plugins-linux-amd64-v1.5.1.tgz

sudo tar zxvf crictl-v1.31.1-linux-amd64.tar.gz -c /usr/local/bin 
sudo tar Czxvf /usr/local containerd-1.7.22-linux-amd64.tar.gz
wget -q https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

sudo mv containerd.service /usr/lib/systemd/system/
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
sudo mkdir -p /etc/containerd/

containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo sed -i 's! BinaryName = ""! BinaryName = "/usr/local/sbin/runc" !g' /etc/containerd/config.toml

sudo tar Cxzvf /opt/cni/bin/cni-plugins-linux-amd64-v1.5.1.tgz
sudo systemctl daemon-reload
sudo systemctl enable containerd.service
sudo systemctl start containerd.service
sudo systemctl status containerd.service

cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/container/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
EOF


sudo crictl ps

sudo apt-get install -y apt-transport-https ca-certificates gpg

curl -fsSL https://pkgs.k8s.io/core: /stable: /v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg1 https://pkgs.k8s.io/core: /stable: /v1.29/deb/ / | sudo tee /etc/apt/sources.list.d/kubernetes.list deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb//

sudo apt-get update

sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

--------------------------------------Setup Load Balancer ------------------------

apt-get install -y haproxy

service haproxy restart


--------------------------------------End Load Balancer ------------------------

--------------------------------------Setup Masternode-1 ------------------------

sudo kubeadm init --control-plane-endpoint=10.0.0.1 --upload-certs

MetalLB_RTAG=$(curl -s https://api.github.com/repos/metallb/metallb/releases/latest|grep tag_name | cut -d " -f 4| sed 's/v//')
echo $MetalLB_RTAG
mkdir ~/metallb & cd ~/metallb
wget https://raw.githubusercontent.com/metallb/metallb/v$MetalLB_RTAG/config/manifests/metallb-native.yaml
kubectl apply -f metallb-native.yaml

vim ipaddress_pools.yaml

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production
  namespace: metallb-system
spec:
  # Production services will go here. Public IPs are expensive, so we leased
  # just 4 of them.
  addresses:
  - 42.176.25.64/30
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: external
  namespace: metallb-system
spec:
  ipAddressPools:
  - production


vim demo.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
  namespace: web
spec:
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: httpd
        image: httpd:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-server-service
  namespace: web
spec:
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer


kubectl apply -f demo.yaml

kubectl get svc -n web

--------------------------------------End Load Balancer ------------------------


kubectl label node prod-k8s-worker01 prod-k8s-worker02 prod-k8s-worker03 node-role.kubernetes.io/worker=worker

</pre>
