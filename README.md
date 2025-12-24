# วิธีติดตั้ง Kubernetes บน Unbutu server 24.04LTS

การติดตั้งนี้มี Server จำนวน 2 เครื่อง ดังนี้
1. Master node = 192.168.1.200
2. Worker node = 192.168.1.201

1. เตรียมระบบพื้นฐาน (ทุก Node)
Kubernetes ต้องการการตั้งค่าระบบที่เฉพาะเจาะจงเพื่อให้เน็ตเวิร์กและหน่วยความจำทำงานได้ถูกต้อง

1.1. ปิด Swap (จำเป็นมาก):

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

1.2. โหลด Kernel Modules ที่จำเป็น:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

1.3 ตั้งค่า Network Forwarding:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

2. ติดตั้ง Docker Engine (ทุก Node)
เราจะติดตั้ง Docker เพื่อเป็น Container Runtime หลัก
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

3. ติดตั้ง `cri-dockerd` (ทุก Node)
- ดาวน์โหลด Binary
```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.16/cri-dockerd-0.3.16.amd64.tgz
tar -xvf cri-dockerd-0.3.16.amd64.tgz
```
- ย้ายไปที่ /usr/local/bin
```bash
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/
```
- ดาวน์โหลดไฟล์ Service จากต้นฉบับ
```bash
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.service cri-docker.socket /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
```
- เริ่มระบบ
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.socket
```

4. ติดตั้ง Kubeadm, Kubelet และ Kubectl (ทุก Node)
```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg

# เพิ่ม Kubernetes GPG Key และ Repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# ติดตั้งเครื่องมือ
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
