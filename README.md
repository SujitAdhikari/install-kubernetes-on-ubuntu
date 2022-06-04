# Install-kubernetes-on-ubuntu

IP configuration:
vi /etc/netplan/00-installer-config.yaml

network:
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.10.200/24]
      gateway4: 192.168.10.100
      nameservers:
        addresses: [192.168.10.100]

### Command active network
$ netplan apply

Set Hostname:
master: hostnamectl set-hostname master.example.com
worker1: hostnamectl set-hostname worker1.example.com
worker2: hostnamectl set-hostname worker3.example.com
worker3: hostnamectl set-hostname worker3.example.com

Hosts file entry:
-------------------
vim /etc/hosts
192.168.10.200 master.example.com master
192.168.10.201 worker1.example.com worker1
192.168.10.202 worker3.example.com worker2
192.168.10.203 worker3.example.com worker3

##Permanently disable swap space from fstab: 
#/swap.img      none    swap    sw      0       0

##runtime not recommended 
swapoff -a #runtime 

Master Node Prepare:
----------------------
##Run below Script>>sh install.sh:

script:
---
#!/bin/sh
#Disable Swap Space
sudo swapoff –a

#Add these IP tables Rule:

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

#Installing docker:
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
sudo apt update
sudo apt install docker-ce -y
systemctl start docker

cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "graph": "/mnt/docker-data",
  "storage-driver": "overlay",
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

systemctl enable docker
systemctl daemon-reload


#Installing Kubernetes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt-get install kubeadm kubelet kubectl -y
---

# rm -f /etc/containerd/config.toml
# systemctl restart contaninerd

root@master:~# kubeadm init 


- Login with normal user execute provided command.
- Notedown join command for workernode

#CNI Plugin: Every node have kube proxy, CNI Plugin is like router, CNI plugin will provide IP. 
#Core DNS is not getting IP so it is not ready state. 

Weave net not working, need to check:
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

Install calico as CNI Plugin:
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml

2. Prepare worker node:
  a. execute command: sh install.sh #don't execute command kubeadm init for workernode
  b. Execute join command
  c. Print Join command from master node:
       kubeadm token create  --print-join-command
       Execute Join command 
  
Basic command:
 
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get pods -n kube-system

Pods:
kubectl run nginx --image=nginx
kubectl get pods -o wide # check pods running on which node

sudo apt-mark hold kubelet kubeadm kubectl  #Lock the version, So it will not udate if we run apt upgrade

Troubleshooting:
#if need to reset for any error on master node.>>
kubeadm reset -f
rm -rf /etc/kubernetes/
rm -fr /var/lib/kubelet/

Remove CNI Plugin:
Remove Weave:
--------------
kubectl delete -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
Need to check below 4 line:
---------------------------
rm -f /opt/cni/bin/weave*
rm -f /etc/cni/net.d/*weave*
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ipvsadm -C

kubectl describe pods weave-net-9h9ll -n kube-system
kubectl logs <weave-pod-name-as-above> -n kube-system weave-npc

kubectl get svc
kebectl delete svc pod-name

Cluster	IP:
-----------
kubectl expose pod nginx --name=web-cip --type=ClusterIP --port=80
kubectl get svc
kubectl get pods -o wide # podip
kubectl describe svc web-cip  #endpoint is all pods ip
kubectl describe pods nginx # Labels

Networks:
Pod Network
Service Network
Node Network
Load Balancer Network

imperative 
declarative

NodePort:
kubectl expose pod nginx --name=web-np --type=NodePort --port=80  >>listen worker and master ip node port 

kubectl expose pod nginx --name=web-lb --type=LoadBalancer --port=80  >>listen worker and master ip node port # here external ip will get from external load balancer(metallb, ELB)

rs:
kubectl create deploy nginx --image=nginx --replica=3

Declarative file create using imperative way:
-----------------------------------------------
kubectl run nginx --image=nginx --dry-run=client -o yaml > nginx.yaml  # pod
kubectl create deploy redis --image=redis --dry-run=client -o yaml > nginx.yaml  # deployment

Online Platform for kubernetes:
-------------------------------
Kubernetes playground

DaemonSets: Every node run 1 pod

Deployment: We will able to set replica, If any pod die, create pod again.
kubectl create deploy nginx --image=nginx
kubectl get rs
kubectl delete deploy nginx

Service : Establish communication in internal cluster and external 
Service Type: 
	Cluster IP: Within cluster able to communicate like app pod to db pod>> etc/k8s/manifests/kube-apiserver.yml
	Node Port: Run on node(server) port from 30000+, will listen same port on every node, We will unable to set dns name here
	Load Balancer: We will get an IP for dns enrty with lb like, AWS (ELB)
