---
layout: post
tags:
  - Ansible
  - k8s
  - kubernetes
  - kubespray
title: "[K8S Deploy Study by Gasida] - Kubespray offline 설치 3 k8s 설치"
---

- 파이썬 버전 체크

```bash
python --version
```

-  venv 실행

```bash
python3.12 -m venv ~/.venv/3.12
source ~/.venv/3.12/bin/activate
which ansible
tree ~/.venv/3.12/ -L 4
```

- kubespary 디렉터리 이동

```bash
cd /root/kubespray-offline/outputs/kubespray-2.30.0

# Install ansible : 이미 설치 완료된 상태
pip install -U pip                # update pip
pip install -r requirements.txt   # Install ansible
```

- offline.yml 파일 복사 후 inventory 복사

```bash
cp ../../offline.yml .
cp -r inventory/sample inventory/mycluster
tree inventory/mycluster/
```

- 웹서버와 이미지 저장소 정보 수정 : http_server, registry_host

```bash
cat offline.yml
sed -i "s/YOUR_HOST/192.168.10.10/g" offline.yml
cat offline.yml | grep 192.168.10.10
```

-  수정 반영된 offline.yml 파일을 inventory 디렉터리 내부로 복사

```bash
cp -f offline.yml inventory/mycluster/group_vars/all/offline.yml
cat inventory/mycluster/group_vars/all/offline.yml
```


- inventory 파일 작성

```bash
cat <<EOF > inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1

[etcd:children]
kube_control_plane

[kube_node]
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12
EOF
```


```bash
cat inventory/mycluster/inventory.ini

# ansible 연결 확인
ansible -i inventory/mycluster/inventory.ini all -m ping


```

-  각 노드에 offline repo 설정

```bash
mkdir offline-repo
cp -r ../playbook/ offline-repo/
tree offline-repo/
ansible-playbook -i inventory/mycluster/inventory.ini offline-repo/playbook/offline-repo.yml
```

- k8s-node 확인

```bash
ssh k8s-node1 tree /etc/yum.repos.d/
ssh k8s-node1 dnf repolist


ssh k8s-node1 cat /etc/yum.repos.d/offline.repo
```



```bash
## 추가로 설치를 위해 기존 repo 제거 : 미실행할 경우, kubespary 실행 시 fail됨
for i in rocky-addons rocky-devel rocky-extras rocky; do
  ssh k8s-node1 "mv /etc/yum.repos.d/$i.repo /etc/yum.repos.d/$i.repo.original"
  ssh k8s-node2 "mv /etc/yum.repos.d/$i.repo /etc/yum.repos.d/$i.repo.original"
done

ssh k8s-node1 tree /etc/yum.repos.d/
ssh k8s-node1 dnf repolist

ssh k8s-node2 tree /etc/yum.repos.d/
ssh k8s-node2 dnf repolist


# admin-lb 에 kubectl 없는 것 확인
which kubectl

# group vars 실습 환경에 맞게 설정
echo "kubectl_localhost: true" >> inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml # 배포를 수행하는 로컬 머신의 bin 디렉토리에도 kubectl 바이너리를 다운로드
sed -i 's|kube_owner: kube|kube_owner: root|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|kube_network_plugin: calico|kube_network_plugin: flannel|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|kube_proxy_mode: ipvs|kube_proxy_mode: iptables|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|enable_nodelocaldns: true|enable_nodelocaldns: false|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
grep -iE 'kube_owner|kube_network_plugin:|kube_proxy_mode|enable_nodelocaldns:' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
echo "enable_dns_autoscaler: false" >> inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml

echo "flannel_interface: enp0s9" >> inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml
grep "^[^#]" inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml

sed -i 's|helm_enabled: false|helm_enabled: true|g' inventory/mycluster/group_vars/k8s_cluster/addons.yml
sed -i 's|metrics_server_enabled: false|metrics_server_enabled: true|g' inventory/mycluster/group_vars/k8s_cluster/addons.yml
grep -iE 'metrics_server_enabled:' inventory/mycluster/group_vars/k8s_cluster/addons.yml
echo "metrics_server_requests_cpu: 25m"     >> inventory/mycluster/group_vars/k8s_cluster/addons.yml
echo "metrics_server_requests_memory: 16Mi" >> inventory/mycluster/group_vars/k8s_cluster/addons.yml

# 지원 버전 정보 확인
cat roles/kubespray_defaults/vars/main/checksums.yml | grep -i kube -A40


```


```bash
cat inventory/mycluster/group_vars/all/offline.yml | grep amd64
etcd_download_url: "{{ files_repo }}/kubernetes/etcd/etcd-v{{ etcd_version }}-linux-amd64.tar.gz"
sed -i 's/amd64/arm64/g' inventory/mycluster/group_vars/all/offline.yml
# -----------------------------------------------
```


- **배포** 

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml -e kube_version="1.34.3"
```


- 설치 후 NetworkManger 에 dns 설정 파일 추가 확인

```bash
ssh k8s-node2 cat /etc/NetworkManager/conf.d/dns.conf
```


> # 하지만 '/etc/NetworkManager/conf.d/99-dns-none.conf' 파일로 인해, 위 설정이 resolv.conf 에 반영되지 않음, 즉 노드에서는 service명으로 도메인 질의는 불가능.


```bash
ssh k8s-node2 cat /etc/resolv.conf
nameserver 192.168.10.10
```

- 설치 후 NetworkManger 에서 특정 NIC은 관리하지 않게 설정 추가 확인

```bash
ssh k8s-node2 cat /etc/NetworkManager/conf.d/k8s.conf
[keyfile]
unmanaged-devices+=interface-name:kube-ipvs0;interface-name:nodelocaldns
```


- kubectl 바이너리 파일을 ansible-playbook 실행한 서버에 다운로드 확인

```bash
file inventory/mycluster/artifacts/kubectl
ls -l inventory/mycluster/artifacts/kubectl
tree inventory/mycluster/

cp inventory/mycluster/artifacts/kubectl /usr/local/bin/
kubectl version --client=true
```


-  k8s admin 자격증명 확인 

```bash
mkdir /root/.kube
scp k8s-node1:/root/.kube/config /root/.kube/
sed -i 's/127.0.0.1/192.168.10.11/g' /root/.kube/config
k9s
```


-  자동완성 및 단축키 설정

```bash

source <(kubectl completion bash)
alias k=kubectl
complete -F __start_kubectl k
echo 'source <(kubectl completion bash)' >> /etc/profile
echo 'alias k=kubectl' >> /etc/profile
echo 'complete -F __start_kubectl k' >> /etc/profile
```


- 이미지 저장소가 192.168.10.10:35000 임을 확인

```bash
kubectl get deploy,sts,ds -n kube-system -owide
```