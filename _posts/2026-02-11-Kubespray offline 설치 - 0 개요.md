---
layout: post
tags:
  - Ansible
  - k8s
  - kubernetes
  - kubespray
title: "[K8S Deploy Study by Gasida] - Kubespray offline 설치 - 0 개요"
---

# 1. 왜 kubespray Offline인가

일반적인 Kubernetes 설치는 인터넷이 연결된 환경을 전제로 한다. 그러나 실제 기업 환경에서는 다음과 같은 구조를 가진다.

```mermaid
graph LR
    Internet --> DMZ
    DMZ --> Internal
    Internal --> K8sCluster

```

- 내부망에서는 외부 인터넷 접속이 불가능
- 필요시 방화벽 정책 승인 후 Bastion을 통해 접근
- 모든 패키지와 이미지가 사전 승인 되어야 한다.

 이러한 환경을 폐쇄망(Air Gap)이라하며 Kubespray Offline은 폐쇄망에서 쿠버네티스가 동작하기 위한 모든 공급망을 구축하는 과정이다.




# 2. 폐쇄망에서 쿠버네티스 운영에 필수 구성 요소

| 구성요소                           | 용도             | 예시 솔루션                  |
| ------------------------------ | -------------- | ----------------------- |
| **NTP Server**                 | 시간 동기화         | 어플라이언스 장비 (HA 구성)       |
| **DNS Server**                 | 도메인 네임 서비스     | 어플라이언스 장비 (HA 구성)       |
| **Network Gateway**            | 내부/외부망 통신      | 라우터/스위치                 |
| **Local YUM/DNF Repo**         | Linux 패키지 미러링  | reposync + createrepo   |
| **Private Container Registry** | 컨테이너 이미지 저장소   | Harbor, Docker Registry |
| **Helm Artifact Repo**         | Helm 차트 저장소    | ChartMuseum, Zot (OCI)  |
| **Private PyPI Mirror**        | Python 패키지 미러링 | Devpi                   |
| **Private Go Module Proxy**    | Go 모듈 프록시      | Athens                  |

# 3. 실습 환경 

#### 구성
- admin (인터넷 가능)
- k8s-node1(Control Plane)
- k8s-node2(Worker)
- VirtualBox + Vagrant 기반



```bash
# Base Image  https://portal.cloud.hashicorp.com/vagrant/discover/bento/rockylinux-10.0
BOX_IMAGE = "bento/rockylinux-10.0" # "bento/rockylinux-9"
BOX_VERSION = "202510.26.0"
N = 2 # max number of Node

Vagrant.configure("2") do |config|

# Nodes 
  (1..N).each do |i|
    config.vm.define "k8s-node#{i}" do |subconfig|
      subconfig.vm.box = BOX_IMAGE
      subconfig.vm.box_version = BOX_VERSION
      subconfig.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--groups", "/Kubespary-offline-Lab"]
        vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
        vb.name = "k8s-node#{i}"
        vb.cpus = 4
        vb.memory = 2048
        vb.linked_clone = true
      end
      subconfig.vm.host_name = "k8s-node#{i}"
      subconfig.vm.network "private_network", ip: "192.168.10.1#{i}"
      subconfig.vm.network "forwarded_port", guest: 22, host: "6000#{i}", auto_correct: true, id: "ssh"
      subconfig.vm.synced_folder "./", "/vagrant", disabled: true
      subconfig.vm.provision "shell", path: "init_cfg.sh" , args: [ N ]
    end
  end

# Admin Node
    config.vm.define "admin" do |subconfig|
      subconfig.vm.box = BOX_IMAGE
      subconfig.vm.box_version = BOX_VERSION
      subconfig.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--groups", "/Kubespary-offline-Lab"]
        vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
        vb.name = "admin"
        vb.cpus = 4
        vb.memory = 2048
        vb.linked_clone = true
      end
      subconfig.vm.host_name = "admin"
      subconfig.vm.network "private_network", ip: "192.168.10.10"
      subconfig.vm.network "forwarded_port", guest: 22, host: "60000", auto_correct: true, id: "ssh"
      subconfig.vm.synced_folder "./", "/vagrant", disabled: true
      subconfig.vm.disk :disk, size: "120GB", primary: true # https://developer.hashicorp.com/vagrant/docs/disks/usage
      subconfig.vm.provision "shell", path: "admin.sh" , args: [ N ]
    end

end
```



- `admin.sh`

```bash
#!/usr/bin/env bash

echo ">>>> Initial Config Start <<<<"


echo "[TASK 1] Change Timezone and Enable NTP"
timedatectl set-local-rtc 0
timedatectl set-timezone Asia/Seoul


echo "[TASK 2] Disable firewalld and selinux"
systemctl disable --now firewalld >/dev/null 2>&1
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config


echo "[TASK 3] Setting Local DNS Using Hosts file"
sed -i '/^127\.0\.\(1\|2\)\.1/d' /etc/hosts
echo "192.168.10.10 admin" >> /etc/hosts
for (( i=1; i<=$1; i++  )); do echo "192.168.10.1$i k8s-node$i" >> /etc/hosts; done


echo "[TASK 4] Delete default routing - enp0s9 NIC" # setenforce 0 설정 필요
nmcli connection modify enp0s9 ipv4.never-default yes
nmcli connection up enp0s9 >/dev/null 2>&1


echo "[TASK 5] Config net.ipv4.ip_forward"
cat << EOF > /etc/sysctl.d/99-ipforward.conf
net.ipv4.ip_forward = 1
EOF
sysctl --system  >/dev/null 2>&1


echo "[TASK 6] Install packages"
dnf install -y python3-pip git sshpass cloud-utils-growpart >/dev/null 2>&1


echo "[TASK 7] Install Helm"
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | DESIRED_VERSION=v3.20.0 bash >/dev/null 2>&1


echo "[TASK 8] Increase Disk Size"
growpart /dev/sda 3 >/dev/null 2>&1 # lsblk
xfs_growfs /dev/sda3 >/dev/null 2>&1 # df -hT /


echo "[TASK 9] Setting SSHD"
echo "root:qwe123" | chpasswd

cat << EOF >> /etc/ssh/sshd_config
PermitRootLogin yes
PasswordAuthentication yes
EOF
systemctl restart sshd >/dev/null 2>&1


echo "[TASK 10] Setting SSH Key"
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa >/dev/null 2>&1
sshpass -p 'qwe123' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.10.10  >/dev/null 2>&1  # cat /root/.ssh/authorized_keys
ssh -o StrictHostKeyChecking=no root@admin-lb hostname >/dev/null 2>&1
for (( i=1; i<=$1; i++  )); do sshpass -p 'qwe123' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.10.1$i >/dev/null 2>&1 ; done
for (( i=1; i<=$1; i++  )); do sshpass -p 'qwe123' ssh -o StrictHostKeyChecking=no root@k8s-node$i hostname >/dev/null 2>&1 ; done


echo "[TASK 11] Install K9s"
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
wget -P /tmp https://github.com/derailed/k9s/releases/latest/download/k9s_linux_${CLI_ARCH}.tar.gz  >/dev/null 2>&1
tar -xzf /tmp/k9s_linux_${CLI_ARCH}.tar.gz -C /tmp
chown root:root /tmp/k9s
mv /tmp/k9s /usr/local/bin/
chmod +x /usr/local/bin/k9s


echo "[TASK 12] ETC"
echo "sudo su -" >> /home/vagrant/.bashrc


echo ">>>> Initial Config End <<<<"

```


- `init_cfg.sh`

```
#!/usr/bin/env bash

echo ">>>> Initial Config Start <<<<"


echo "[TASK 1] Change Timezone and Enable NTP"
timedatectl set-local-rtc 0
timedatectl set-timezone Asia/Seoul


echo "[TASK 2] Disable firewalld and selinux"
systemctl disable --now firewalld >/dev/null 2>&1
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config


echo "[TASK 3] Disable and turn off SWAP & Delete swap partitions"
swapoff -a
sed -i '/swap/d' /etc/fstab
sfdisk --delete /dev/sda 2 >/dev/null 2>&1
partprobe /dev/sda >/dev/null 2>&1


echo "[TASK 4] Config kernel & module"
cat << EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
vxlan
EOF
modprobe overlay >/dev/null 2>&1
modprobe br_netfilter >/dev/null 2>&1

cat << EOF >/etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system >/dev/null 2>&1


echo "[TASK 5] Setting Local DNS Using Hosts file"
sed -i '/^127\.0\.\(1\|2\)\.1/d' /etc/hosts
echo "192.168.10.10 admin" >> /etc/hosts
for (( i=1; i<=$1; i++  )); do echo "192.168.10.1$i k8s-node$i" >> /etc/hosts; done


echo "[TASK 6] Delete default routing - enp0s9 NIC" # setenforce 0 설정 필요
nmcli connection modify enp0s9 ipv4.never-default yes
nmcli connection up enp0s9 >/dev/null 2>&1


echo "[TASK 7] Setting SSHD"
echo "root:qwe123" | chpasswd

cat << EOF >> /etc/ssh/sshd_config
PermitRootLogin yes
PasswordAuthentication yes
EOF
systemctl restart sshd >/dev/null 2>&1


echo "[TASK 8] Install packages"
dnf install -y python3-pip git >/dev/null 2>&1


echo "[TASK 9] ETC"
echo "sudo su -" >> /home/vagrant/.bashrc


echo ">>>> Initial Config End <<<<"

```



```bash
mkdir k8s-offline
cd k8s-offline

curl -O https://raw.githubusercontent.com/gasida/vagrant-lab/refs/heads/main/k8s-kubespary-offline/Vagrantfile
curl -O https://raw.githubusercontent.com/gasida/vagrant-lab/refs/heads/main/k8s-kubespary-offline/admin.sh
curl -O https://raw.githubusercontent.com/gasida/vagrant-lab/refs/heads/main/k8s-kubespary-offline/init_cfg.sh

vagrant up
vagrant status

# admin, k8s-node1, k8s-node2 각각 접속 : 호스트 OS에 sshpass가 없을 경우 ssh로 root로 접속 후 암호 qwe123 입력
sshpass -p 'qwe123' ssh root@192.168.10.10 # ssh root@192.168.10.10
sshpass -p 'qwe123' ssh root@192.168.10.11 # ssh root@192.168.10.11
sshpass -p 'qwe123' ssh root@192.168.10.12 # ssh root@192.168.10.12
```