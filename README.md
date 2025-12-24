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
