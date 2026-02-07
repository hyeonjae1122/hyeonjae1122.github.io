---
tags:
  - Ansible
layout: post
title: 2026-02-07-Kubespary HA & Upgrade - 0 실습 환경 배포하기
categories:
---


> 실습환경 배포를 위한 리소스를 리스트 업

- Vagrantfile

```
# Base Image  https://portal.cloud.hashicorp.com/vagrant/discover/bento/rockylinux-10.0
BOX_IMAGE = "bento/rockylinux-10.0" # "bento/rockylinux-9"
BOX_VERSION = "202510.26.0"
N = 5 # max number of Node

Vagrant.configure("2") do |config|

# Nodes 
  (1..M).each do |i|
    config.vm.define "k8s-node#{i}" do |subconfig|
      subconfig.vm.box = BOX_IMAGE
      subconfig.vm.box_version = BOX_VERSION
      subconfig.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--groups", "/Kubespary-Lab"]
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
      subconfig.vm.provision "shell", path: "init_cfg.sh"
    end
  end

# Admin & LoadBalancer Node
    config.vm.define "admin-lb" do |subconfig|
      subconfig.vm.box = BOX_IMAGE
      subconfig.vm.box_version = BOX_VERSION
      subconfig.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--groups", "/Kubespary-Lab"]
        vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
        vb.name = "admin-lb"
        vb.cpus = 2
        vb.memory = 1024
        vb.linked_clone = true
      end
      subconfig.vm.host_name = "admin-lb"
      subconfig.vm.network "private_network", ip: "192.168.10.10"
      subconfig.vm.network "forwarded_port", guest: 22, host: "60000", auto_correct: true, id: "ssh"
      subconfig.vm.synced_folder "./", "/vagrant", disabled: true
      subconfig.vm.provision "shell", path: "admin-lb.sh"
    end

end
```


# admin-lb.sh

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
echo "192.168.10.10 k8s-api-srv.admin-lb.com admin-lb" >> /etc/hosts
for (( i=1; i<=$1; i++  )); do echo "192.168.10.1$i k8s-node$i" >> /etc/hosts; done


echo "[TASK 4] Delete default routing - enp0s9 NIC" # setenforce 0 설정 필요
nmcli connection modify enp0s9 ipv4.never-default yes
nmcli connection up enp0s9 >/dev/null 2>&1


echo "[TASK 5] Install kubectl"
cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubectl
EOF
dnf install -y -q kubectl --disableexcludes=kubernetes >/dev/null 2>&1


echo "[TASK 6] Install HAProxy"
dnf install -y haproxy >/dev/null 2>&1

cat << EOF > /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  tcplog
    option                  dontlognull
    option http-server-close
    #option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

# ---------------------------------------------------------------------
# Kubernetes API Server Load Balancer Configuration
# ---------------------------------------------------------------------
frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    option tcp-check
    option log-health-checks
    timeout client 3h
    timeout server 3h
    balance roundrobin
    server k8s-node1 192.168.10.11:6443 check check-ssl verify none inter 10000
    server k8s-node2 192.168.10.12:6443 check check-ssl verify none inter 10000
    server k8s-node3 192.168.10.13:6443 check check-ssl verify none inter 10000

# ---------------------------------------------------------------------
# HAProxy Stats Dashboard - http://192.168.10.10:9000/haproxy_stats
# ---------------------------------------------------------------------
listen stats
    bind *:9000
    mode http
    stats enable
    stats uri /haproxy_stats
    stats realm HAProxy\ Statistic
    stats admin if TRUE

# ---------------------------------------------------------------------
# Configure the Prometheus exporter - curl http://192.168.10.10:8405/metrics
# ---------------------------------------------------------------------
frontend prometheus
    bind *:8405
    mode http
    http-request use-service prometheus-exporter if { path /metrics }
    no log
EOF
systemctl enable --now haproxy >/dev/null 2>&1


echo "[TASK 7] Install nfs-utils"
dnf install -y nfs-utils >/dev/null 2>&1
systemctl enable --now nfs-server >/dev/null 2>&1
mkdir -p /srv/nfs/share
chown nobody:nobody /srv/nfs/share
chmod 755 /srv/nfs/share
echo '/srv/nfs/share *(rw,async,no_root_squash,no_subtree_check)' > /etc/exports
exportfs -rav


echo "[TASK 8] Install packages"
dnf install -y python3-pip git sshpass >/dev/null 2>&1


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
for (( i=1; i<=$1; i++  )); do sshpass -p 'qwe123' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.10.1$i >/dev/null 2>&1 ; done

ssh -o StrictHostKeyChecking=no root@admin-lb hostname >/dev/null 2>&1
for (( i=1; i<=$1; i++  )); do sshpass -p 'qwe123' ssh -o StrictHostKeyChecking=no root@k8s-node$i hostname >/dev/null 2>&1 ; done


echo "[TASK 11] Clone Kubespray Repository"
git clone -b v2.29.1 https://github.com/kubernetes-sigs/kubespray.git /root/kubespray >/dev/null 2>&1

cp -rfp /root/kubespray/inventory/sample /root/kubespray/inventory/mycluster
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13 etcd_member_name=etcd3

[etcd:children]
kube_control_plane

[kube_node]
k8s-node4 ansible_host=192.168.10.14 ip=192.168.10.14
#k8s-node5 ansible_host=192.168.10.15 ip=192.168.10.15
EOF


echo "[TASK 12] Install Python Dependencies"
pip3 install -r /root/kubespray/requirements.txt >/dev/null 2>&1 # pip3 list


echo "[TASK 13] Install K9s"
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
wget -P /tmp https://github.com/derailed/k9s/releases/latest/download/k9s_linux_${CLI_ARCH}.tar.gz  >/dev/null 2>&1
tar -xzf /tmp/k9s_linux_${CLI_ARCH}.tar.gz -C /tmp
chown root:root /tmp/k9s
mv /tmp/k9s /usr/local/bin/
chmod +x /usr/local/bin/k9s


echo "[TASK 14] Install kubecolor"
dnf install -y -q 'dnf-command(config-manager)' >/dev/null 2>&1
dnf config-manager --add-repo https://kubecolor.github.io/packages/rpm/kubecolor.repo >/dev/null 2>&1
dnf install -y -q kubecolor >/dev/null 2>&1


echo "[TASK 15] Install Helm"
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | DESIRED_VERSION=v3.18.6 bash >/dev/null 2>&1


echo "[TASK 16] ETC"
echo "sudo su -" >> /home/vagrant/.bashrc


echo ">>>> Initial Config End <<<<"

```

# init_cfg.sh

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
echo "192.168.10.10 k8s-api-srv.admin-lb.com admin-lb" >> /etc/hosts
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
dnf install -y git nfs-utils >/dev/null 2>&1


echo "[TASK 9] ETC"
echo "sudo su -" >> /home/vagrant/.bashrc


echo ">>>> Initial Config End <<<<"

```




# 실습환경 배포

```bash
mkdir k8s-ha-kubespary
cd k8s-ha-kubespary

# 파일 다운로드
curl -O https://raw.githubusercontent.com/gasida/vagrant-lab/refs/heads/main/k8s-ha-kubespary/Vagrantfile
curl -O https://raw.githubusercontent.com/gasida/vagrant-lab/refs/heads/main/k8s-ha-kubespary/admin-lb.sh
curl -O https://raw.githubusercontent.com/gasida/vagrant-lab/refs/heads/main/k8s-ha-kubespary/init_cfg.sh
```

- 배포

```
vagrant up
vagrant status
```



# k8s 배포

```bash
# 작업용 inventory 디렉터리 확인
cd /root/kubespray/
git describe --tags
git --no-pager tag

tree inventory/mycluster/

# inventory.ini 확인
cat /root/kubespray/inventory/mycluster/inventory.ini

# 아래 hostvars 에 선언 적용된 값 찾기는 아래 코드 블록 참고
ansible-inventory -i /root/kubespray/inventory/mycluster/inventory.ini --list

ansible-inventory -i /root/kubespray/inventory/mycluster/inventory.ini --graph

# k8s_cluster.yml # for every node in the cluster (not etcd when it's separate)
sed -i 's|kube_owner: kube|kube_owner: root|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|kube_network_plugin: calico|kube_network_plugin: flannel|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|kube_proxy_mode: ipvs|kube_proxy_mode: iptables|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|enable_nodelocaldns: true|enable_nodelocaldns: false|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
grep -iE 'kube_owner|kube_network_plugin:|kube_proxy_mode|enable_nodelocaldns:' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
## coredns autoscaler 미설치
echo "enable_dns_autoscaler: false" >> inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml

# flannel 설정 수정
echo "flannel_interface: enp0s9" >> inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml
grep "^[^#]" inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml

# addons
sed -i 's|metrics_server_enabled: false|metrics_server_enabled: true|g' inventory/mycluster/group_vars/k8s_cluster/addons.yml
grep -iE 'metrics_server_enabled:' inventory/mycluster/group_vars/k8s_cluster/addons.yml
## cat roles/kubernetes-apps/metrics_server/defaults/main.yml                            # 메트릭서버 관련 디폴트 변수 참고
## cat roles/kubernetes-apps/metrics_server/templates/metrics-server-deployment.yaml.j2  # jinja2 템플릿 파일 참고
echo "metrics_server_requests_cpu: 25m"     >> inventory/mycluster/group_vars/k8s_cluster/addons.yml
echo "metrics_server_requests_memory: 16Mi" >> inventory/mycluster/group_vars/k8s_cluster/addons.yml


# 지원 버전 정보 확인
cat roles/kubespray_defaults/vars/main/checksums.yml | grep -i kube -A40

# 배포: 아래처럼 반드시 ~/kubespray 디렉토리에서 ansible-playbook 를 실행하자! 8분 정도 소요
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml --list-tasks # 배포 전, Task 목록 확인
ANSIBLE_FORCE_COLOR=true ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml -e kube_version="1.32.9" | tee kubespray_install.log


# 설치 확인
more kubespray_install.log

# facts 수집 정보 확인
tree /tmp

# local_release_dir: "/tmp/releases" 확인
ssh k8s-node1 tree /tmp/releases
ssh k8s-node4 tree /tmp/releases

# sysctl 적용값 확인
ssh k8s-node1 grep "^[^#]" /etc/sysctl.conf
ssh k8s-node4 grep "^[^#]" /etc/sysctl.conf

# etcd 백업 확인
for i in {1..3}; do echo ">> k8s-node$i <<"; ssh k8s-node$i tree /var/backups; echo; done

# k8s api 호출 확인 : IP, Domain
cat /etc/hosts
for i in {1..3}; do echo ">> k8s-node$i <<"; curl -sk https://192.168.10.1$i:6443/version | grep Version; echo; done
for i in {1..3}; do echo ">> k8s-node$i <<"; curl -sk https://k8s-node$i:6443/version | grep Version; echo; done

# k8s admin 자격증명 확인 : 컨트롤 플레인 노드들은 apiserver 파드가 배치되어 있으니 127.0.0.1:6443 엔드포인트 설정됨
for i in {1..3}; do echo ">> k8s-node$i <<"; ssh k8s-node$i kubectl cluster-info -v=6; echo; done


mkdir /root/.kube
scp k8s-node1:/root/.kube/config /root/.kube/
cat /root/.kube/config | grep server

# API Server 주소를 localhost에서 컨트롤 플레인 1번 node P로 변경 : 1번 node 장애 시, 직접 수동으로 다른 node IP 변경 필요.
kubectl get node -owide -v=6
sed -i 's/127.0.0.1/192.168.10.11/g' /root/.kube/config
혹은 
sed -i 's/127.0.0.1/192.168.10.12/g' /root/.kube/config
sed -i 's/127.0.0.1/192.168.10.13/g' /root/.kube/config


kubectl get node -owide -v=6

ansible-inventory -i /root/kubespray/inventory/mycluster/inventory.ini --graph
kubectl describe node | grep -E 'Name:|Taints'

kubectl get pod -A

# 노드별 파드 CIDR 확인
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'


# etcd 정보 확인 : etcd name 확인
ssh k8s-node1 etcdctl.sh member list -w table

for i in {1..3}; do echo ">> k8s-node$i <<"; ssh k8s-node$i etcdctl.sh endpoint status -w table; echo; done

# k9s 실행
k9s

# 자동완성 및 단축키 설정
source <(kubectl completion bash)
alias k=kubectl
alias kc=kubecolor
complete -F __start_kubectl k
echo 'source <(kubectl completion bash)' >> /etc/profile
echo 'alias k=kubectl' >> /etc/profile
echo 'alias kc=kubecolor' >> /etc/profile
echo 'complete -F __start_kubectl k' >> /etc/profile

```


# 컴포넌트 확인

- 컴포넌트 확인

```bash
# kube-apiserver 정보 확인
ssh k8s-node1 cat /etc/kubernetes/manifests/kube-apiserver.yaml

# kube-controller-manager
ssh k8s-node1 cat /etc/kubernetes/manifests/kube-controller-manager.yaml

#kube-scheduler
ssh k8s-node1 cat /etc/kubernetes/manifests/kube-scheduler.yaml
```

- lease 정보 확인
    - apiserver 는 모두 Active 동작, kcm / scheduler 은 1대만 리더 역할

```bash
kubectl get lease -n kube-system
```


- 인증서 정보 확인

```
for i in {1..3}; do echo ">> k8s-node$i <<"; ssh k8s-node$i ls -l /etc/kubernetes/super-admin.conf ; echo; done


for i in {1..3}; do echo ">> k8s-node$i <<"; ssh k8s-node$i kubeadm certs check-expiration ; echo; done

```


- coredns service IP 확인
    - kubespray는 coredns의 서비스명이 kube-dns가 아니라 coredns이며, clusterIP 가 10.233.0.3

```bash
kubectl exec -it -n kube-system nginx-proxy-k8s-node4 -- cat /etc/resolv.conf

kubectl get svc -n kube-system coredns
kubectl get cm -n kube-system kubelet-config -o yaml | grep clusterDNS -A2
```


- kubeadm 통한 직접 설정 시 : 10.96.0.0/16 대역 중 10번째 IP 사용 확인

```bash
kubectl get svc,ep -n kube-system
```


- kubespray task에 의해서 호스트에서도 서비스명 도메인 질의를 위해 ns 최상단 추가 등 확인

```
ssh k8s-node1 cat /etc/resolv.conf
ssh k8s-node5 cat /etc/resolv.conf
```

-  kubeadm-config cm 정보 확인

```
kubectl get cm -n kube-system kubeadm-config -o yaml
```

- kubeadm 과 동일하게 kubelet node 최초 join 시 CSR 사용 확인

```
kubectl get csr
```