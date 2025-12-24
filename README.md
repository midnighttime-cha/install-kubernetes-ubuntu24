# วิธีติดตั้ง Kubernetes บน Unbutu server 24.04LTS

การติดตั้งนี้มี Server จำนวน 2 เครื่อง ดังนี้
1. Master node = 192.168.1.200
2. Worker node = 192.168.1.201

## 1. Config OS (ทำทั้ง master & worker nodes)
ทำการอัพเดทระบบ
```bash
sudo apt update && sudo apt upgrade -y
```

ทำการ Disable swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
จากนั้น Add Kernel Parameters
```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```
ตั้งค่า kernel parameters
```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## 2. ติดตั้ง Docker (ทำทั้ง master & worker nodes)
ติดตั้ง Docker’s apt repository 
- Add Docker's official GPG key:
```bash
sudo apt update
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```
- Add the repository to Apt sources:
```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
จากนั้นทำการติดตั้ง ด้วยคำสั่ง
```bash
sudo apt update
sudo apt install -y containerd.io
```

## 3. ติดตั้ง kubeadm, kubectl, kubelet(ทำทั้ง master & worker nodes)
ตั้งค่า containerd
```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
เพิ่ม Apt repository ของ kubernetes
```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```


จากนั้นทำการ ติดตั้ง kubeadm, kubelet และ kubectl ได้เลย
```bash
sudo apt update 
sudo apt install -y kubelet kubeadm kubectl 
sudo apt-mark hold kubelet kubeadm kubectl
```
หลังจากติดตั้งเสร็จให้ทำการ อัพเดต kubelet เป็น ip ของ node นั้น ๆ โดยแทนค่า <NODE_IP> ตาม ip ของ node นั้น ๆ เช่น 192.168.1.200 ถ้าทำบน master node และ 192.168.1.201 ถ้าทำบน worker node
```bash
echo "KUBELET_EXTRA_ARGS=--node-ip=<NODE_IP>" | sudo tee /etc/default/kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl restart containerd
```

## 4. Setup master node
Setup Master Node (ทำแค่ที่ master node) แทนค่า <MASTER_NODE_IP> เป็น ip ในกรณีนี้คือ 192.168.1.200
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<MASTER_NODE_IP>
```
เมื่อเสร็จแล้วจะมีคำสั่งแสดงขึ้นมาดังต่อไปนี้
```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf
You should now deploy a pod network to the cluster. Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at: https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities and service account keys on each node and then running the following as root:

  kubeadm join 192.168.1.200:6443 --token x06xhb.sdfsdfsdfsdf \
        --discovery-token-ca-cert-hash sha256:sdfksdflj98sdfd8f0sdf8sd7fsd0f0sdf8xcv7xcv9xcv7vxcv
```
ในเครื่อง master node ให้ copy ต่อไปนี้แล้วรัน
```bash
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
ติดตั้ง Network plugin ให้กับ Cluster
```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
kubectl apply -f calico.yaml
```
หลังจากเสร็จสิ้นให้ทำการตรวจสอบด้วยคำสั่ง
```bash
kubectl get node
```
จะแสดง master node ขึ้นมา เป็นอันเสร็จสิ้นในการติดตั้งฝั่ง master node
```
NAME              STATUS     ROLES           AGE   VERSION
k8s-master-node   Ready      control-plane   24m   v1.28.1
```

## 5. Join worker node เข้า cluster
ส่วนในเครื่อง worker node ให้รันคำสั่งต่อไปนี้ (รันด้วย sudo)
```bash
sudo kubeadm join 192.168.1.200:6443 --token x06xhb.sdfsdfsdfsdf \
        --discovery-token-ca-cert-hash sha256:sdfksdflj98sdfd8f0sdf8sd7fsd0f0sdf8xcv7xcv9xcv7vxcv
```
เมื่อเสร็จสิ้นให้กลับไปตรวจสอบการการ join cluster ที่เครื่อง master node
```bash
kubectl get node
```
จะแสดงเครื่อง master node และเครื่อง worker node ตามตัวอย่างต่อไปนี้
```
NAME              STATUS     ROLES           AGE   VERSION
k8s-master-node   Ready      control-plane   24m   v1.28.1
k8s-woker-01      NotReady   <none>          10s   v1.28.1
```
เป็นอันเสร็จสิ้นการติดตั้ง

## อ้างอิงจาก
- [ตั้ง kubernetes cluster เองแบบลำบากด้วย kubeadm](https://srank123.medium.com/%E0%B8%95%E0%B8%B1%E0%B9%89%E0%B8%87-kubernetes-cluster-%E0%B9%80%E0%B8%AD%E0%B8%87%E0%B9%81%E0%B8%9A%E0%B8%9A%E0%B8%A5%E0%B8%B3%E0%B8%9A%E0%B8%B2%E0%B8%81%E0%B8%94%E0%B9%89%E0%B8%A7%E0%B8%A2-kubeadm-c7c84ba58dc7)
