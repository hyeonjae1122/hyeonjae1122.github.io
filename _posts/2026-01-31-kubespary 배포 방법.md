---
layout: post
title: "[K8S Deploy Study by Gasida] -  kubespary 배포 방법"
categories:
  - kubernetes
tags:
  - k8s
  - kubespray
  - k9s
---
# TL;DR

> kubespary로 k8s 환경 구축시 목표 환경을 위한 파라미터 설정 방법을 알아본다.

<br>
# kubespary 설치 사전 요건

https://github.com/kubernetes-sigs/kubespray/tree/master?tab=readme-ov-file#requirements
- k8s version  > v1.30
- Ansible v2.14+, Jinja 2.11+
- Control Plane
    - Memory : 2GB
- Work Node
    - Memory : 1GB
- Linux Kernel Requirements: 5.8+
- Rocky Linux 9, 10 (experimental in 10)

# kubespray 다운 및 종속성 설치

```bash
git clone -b v2.29.1 https://github.com/kubernetes-sigs/kubespray.git /root/kubespray
```

```
pip3 install -r /root/kubespray/requirements.txt
```


# kubespray를 통한 k8s 배포

- Inventory 디렉터리 복사

```bash
cp -rfp /root/kubespray/inventory/sample /root/kubespray/inventory/mycluster
tree inventory/mycluster/
```


- inventory.ini 작성

```bash
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
k8s-ctr ansible_host=192.168.10.10 ip=192.168.10.10

[kube_control_plane]
k8s-ctr

[etcd:children]
kube_control_plane

[kube_node]
k8s-ctr
EOF
```



## 전체

```bash
grep "^[^#]" inventory/mycluster/group_vars/all/all.yml
```


- 테스트할 기능 관련 수정

```bash
sed -i 's|kube_network_plugin: calico|kube_network_plugin: flannel|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|kube_proxy_mode: ipvs|kube_proxy_mode: iptables|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|enable_nodelocaldns: true|enable_nodelocaldns: false|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|auto_renew_certificates: false|auto_renew_certificates: true|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|# auto_renew_certificates_systemd_calendar|auto_renew_certificates_systemd_calendar|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

- 확인

```bash
grep -iE 'kube_network_plugin:|kube_proxy_mode|enable_nodelocaldns:|^auto_renew_certificates' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

## CNI

- flannel 관련 설정 경로
    - `inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml`

```bash
cat inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml
```

- flannel 설정 수정

```bash
echo "flannel_interface: enp0s9" >> inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml
grep "^[^#]" inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml
```


## 애드온 

- 선호하는 애드온 수정할 경우 항목 검색하기
    - `inventory/mycluster/group_vars/k8s_cluster/addons.yml`
 
```bash
grep "^[^#]" inventory/mycluster/group_vars/k8s_cluster/addons.yml
```


- 테스트할 기능 관련 수정

```bash
sed -i 's|helm_enabled: false|helm_enabled: true|g' inventory/mycluster/group_vars/k8s_cluster/addons.yml
sed -i 's|metrics_server_enabled: false|metrics_server_enabled: true|g' inventory/mycluster/group_vars/k8s_cluster/addons.yml
sed -i 's|node_feature_discovery_enabled: false|node_feature_discovery_enabled: true|g' inventory/mycluster/group_vars/k8s_cluster/addons.yml
grep -iE 'helm_enabled:|metrics_server_enabled:|node_feature_discovery_enabled:' inventory/mycluster/group_vars/k8s_cluster/addons.yml
```


## ETCD

-  kubespray 에서는 파드가 아닌 `systemd unit`으로 설치한다.
- `inventory/mycluster/group_vars/all/etcd.yml`

```bash
grep "^[^#]" inventory/mycluster/group_vars/all/etcd.yml
```


## containerd

- `inventory/mycluster/group_vars/all/containerd.yml`

```bash
cat inventory/mycluster/group_vars/all/containerd.yml
```



## 지원되는 버전 정보 확인하기 

- `roles/kubespray_defaults/vars/main/checksums.yml`

```bash 
cat roles/kubespray_defaults/vars/main/checksums.yml | grep -i kube -A40
```



# 배포
- `~/kubespray` 디렉토리에서 `ansible-playbook` 을 실행

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml -e kube_version="1.33.3" --list-tasks # 배포 전, Task 목록 확인

ANSIBLE_FORCE_COLOR=true ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml -e kube_version="1.33.3" | tee kubespray_install.log
```


# 설치확인

 - 확인 

```bash
# 설치 확인 : /root/.kube/config
more kubespray_install.log
kubectl get node -v=6
cat /root/.kube/config
```

```bash
kubectl get node -owide
kubectl get pod -A
```

- (Optional) 출력 비교하기 위한 정보저장

```bash
# 기본 환경 정보 출력 저장
ip addr | tee -a ip_addr-2.txt 
ss -tnlp | tee -a ss-2.txt
df -hT | tee -a df-2.txt
findmnt | tee -a findmnt-2.txt
sysctl -a | tee -a sysctl-2.txt

# 파일 출력 비교 : 빠져나오기 ':q' -> ':q' => 변경된 부분이 어떤 동작과 역할인지 조사해보기! , ctrl + f / b
vi -d ip_addr-1.txt ip_addr-2.txt
vi -d ss-1.txt ss-2.txt
vi -d df-1.txt df-2.txt
vi -d findmnt-1.txt findmnt-2.txt
vi -d sysctl-1.txt sysctl-2.txt
```


 - (Optional) alias, 자동완성 및 k9s 설치

```bash
# Source the completion
source <(kubectl completion bash)
source <(kubeadm completion bash)

# Alias kubectl to k
alias k=kubectl
complete -o default -F __start_kubectl k
```


- k9s 설치  https://github.com/derailed/k9s
```bash
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
wget https://github.com/derailed/k9s/releases/latest/download/k9s_linux_${CLI_ARCH}.tar.gz
tar -xzf k9s_linux_*.tar.gz
ls -al k9s
chown root:root k9s
mv k9s /usr/local/bin/
chmod +x /usr/local/bin/k9s
```