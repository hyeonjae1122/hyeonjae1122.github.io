
1. 리눅스에 VirtualBox 설치하기

```bash
# 저장소 추가
sudo mkdir -p /etc/apt/keyrings
wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor --yes --output /etc/apt/keyrings/oracle-virtualbox2016.gpg

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/oracle-virtualbox2016.gpg] https://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list

# 업데이트 및 설치
sudo apt-get update
sudo apt-get install -y virtualbox-7.0
sudo apt-get install -y virtualbox-ext-pack

# 사용자 추가
sudo usermod -aG vboxusers $USER
```


- 트러블 슈팅

```bash
# IP추가
sudo bash -c 'echo "* 192.168.10.0/24" >> /etc/vbox/networks.conf'
```
	
-  VirtualBox와 KVM 충돌
```bash
# KVM 모듈 언로드 
sudo modprobe -r kvm_intel 
# 또는 AMD CPU인 경우 
sudo modprobe -r kvm_amd 

# 확인 
lsmod | grep kvm 

# 부팅 후에도 자동 언로드 설정 
sudo bash -c 'echo "blacklist kvm_intel" >> /etc/modprobe.d/blacklist.conf'

# 또는 AMD 
sudo bash -c 'echo "blacklist kvm_amd" >> /etc/modprobe.d/blacklist.conf' 

# 재부팅
sudo reboot
```


2. 리눅스에 Vagrant 설치하기
```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install vagrant
```


3. Vagrant로 가상 머신 프로비저닝 하기

```bash
# Rocky Linux용 init_cfg.sh 수정


#!/usr/bin/env bash
echo ">>>> Initial Config Start <<<<"

echo "[TASK 1] Setting Profile & Bashrc"
echo "sudo su -" >> /home/vagrant/.bashrc
echo 'alias vi=vim' >> /etc/profile
ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime # Change Timezone


#########################################
########### SELinux 사용 #################
#########################################
echo "[TASK 2] Disable SELinux (Rocky Linux uses SELinux, not AppArmor)"
setenforce 0 >/dev/null 2>&1
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

echo "[TASK 3] Disable and turn off SWAP"
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab

echo "[TASK 4] Install Packages"
yum update -y -q >/dev/null 2>&1
yum install -y tree git jq yq unzip vim sshpass >/dev/null 2>&1

echo "[TASK 5] Setting Root Password"
echo "root:qwe123" | chpasswd

echo "[TASK 6] Setting Sshd Config"
cat << EOF >> /etc/ssh/sshd_config
PasswordAuthentication yes
PermitRootLogin yes
EOF
systemctl restart sshd >/dev/null 2>&1

echo "[TASK 7] Setting Local DNS Using Hosts file"
sed -i '/^127\.0\.\(1\|2\)\.1/d' /etc/hosts
cat << EOF >> /etc/hosts
192.168.56.10  jumpbox
192.168.56.100 server.kubernetes.local server
192.168.56.101 node-0.kubernetes.local node-0
192.168.56.102 node-1.kubernetes.local node-1
EOF

echo ">>>> Initial Config End <<<<"
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T043413999Z.png)



```bash
# 사용자 확인
whoami
pwd

# OS version 확인
cat /etc/os-release

# SELinux 상태 확인
getenforce
sestatus
cat /etc/selinux/config

# /etc/hosts 파일 내용 확인
cat /etc/hosts

```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T043521888Z.png)


```bash
whoami
root

# Tool Install
dnf install tree git jq yq unzip vim sshpass -y

pwd
git clone --depth 1 https://github.com/kelseyhightower/kubernetes-the-hard-way.git

cd kubernetes-the-hard-way
tree
pwd

# CPU 아키텍처 확인 
uname -m
uname -a

# # Download Kubernetes
# https://www.downloadkubernetes.com/
vi downloads-amd64-latest.txt
# https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-scheduler
# https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubectl
# https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubelet
# https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-apiserver
# https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-controller-manager
# https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-proxy
# https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.34.0/crictl-v1.34.0-linux-amd64.tar.gz
# https://github.com/opencontainers/runc/releases/download/v1.4.0/runc.amd64

 # https://github.com/containernetworking/plugins/releases/download/v1.8.0/cni-plugins-linux-amd64-v1.8.0.tgz

 # https://github.com/containerd/containerd/releases/download/v2.1.5/containerd-2.1.5-linux-amd64.tar.gz

 # https://github.com/etcd-io/etcd/releases/download/v3.5.26/etcd-v3.5.26-linux-amd64.tar.gz

wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-amd64-latest.txt
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T051455703Z.png)

```bash
mkdir -p downloads/{client,cni-plugins,controller,worker}
tree -d downloads

# 압축풀기
tar -xvf downloads/crictl-v1.34.0-linux-amd64.tar.gz \
-C downloads/worker/ && tree -ug downloads

tar -xvf downloads/containerd-2.1.5-linux-amd64.tar.gz --strip-components 1 -C downloads/worker/ && tree -ug downloads

tar -xvf downloads/cni-plugins-linux-amd64-v1.8.0.tgz -C downloads/cni-plugins/ && tree -ug downloads

tar -xvf downloads/etcd-v3.5.26-linux-amd64.tar.gz -C downloads/ --strip-components 1 etcd-v3.5.26-linux-amd64/etcdctl etcd-v3.5.26-linux-amd64/etcd && tree -ug downloads

# 확인
tree downloads/worker/ 
tree downloads/cni-plugins
ls -l downloads/{etcd,etcdctl}



# 파일 이동 
mv downloads/{etcdctl,kubectl} downloads/client/ 
mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} downloads/controller/ 
mv downloads/{kubelet,kube-proxy} downloads/worker/ 
mv downloads/runc.${ARCH} downloads/worker/runc

# 확인 
tree downloads/client/ 
tree downloads/controller/ 
tree downloads/worker/

# 불필요한 압축 파일 제거 
ls -l downloads/*gz 
rm -rf downloads/*gz

# Make the binaries executable. 
ls -l downloads/{client,cni-plugins,controller,worker}/* 
chmod +x downloads/{client,cni-plugins,controller,worker}/* 
ls -l downloads/{client,cni-plugins,controller,worker}/*

# 일부 파일 소유자 변경 
tree -ug downloads # cat /etc/passwd | grep vagrant && cat /etc/group | grep vagrant 
chown root:root downloads/client/etcdctl 
chown root:root downloads/controller/etcd 
chown root:root downloads/worker/crictl 
tree -ug downloads

# kubernetes client 도구인 kubectl를 설치 
ls -l downloads/client/kubectl 
cp downloads/client/kubectl /usr/local/bin/
```


![[2026-01-04-Bootstrap k8s the hard way-18.png]]



## SSH 접속 환경 설정

```bash
# Machine Database (서버 속성 저장 파일) : IPV4_ADDRESS FQDN HOSTNAME POD_SUBNET
## 참고) server(controlplane)는 kubelet 동작하지 않아서, 파드 네트워크 대역 설정 필요 없음
cat <<EOF > machines.txt
192.168.10.100 server.kubernetes.local server
192.168.10.101 node-0.kubernetes.local node-0 10.200.0.0/24
192.168.10.102 node-1.kubernetes.local node-1 10.200.1.0/24
EOF
cat machines.txt

while read IP FQDN HOST SUBNET; do
  echo "${IP} ${FQDN} ${HOST} ${SUBNET}"
done < machines.txt


# Configuring SSH Access 설정
 
# sshd config 설정 파일 확인 : 이미 암호 기반 인증 접속 설정 되어 있음
grep "^[^#]" /etc/ssh/sshd_config
...
PasswordAuthentication yes
PermitRootLogin yes

# Generate a new SSH key
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
ls -l /root/.ssh
-rw------- 1 root root 2602 Jan  2 21:07 id_rsa
-rw-r--r-- 1 root root  566 Jan  2 21:07 id_rsa.pub

# Copy the SSH public key to each machine
while read IP FQDN HOST SUBNET; do
  sshpass -p 'qwe123' ssh-copy-id -o StrictHostKeyChecking=no root@${IP}
done < machines.txt

while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} cat /root/.ssh/authorized_keys
done < machines.txt

# Once each key is added, verify SSH public key access is working
# 아래는 IP 기반으로 접속 확인
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname
done < machines.txt


# Hostnames 설정

# 확인 : init_cfg.sh 로 이미 설정되어 있음
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} cat /etc/hosts
done < machines.txt

while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname --fqdn
done < machines.txt

# 아래는 hostname 으로 ssh 접속 확인
cat /etc/hosts
while read IP FQDN HOST SUBNET; do
  sshpass -p 'qwe123' ssh -n -o StrictHostKeyChecking=no root@${HOST} hostname
done < machines.txt

while read IP FQDN HOST SUBNET; do
  sshpass -p 'qwe123' ssh -n root@${HOST} uname -o -m -n
done < machines.txt

```


# CA 설정 및 TLS 인증서 생성

```
[req]
distinguished_name = req_distinguished_name
prompt             = no                      # CSR 생성 시 대화형 입력 없음
x509_extensions    = ca_x509_extensions      # CA 인증서 생성 시 사용할 확장

[ca_x509_extensions]                         # CA 인증서 설정 (Root of Trust)
basicConstraints = CA:TRUE                   # CA 권한 인증서
keyUsage         = cRLSign, keyCertSign      # 다른 인증서를 서명 가능, Kubernetes 모든 인증의 신뢰 루트

[req_distinguished_name]
C   = US
ST  = Washington
L   = Seattle
CN  = CA                                     # 클러스터 CA

[admin]                                      # Admin 사용자 (kubectl)
distinguished_name = admin_distinguished_name
prompt             = no
req_extensions     = default_req_extensions

[admin_distinguished_name]
CN = admin                                   # [K8S] CN → user
O  = system:masters                          # [K8S] O → group , system:masters - Kubernetes 슈퍼유저 그룹, 모든 RBAC 인가 우회

# Service Accounts
#
# The Kubernetes Controller Manager leverages a key pair to generate
# and sign service account tokens as described in the
# [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/)
# documentation.

[service-accounts]                           # Service Account 서명자
distinguished_name = service-accounts_distinguished_name
prompt             = no
req_extensions     = default_req_extensions

[service-accounts_distinguished_name] 
CN = service-accounts                        # controller-manager가 사용하는 ServiceAccount 토큰 서명용 인증서 , apiserver에서 --service-account-key-file 로 사용

# Worker Nodes
#
# Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/)
# called Node Authorizer, that specifically authorizes API requests made
# by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet).
# In order to be authorized by the Node Authorizer, Kubelets must use a credential
# that identifies them as being in the `system:nodes` group, with a username
# of `system:node:<nodeName>`.

[node-0]                                    # Worker Node 인증서 (kubelet)
distinguished_name = node-0_distinguished_name
prompt             = no
req_extensions     = node-0_req_extensions

[node-0_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth  # clientAuth: apiserver → kubelet & serverAuth: kubelet HTTPS 서버(10250)
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Node-0 Certificate"
subjectAltName       = DNS:node-0, IP:127.0.0.1
subjectKeyIdentifier = hash

[node-0_distinguished_name]
CN = system:node:node-0                     # kubelet 사용자 , CN = system:node:<nodeName>
O  = system:nodes                           # Node Authorizer 그룹  ,O = system:nodes
C  = US
ST = Washington
L  = Seattle

[node-1]
distinguished_name = node-1_distinguished_name
prompt             = no
req_extensions     = node-1_req_extensions

[node-1_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Node-1 Certificate"
subjectAltName       = DNS:node-1, IP:127.0.0.1
subjectKeyIdentifier = hash

[node-1_distinguished_name]
CN = system:node:node-1
O  = system:nodes
C  = US
ST = Washington
L  = Seattle


# Kube Proxy Section
[kube-proxy]                                # kube-proxy
distinguished_name = kube-proxy_distinguished_name
prompt             = no
req_extensions     = kube-proxy_req_extensions

[kube-proxy_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Kube Proxy Certificate"
subjectAltName       = DNS:kube-proxy, IP:127.0.0.1
subjectKeyIdentifier = hash

[kube-proxy_distinguished_name]
CN = system:kube-proxy
O  = system:node-proxier                    # system:node-proxier ClusterRoleBinding 존재 , 서비스 네트워크 제어 가능
C  = US
ST = Washington
L  = Seattle


# Controller Manager
[kube-controller-manager]
distinguished_name = kube-controller-manager_distinguished_name
prompt             = no
req_extensions     = kube-controller-manager_req_extensions

[kube-controller-manager_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Kube Controller Manager Certificate"
subjectAltName       = DNS:kube-controller-manager, IP:127.0.0.1
subjectKeyIdentifier = hash

[kube-controller-manager_distinguished_name] # 클러스터 상태 관리, Node, ReplicaSet, SA 토큰 등 관리
CN = system:kube-controller-manager
O  = system:kube-controller-manager
C  = US
ST = Washington
L  = Seattle


# Scheduler
[kube-scheduler]
distinguished_name = kube-scheduler_distinguished_name
prompt             = no
req_extensions     = kube-scheduler_req_extensions

[kube-scheduler_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Kube Scheduler Certificate"
subjectAltName       = DNS:kube-scheduler, IP:127.0.0.1
subjectKeyIdentifier = hash

[kube-scheduler_distinguished_name]
CN = system:kube-scheduler
O  = system:kube-scheduler                  # Pod 스케줄링 전용 권한
C  = US
ST = Washington
L  = Seattle


# API Server
#
# The Kubernetes API server is automatically assigned the `kubernetes`
# internal dns name, which will be linked to the first IP address (`10.32.0.1`)
# from the address range (`10.32.0.0/24`) reserved for internal cluster
# services.

[kube-api-server]                           # API Server 인증서
distinguished_name = kube-api-server_distinguished_name
prompt             = no
req_extensions     = kube-api-server_req_extensions

[kube-api-server_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client, server
nsComment            = "Kube API Server Certificate"
subjectAltName       = @kube-api-server_alt_names
subjectKeyIdentifier = hash

[kube-api-server_alt_names]                 # SAN (Subject Alternative Name) : 모든 내부/외부 접근 주소
IP.0  = 127.0.0.1
IP.1  = 10.32.0.1
DNS.0 = kubernetes
DNS.1 = kubernetes.default
DNS.2 = kubernetes.default.svc
DNS.3 = kubernetes.default.svc.cluster
DNS.4 = kubernetes.svc.cluster.local
DNS.5 = server.kubernetes.local
DNS.6 = api-server.kubernetes.local

[kube-api-server_distinguished_name]
CN = kubernetes
C  = US
ST = Washington
L  = Seattle


[default_req_extensions]                    # 공통 CSR 확장 : 대부분 클라이언트 인증서 -> kubelet / apiserver만 serverAuth 추가
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Admin Client Certificate"
subjectKeyIdentifier = hash
```