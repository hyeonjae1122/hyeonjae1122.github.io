
집에 남아도는 `x86_64` 아키텍처의 리눅스 서버로 Rocky Linux 환경으로 실습을 진행 하였습니다. 

`최종 구성도`

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T111321715Z.png)



# Prerequisites

- 가상머신 프로비저닝을 위한 VirtualBox 설치

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

- 가상머신 프로비저닝을 위한 Vagrant 설치

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install vagrant
```

> 설치시 트러블 슈팅

```bash
# IP추가
sudo bash -c 'echo "* 192.168.10.0/24" >> /etc/vbox/networks.conf'
```

-  VirtualBox와 KVM 충돌시

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


# Vagrant로 가상 머신 프로비저닝 하기

#### Rocky Linux용 자동화 스크립트 다운로드(init_cfg.sh)

- TASK 1 : vagrant 사용자가 로그인할 때 자동으로 root 권한으로 전환되게 설정하고, vi 명령어를 vim으로 자동 변환하며, 서버의 시간대를 서울(Asia/Seoul)로 설정
- TASK 2: Rocky Linux는 보안 강화 모듈인 SELinux를 사용
- TASK 3: 메모리 가상 확장 기능인 SWAP을 끈다.
- TASK 4: 필요한 유틸리티들을 설치
- TASK 5: 관리자 계정의 비밀번호를 'qwe123'으로 설정
- TASK 6: 원격 접속을 위해 암호 인증과 root 로그인을 허용하도록 설정한 후 SSH 서비스를 재시작한다.
- TASK 7:  hosts 파일에 IP와 호스트명을 매핑하여, DNS 서버 없이도 각 서버에 접근할 수 있게 설정

[init_cfg.sh 다운로드 Link](https://raw.githubusercontent.com/hyeonjae1122/Vagrant-for-Rocky-Linux/refs/heads/main/init_cfg.sh)

```bash
wget https://raw.githubusercontent.com/hyeonjae1122/Vagrant-for-Rocky-Linux/refs/heads/main/init_cfg.sh
```


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
192.168.10.10  jumpbox
192.168.10.100 server.kubernetes.local server
192.168.10.101 node-0.kubernetes.local node-0
192.168.10.102 node-1.kubernetes.local node-1
EOF

echo ">>>> Initial Config End <<<<"
```


#### Rockylinux 이미지와 버전정보

[Vagrantfile 다운로드 Link](https://raw.githubusercontent.com/hyeonjae1122/Vagrant-for-Rocky-Linux/refs/heads/main/vagrantfile) 

```bash
wget https://raw.githubusercontent.com/hyeonjae1122/Vagrant-for-Rocky-Linux/refs/heads/main/vagrantfile
```

[Box Link](https://portal.cloud.hashicorp.com/vagrant/discover/bento/rockylinux-9) bento/rockylinux-9 이미지 경로

```bash
BOX_IMAGE = "bento/rockylinux-9"
BOX_VERSION = "202510.26.0"
```


#### Vagrant 를 이용한 가상 머신 배포

총 4대의 가상머신을 배포한다. 

| NAME    | Description         | CPU | RAM     | NIC1      | NIC2               | HOSTNAME                           |
| ------- | ------------------- | --- | ------- | --------- | ------------------ | ---------------------------------- |
| jumpbox | Administration host | 2   | 1536 MB | 10.0.2.15 | **192.168.10.10**  | **jumpbox**                        |
| server  | Kubernetes server   | 2   | 2GB     | 10.0.2.15 | **192.168.10.100** | server.kubernetes.local **server** |
| node-0  | Kubernetes worker   | 2   | 2GB     | 10.0.2.15 | **192.168.10.101** | node-0.kubernetes.local **node-0** |
| node-1  | Kubernetes worker   | 2   | 2GB     | 10.0.2.15 | **192.168.10.102** | node-1.kubernetes.local **node-1** |

배포 명령어 실행

```bash
ls -al
-rw-rw-r-- 1 hyeonjae hyeonjae 3073 Jan 10 18:01 vagrantfile

vagrant up
```

```bash
# 실습용 OS 이미지 자동 다운로드 확인
vagrant box list
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T112658778Z.png)


```
# 배포된 가상머신 확인
vagrant status
```

아래와 같이 예상대로 4대의 가상머신이 배포된 것을 확인할 수 있다.

![|600x150](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T043413999Z.png)


#### jumpbox 가상머신접속

jumpbox에 ssh접속을 위한 명령어이다. 아주 많이 쓰이는 명렁어이므로 익숙해지자.

```ssh
vagrant ssh jumpbox
```


머신 상태확인 명렁어 모음

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

`cat /etc/os-release` 를 통하여 `Rock Linux` 임을 확인

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T043521888Z.png)


필요한 각종 Tool을 설치한다. 기본 저장소에 없는 패키지들도 있으니 `epel-release`를 추가하자.

```bash
# Tool Install
## EPEL = Extra Packages for Enterprise Linux**
### Rocky Linux(RHEL 기반)의 **기본 저장소에 없는 추가 패키지들을 제공하는 저장소

dnf -y install epel-release 
dnf install tree git jq yq unzip vim sshpass -y

# Installed:
#  git-2.47.3-1.el9_6.x86_64                     git-core-2.47.3-1.el9_6.x86_64
#  git-core-doc-2.47.3-1.el9_6.noarch            perl-DynaLoader-1.47-481.1.el9_6.x86_64
#  perl-Error-1:0.17029-7.el9.0.1.noarch         perl-File-Find-1.37-481.1.el9_6.noarch
#  perl-Git-2.47.3-1.el9_6.noarch                perl-TermReadKey-2.38-11.el9.x86_64
#  perl-lib-0.65-481.1.el9_6.x86_64              sshpass-1.09-4.el9.x86_64
#  yq-4.47.1-2.el9.x86_64
# Complete!
```


k8s hardway에 필요한 리소스를 다운 받기 위해 아래 레포지토리를 클론하자. 
```bash
# Sync GitHub Repository ## --depth 1 : 최신 커밋만 가져오는 shallow clone을 의미
git clone --depth 1 https://github.com/kelseyhightower/kubernetes-the-hard-way.git

cd kubernetes-the-hard-way
```

아키텍처 확인

```bash
# CPU 아키텍처 확인 
uname -m # x86_64
uname -a # Linux jumpbox 5.14.0-570.52.1.el9_6.x86_64 #1 SMP PREEMPT_DYNAMIC Wed Oct 15 13:59:22 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
```

레포지토리 파일 내용 확인

```bash
cd kubernetes-the-hard-way

ls -al
total 60
drwxr-xr-x. 6 root root  4096 Jan 10 20:52 .
dr-xr-x---. 4 root root   150 Jan 10 20:52 ..
-rw-r--r--. 1 root root  5863 Jan 10 20:40 ca.conf
drwxr-xr-x. 2 root root  4096 Jan 10 20:40 configs
-rw-r--r--. 1 root root  1059 Jan 10 20:40 CONTRIBUTING.md
-rw-r--r--. 1 root root   407 Jan 10 20:40 COPYRIGHT.md
drwxr-xr-x. 2 root root  4096 Jan 10 20:40 docs
-rw-r--r--. 1 root root   839 Jan 10 20:40 downloads-amd64.txt
-rw-r--r--. 1 root root   839 Jan 10 20:40 downloads-arm64.txt
drwxr-xr-x. 8 root root   178 Jan 10 20:40 .git
-rw-r--r--. 1 root root  1167 Jan 10 20:40 .gitignore
-rw-r--r--. 1 root root 11358 Jan 10 20:40 LICENSE
-rw-r--r--. 1 root root  2624 Jan 10 20:40 README.md
drwxr-xr-x. 2 root root  4096 Jan 10 20:40 units
```

k8s 구성을 위한 컴포넌트 다운로드한다. 위에서 다운로드한 레포지토리에 있는 바이너리 파일들의 링크는 버전이 낮으므로 아래와 같이 1.34버전에 맞춰준다.

[쿠버네티스 다운로드 공식 링크] (https://www.downloadkubernetes.com/)

```sh
cat > downloads-amd64.txt << 'EOF' https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-scheduler https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubectl https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubelet https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-apiserver https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-controller-manager https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-proxy https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.34.0/crictl-v1.34.0-linux-amd64.tar.gz https://github.com/opencontainers/runc/releases/download/v1.4.0/runc.amd64 https://github.com/containernetworking/plugins/releases/download/v1.8.0/cni-plugins-linux-amd64-v1.8.0.tgz https://github.com/containerd/containerd/releases/download/v2.1.5/containerd-2.1.5-linux-amd64.tar.gz https://github.com/etcd-io/etcd/releases/download/v3.6.7/etcd-v3.6.7-linux-amd64.tar.gz 
EOF
```


최신버전으로 변경하였다면 wget으로 다운로드한다. 

```sh
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-amd64.txt
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T122029768Z.png)

각종 컴포넌트 분류를 위한 폴더를 만들고 압축을 풀어준다.

```bash
mkdir -p downloads/{client,cni-plugins,controller,worker}
tree -d downloads

# 압축풀기
tar -xvf downloads/crictl-v1.34.0-linux-amd64.tar.gz \
-C downloads/worker/ && tree -ug downloads

tar -xvf downloads/containerd-2.1.5-linux-amd64.tar.gz --strip-components 1 -C downloads/worker/ && tree -ug downloads

tar -xvf downloads/cni-plugins-linux-amd64-v1.8.0.tgz -C downloads/cni-plugins/ && tree -ug downloads

tar -xvf downloads/etcd-v3.6.7-linux-amd64.tar.gz \
-C downloads/ \
--strip-components 1 \
etcd-v3.6.7-linux-amd64/etcdctl \
etcd-v3.6.7-linux-amd64/etcd && tree -ug downloads

# 확인
tree downloads/worker/ 
tree downloads/cni-plugins
ls -l downloads/{etcd,etcdctl}

# 파일 이동 
mv downloads/{etcdctl,kubectl} downloads/client/ 
mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} downloads/controller/ 
mv downloads/{kubelet,kube-proxy} downloads/worker/ 
mv downloads/runc.amd64 downloads/worker/runc

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


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T122637566Z.png)

kubectl 버전 확인

```sh
kubectl version --client
```

![[2026-01-04-Bootstrap k8s the hard way-18.png]]



# SSH 접속 환경 설정

`vagrant ssh jumpbox`

- SSH 접속환경을 설정해준다. 
	- IP , FQDN, HOST, SUBNET을 설정
	- 새로운 SSH key를 생성하고 각각 노드에 전달
	- Hostname 설정 및 ssh 접속 확인


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

- 총 9개의 항목에 대해 개인키 및 인증서를 생성한다. 

| 항목                      | 개인키                         | CSR                         | 인증서                         | 참고 정보                                                                      | X509v3 Extended Key Usage                              |
| ----------------------- | --------------------------- | --------------------------- | --------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------ |
| Root CA                 | ca.key                      | X                           | ca.crt                      |                                                                            |                                                        |
| admin                   | admin.key                   | admin.csr                   | admin.crt                   | CN = admin, O = system:masters                                             | TLS **Web Client** Authentication                      |
| node-0                  | node-0.key                  | node-0.csr                  | node-0.crt                  | CN = system:node:node-0, O = system:nodes                                  | TLS **Web** ==**Server**== **/ Client** Authentication |
| node-1                  | node-1.key                  | node-1.csr                  | node-1.crt                  | CN = system:node:node-1, O = system:nodes                                  | TLS **Web** ==**Server**== **/ Client** Authentication |
| kube-proxy              | kube-proxy.key              | kube-proxy.csr              | kube-proxy.crt              | CN = system:kube-proxy, O = system:node-proxier                            | TLS **Web** ==Server== / **Client** Authentication     |
| kube-scheduler          | kube-scheduler.key          | kube-scheduler              | kube-scheduler.crt          | CN = system:kube-scheduler, O = system:kube-scheduler                      | TLS **Web** ==Server== / **Client** Authentication     |
| kube-controller-manager | kube-controller-manager.key | kube-controller-manager.csr | kube-controller-manager.crt | CN = system:kube-controller-manager, O = system:kube-controller-manager    | TLS **Web** ==Server== / **Client** Authentication     |
| kube-api-server         | kube-api-server.key         | kube-api-server.csr         | kube-api-server.crt         | CN = kubernetes, SAN: IP(127.0.0.1, ==**10.32.0.1**==), DNS(kubernetes,..) | TLS **Web** ==**Server**== **/ Client** Authentication |
| service-accounts        | service-accounts.key        | service-accounts.csr        | service-accounts.crt        | CN = service-accounts                                                      | TLS **Web Client** Authentication                      |
|                         |                             |                             |                             |                                                                            |                                                        |


| 항목                  | 네트워크 대역 or IP     |
| ------------------- | ----------------- |
| **clusterCIDR**     | 10.200.0.0/16     |
| → node-0 PodCIDR    | 10.200.0.0/24     |
| → node-1 PodCIDR    | 10.200.1.0/24     |
| **ServiceCIDR**     | **10.32.0.0/24**  |
| → **api clusterIP** | ==**10.32.0.1**== |

`ca.conf`  내용 살펴보기

| 구분                          | 역할                   |
| --------------------------- | -------------------- |
| `[req]`                     | OpenSSL 요청 기본 동작     |
| `[ca_*]`                    | CA 인증서               |
| `[admin]`                   | 관리자 (kubectl)        |
| `[service-accounts]`        | ServiceAccount 토큰 서명 |
| `[node-*]`                  | 워커 노드(kubelet)       |
| `[kube-proxy]`              | kube-proxy           |
| `[kube-controller-manager]` | 컨트롤러                 |
| `[kube-scheduler]`          | 스케줄러                 |
| `[kube-api-server]`         | API Server           |
| `[default_req_extensions]`  | 공통 CSR 옵션            |

```bash
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

## 각 컴포넌트별 개인키 , CSR, 인증서 생성

```bash
# Root CA 개인키
openssl genrsa -out ca.key 4096
openssl req -x509 -new -sha512 -noenc \
  -key ca.key -days 3653 \
  -config ca.conf \
  -out ca.crt
  
# Create Client and Server Certificates : admin
openssl genrsa -out admin.key 4096
openssl req -new -key admin.key -sha256 \
  -config ca.conf -section admin \
  -out admin.csr

openssl x509 -req -days 3653 -in admin.csr \
  -copy_extensions copyall \
  -sha256 -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out admin.crt  
  
# Create Client and Server Certificates: 나머지
certs=( "node-0" "node-1" "kube-proxy" "kube-scheduler" "kube-controller-manager" "kube-api-server" "service-accounts" )

# 확인 
echo ${certs[*]}

# 개인키 생성, csr 생성, 인증서 생성 
for i in ${certs[*]}; do
openssl genrsa -out "${i}.key" 4096

openssl req -new -key "${i}.key" -sha256 \
-config "ca.conf" -section ${i} \
-out "${i}.csr"

openssl x509 -req -days 3653 -in "${i}.csr" \
-copy_extensions copyall \
-sha256 -CA "ca.crt" \
-CAkey "ca.key" \
-CAcreateserial \
-out "${i}.crt"
done

```

생성된 개인키 , csr, 인증서 확인

```bash
ls -1 *.crt *.key *.csr

admin.crt
admin.csr
admin.key
ca.crt
ca.key
kube-api-server.crt
kube-api-server.csr
kube-api-server.key
kube-controller-manager.crt
kube-controller-manager.csr
kube-controller-manager.key
kube-proxy.crt
kube-proxy.csr
kube-proxy.key
kube-scheduler.crt
kube-scheduler.csr
kube-scheduler.key
node-0.crt
node-0.csr
node-0.key
node-1.crt
node-1.csr
node-1.key
service-accounts.crt
service-accounts.csr
service-accounts.key
```

인증서 / 개인키 , csr 정보 를 확인

```bash
# 각종 key 확인
openssl rsa -in ca.key -text -noout
openssl rsa -in admin.key -text -noout
openssl rsa -in node-0.key -text -noout
openssl rsa -in node-1.key -text -noout
openssl rsa -in kube-proxy.key -text -noout
openssl rsa -in kube-scheduler.key -text -noout
openssl rsa -in kube-controller-manager.key -text -noout
openssl rsa -in kube-api-server.key -text -noout
openssl rsa -in service-accounts.key -text -noout

#각종 csr 확인
# Root CA 인증서 생성하였으므로(Self-Signed certificate) 없음
openssl req -in admin.csr -text -noout
openssl req -in node-0.csr -text -noout
openssl req -in node-1.csr -text -noout
openssl req -in kube-proxy.csr -text -noout
openssl req -in kube-scheduler.csr -text -noout
openssl req -in kube-controller-manager.csr -text -noout
openssl req -in kube-api-server.csr -text -noout
openssl req -in service-accounts.csr -text -noout

# 각종 인증서 확인
openssl x509 -in ca.crt -text -noout
openssl x509 -in admin.crt -text -noout
openssl x509 -in node-0.crt -text -noout
openssl x509 -in node-1.crt -text -noout
openssl x509 -in kube-proxy.crt -text -noout
openssl x509 -in kube-scheduler.crt -text -noout
openssl x509 -in kube-controller-manager.crt -text -noout
openssl x509 -in kube-api-server.crt -text -noout
openssl x509 -in service-accounts.crt -text -noout
```

## node-0 , node-1에 클라이언트 및 서버 증명서 전달

```bash
for host in node-0 node-1; do
ssh root@${host} mkdir /var/lib/kubelet/

scp ca.crt root@${host}:/var/lib/kubelet/
scp ${host}.crt \
 root@${host}:/var/lib/kubelet/kubelet.crt

scp ${host}.key \
  root@${host}:/var/lib/kubelet/kubelet.key
done

# 확인
ssh node-0 ls -l /var/lib/kubelet 
ssh node-1 ls -l /var/lib/kubelet

# 서버머신에 적절한 개인키 및 증명서를 전달
scp \ 
  ca.key ca.crt \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  root@server:~  
  
# 확인
ssh server ls -l /root
  
```

## API Server와 통신을 위한 Client 인증 설정 파일 작성

- `kubelet`  k8s config 파일

```bash
# apiserver 파드 args 정보
kubectl describe pod -n kube-system kube-apiserver-myk8s-control-plane
    Command:
      kube-apiserver
      --authorization-mode=Node,RBAC  


# Generate a kubeconfig file for the node-0 and node-1 worker nodes

# config set-cluster
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=node-0.kubeconfig && ls -l node-0.kubeconfig && cat node-0.kubeconfig

kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=node-1.kubeconfig && ls -l node-1.kubeconfig && cat node-1.kubeconfig

# config set-credentials
kubectl config set-credentials system:node:node-0 \
  --client-certificate=node-0.crt \
  --client-key=node-0.key \
  --embed-certs=true \
  --kubeconfig=node-0.kubeconfig && cat node-0.kubeconfig

kubectl config set-credentials system:node:node-1 \
  --client-certificate=node-1.crt \
  --client-key=node-1.key \
  --embed-certs=true \
  --kubeconfig=node-1.kubeconfig && cat node-1.kubeconfig
  
# set-context : default 추가
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:node-0 \
  --kubeconfig=node-0.kubeconfig && cat node-0.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:node-1 \
  --kubeconfig=node-1.kubeconfig && cat node-1.kubeconfig


# use-context : current-context 에 default 추가
kubectl config use-context default \
  --kubeconfig=node-0.kubeconfig

kubectl config use-context default \
  --kubeconfig=node-1.kubeconfig


#
ls -l *.kubeconfig
-rw------- 1 root root 10157 Jan  3 14:55 node-0.kubeconfig
-rw------- 1 root root 10068 Jan  3 14:50 node-1.kubeconfig
```

- `kube-proxy` k8s config 파일

```bash
# Generate a kubeconfig file for the kube-proxy service
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.crt \
  --client-key=kube-proxy.key \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default \
  --kubeconfig=kube-proxy.kubeconfig

# 확인
cat kube-proxy.kubeconfig
```

- `kube-controller-manager` k8s config 파일

```bash
# Generate a kubeconfig file for the kube-controller-manager service
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.crt \
  --client-key=kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default \
  --kubeconfig=kube-controller-manager.kubeconfig

# 확인
cat kube-controller-manager.kubeconfig
```

- `kube-scheduler` k8s config 파일

```bash
# Generate a kubeconfig file for the kube-scheduler service
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.crt \
  --client-key=kube-scheduler.key \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default \
  --kubeconfig=kube-scheduler.kubeconfig

# 확인
cat kube-scheduler.kubeconfig
```

- `admin` k8s config 파일

```bash
# Generate a kubeconfig file for the admin user
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default \
  --kubeconfig=admin.kubeconfig

# 확인
cat admin.kubeconfig
```

## API Server와 통신을 위한 Client 인증 설정 파일을 서버 및 노드에 전달

- `kubelet` and `kube-proxy` 의 kubeconfig를 `node-0` 및 `node-1`에 복사

```bash
#
ls -l *.kubeconfig


# kubelet and kube-proxy kubeconfig 파일을 node-0 및 node-1에 복사
for host in node-0 node-1; do
  ssh root@${host} "mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig \
    root@${host}:/var/lib/kube-proxy/kubeconfig \

  scp ${host}.kubeconfig \
    root@${host}:/var/lib/kubelet/kubeconfig
done

# 확인
ssh node-0 ls -l /var/lib/*/kubeconfig
ssh node-1 ls -l /var/lib/*/kubeconfig

```


- `kube-controller-manager` 및 `kube-scheduler`의  kubeconfig 파일을 server에 복사

```bash
# kube-controller-manager and kube-scheduler kubeconfig 파일을 server에 복사
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  root@server:~/

# 확인
ssh server ls -l /root/*.kubeconfig
```

# Server 노드에 etcd 서비스 기동


```bash
scp \
  kubernetes-the-hard-way/downloads/controller/etcd \
  kubernetes-the-hard-way/downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```


# Server 노드에 api-server,scheduler,kcm 서비스 기동


# node-0
```bash
ssh root@node-0
# EPEL 저장소 활성화 bridge-utils 설치를 위해
sudo dnf -y install epel-release 

# 
dnf -y install socat conntrack ipset kmod psmisc bridge-utils

# Disable Swap : Verify if swap is disabled:
swapon --show

#
mkdir -p \
 /etc/cni/net.d \ 
 /opt/cni/bin \
 /var/lib/kubelet \ 
 /var/lib/kube-proxy \
 /var/lib/kubernetes \
 /var/run/kubernetes
```


갑자기 되었는데?
```bash
# node-1에서
sudo systemctl status firewalld
getenforce

# 만약 활성화되어 있으면
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo setenforce 0


# 재시작

sudo systemctl restart containerd
sudo systemctl restart kubelet
sudo systemctl restart kube-proxy

sleep 5

# 상태 확인
sudo systemctl status kubelet --no-pager | tail -10
```