This documentation guides you in setting up a cluster with 3 master nodes, one worker node and a load balancer node using HAProxy

## Bring up all the virtual machines
```
vagrant up
```

## Set up load balancer node
```
vagrant ssh loadbalancer
```

##### Install Haproxy
```
sudo apt update 
sudo apt install -y haproxy
```
##### Configure haproxy
Append the below lines to **/etc/haproxy/haproxy.cfg**

```
sudo nano /etc/haproxy/haproxy.cfg
```

```
frontend kubernetes-frontend
    bind 172.16.16.100:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server kmaster1 172.16.16.101:6443 check fall 3 rise 2
    server kmaster2 172.16.16.102:6443 check fall 3 rise 2
    server kmaster3 172.16.16.103:6443 check fall 3 rise 2
```
##### Restart haproxy service
```
sudo systemctl restart haproxy
```

## On all kubernetes nodes (kmaster1, kmaster2,kmaster3, kworker1)
##### Disable Firewall
```
sudo ufw disable
```
##### Disable swap
```
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```
##### Update sysctl settings for Kubernetes networking
```
sudo cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
##### Install docker engine
```
  sudo apt install -y sudo apt-transport-https ca-certificates curl gnupg-agent software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  sudo  apt update && sudo apt install -y docker-ce=5:19.03.10~3-0~ubuntu-focal containerd.io
```
### Kubernetes Setup
##### Add Apt repository
```
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
  sudo echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
}
```
##### Install Kubernetes components
```
sudo apt update && sudo apt install -y kubeadm=1.19.2-00 kubelet=1.19.2-00 kubectl=1.19.2-00
```
## On any one of the Kubernetes master node (Eg: kmaster1)
##### Initialize Kubernetes Cluster
```
sudo kubeadm init --control-plane-endpoint="172.16.16.100:6443" --upload-certs --apiserver-advertise-address=172.16.16.101 --pod-network-cidr=192.168.0.0/16
```
Copy the commands to join other master nodes and worker nodes.
##### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml
```

## Join other nodes to the cluster 
> Use the respective kubeadm join commands you copied from the output of kubeadm init command on the first master.

> IMPORTANT: You also need to pass --apiserver-advertise-address to the join command when you join the other master node.
