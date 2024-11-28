# วิธีติดตั้ง Kubernetes บน Unbutu server 24.04LTS

## 1. ทำการ Update/upgrade unbutu
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apt-transport-https curl -y
```

## 2. ติดตั้ง Contianerd
```bash
sudo apt install containerd -y
```

## 3. ตั้งค่า Continered
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

## 4. ติดตั้ง kubernetes components
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 5. Disable swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 6. ทำการ Load necessary kernel modules
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

## 7. ทำการตั้งค่า Parameter สำหรับ sysctl
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
จากนั้นรันคำสั่ง
```bash
sudo sysctl --system
```

## 8. เริ่มต้นการทำงาน Cluster (รันเฉพาะ master node):
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

## 9. ตั้งค่า kubeconfig สำหรับ User
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
จากนั้นมองหาประโยค `Alternatively, if you are the root user, you can run` แล้วรันตามคำสั่งที่ระบบระบุมาให้ตัวอย่างเช่น
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```
และทำการ copy คำสั่ง join cluster เก็บไว้ โดยมองหาประโยค `Then you can join any number of worker nodes by running the following on each as root:` ดังตั้วอย่างต่อไปนี้
```bash
kubeadm join 172.31.19.36:6443 --token 922x9d.v0jn4c8he0s286js \
          --discovery-token-ca-cert-hash sha256:8897fd8eb97f2ea0686ccf7507f287ffffd5cf681496fb324940330561c80e4c
```
โดยเก็บไว้อย่าให้หายเพื่อใช้ในการ join cluster

## 10. ติดตั้ง Flannel network plugin (รันเฉพาะ master node)
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

## 11. ทำการ Verify การติดตั้ง
```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

## 12. ทำการ Join worker nodes กับ Cluster
- เริ่มติดตั้ง Kubernetes ใน Worker node ตามลำดับ
- ใช้คำสั่ง `kubeadm join` เพื่อให้รองรับคำสั่ง `kubeadm init` แสดงผลออกบน master node
- โดยจะต้องรันคำสั่งด้วย `root`
```bash
kubeadm join 172.31.19.36:6443 --token 922x9d.v0jn4c8he0s286js --discovery-token-ca-cert-hash sha256:8897fd8eb97f2ea0686ccf7507f287ffffd5cf681496fb324940330561c80e4c
```
