

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
