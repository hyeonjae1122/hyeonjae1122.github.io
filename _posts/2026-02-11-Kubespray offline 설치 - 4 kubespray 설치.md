---
layout: post
tags:
  - Ansible
  - k8s
  - kubernetes
  - kubespray
title: "[K8S Deploy Study by Gasida] - Kubespray offline 설치 - 4 kubespray 설치"
---
이전 단계에서 폐쇄망 환경을 위한 모든 에셋(OS 패키지, 컨테이너 이미지, PyPI)을 `outputs/` 디렉터리에 다운로드하고 로컬 Nginx와 Registry를 통해 서빙할 준비를 마쳤다.

이제 본격적으로 격리된 파이썬 가상환경(venv)에 진입하여 Ansible 플레이북을 조작하고, 실제 타겟 노드(`k8s-node1`, `k8s-node2`)에 Kubernetes 클러스터를 프로비저닝 한다.

#  1. 파이썬 가상환경(venv) 진입 및 Ansible 준비

Kubespray는 Ansible 기반이므로 시스템 파이썬 환경과 격리된 venv를 사용하는 것이 안전하다.

```bash
# 파이썬 버전 확인
python --version
# Python 3.12.12

# venv 실행 및 활성화
python3.12 -m venv ~/.venv/3.12
source ~/.venv/3.12/bin/activate

# ansible 설치 확인
which ansible
tree ~/.venv/3.12/ -L 4
```

```bash
# kubespray 디렉터리 이동
cd /root/kubespray-offline/outputs/kubespray-2.30.0

# 의존성 패키지 설치 (이전 단계에서 pypi 미러를 구성했으므로 로컬에서 빠르게 설치됨)
pip install -U pip
pip install -r requirements.txt
```

# 2. 오프라인 인벤토리 및 변수(`offline.yml`) 설정

Kubespray가 외부 인터넷망이 아닌 우리가 구축한 `admin` 노드(`192.168.10.10`)의 Nginx와 Registry를 바라보도록 설정한다. 

## 2.1 로컬 레포지토리 주소 매핑

```bash
# 기본 제공되는 offline.yml 복사 및 mycluster 인벤토리 생성
cp ../../offline.yml .
cp -r inventory/sample inventory/mycluster

# 웹서버와 이미지 저장소 IP 일괄 수정 (YOUR_HOST -> 192.168.10.10)
sed -i "s/YOUR_HOST/192.168.10.10/g" offline.yml

# 수정된 결과 확인
cat offline.yml | grep 192.168.10.10
# http_server: "http://192.168.10.10"
# registry_host: "192.168.10.10:35000"

# 완성된 offline.yml을 mycluster group_vars로 덮어쓰기
\cp -f offline.yml inventory/mycluster/group_vars/all/offline.yml
```

## 2.2 타겟 노드 인벤토리(`inventory.ini`) 작성

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
# Ansible 연결 테스트
ansible -i inventory/mycluster/inventory.ini all -m ping
```

# 3. 타겟 노드의 기본 Repo 차단 및 오프라인 Repo 설정

왜 기존 Repo를 지워야 하나?
폐쇄망 노드들은 인터넷이 차단되어 있다. 만약 기존의 `AppStream`, `BaseOS` 등 외부 통신을 시도하는 `.repo` 파일들을 살려두면, Ansible이 `dnf install` 작업을 수행할 때마다 인터넷 연결 타임아웃(Timeout)이 발생할 때까지 수 분간 멈춰있게 되며, 결국 배포가 실패(Fail)하게 된다. 망분리 환경에서는 외부 Repo를 반드시 `.original` 등으로 백업/비활성화해야 한다.


```bash
# offline-repo 플레이북 복사 및 실행
mkdir offline-repo
cp -r ../playbook/ offline-repo/
ansible-playbook -i inventory/mycluster/inventory.ini offline-repo/playbook/offline-repo.yml

# 노드 접속하여 외부 repo 비활성화 (매우 중요 ⭐)
for i in rocky-addons rocky-devel rocky-extras rocky; do
  ssh k8s-node1 "mv /etc/yum.repos.d/$i.repo /etc/yum.repos.d/$i.repo.original"
  ssh k8s-node2 "mv /etc/yum.repos.d/$i.repo /etc/yum.repos.d/$i.repo.original"
done

# 적용 확인
ssh k8s-node1 dnf repolist

```


# 4. K8s 클러스터 커스텀 설정 (`group_vars` 튜닝)

실습 환경에 맞게 CNI, DNS, 메트릭 서버 등의 설정을 오버라이드한다.

```bash
# 1. admin 노드에도 kubectl 바이너리 자동 복사 설정
echo "kubectl_localhost: true" >> inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|kube_owner: kube|kube_owner: root|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml

# 2. 네트워크 및 프록시 설정 (Calico/IPVS -> Flannel/IPTables)
sed -i 's|kube_network_plugin: calico|kube_network_plugin: flannel|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
sed -i 's|kube_proxy_mode: ipvs|kube_proxy_mode: iptables|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
echo "flannel_interface: enp0s9" >> inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml

# 3. DNS 스케일러 및 NodeLocalDNS 비활성화 (경량화)
sed -i 's|enable_nodelocaldns: true|enable_nodelocaldns: false|g' inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
echo "enable_dns_autoscaler: false" >> inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml

# 4. Helm 및 Metrics Server 활성화
sed -i 's|helm_enabled: false|helm_enabled: true|g' inventory/mycluster/group_vars/k8s_cluster/addons.yml
sed -i 's|metrics_server_enabled: false|metrics_server_enabled: true|g' inventory/mycluster/group_vars/k8s_cluster/addons.yml
echo "metrics_server_requests_cpu: 25m"     >> inventory/mycluster/group_vars/k8s_cluster/addons.yml
echo "metrics_server_requests_memory: 16Mi" >> inventory/mycluster/group_vars/k8s_cluster/addons.yml
```

#  5. 트러블슈팅: Apple Silicon (macOS) 사용자의 ARM64 이슈

PC가 M1/M2 등 Apple Silicon 기반(ARM64) 환경일 경우, 사전 다운로드 단계에서 `amd64`용이 아닌 `arm64`용 바이너리가 다운로드된다. 하지만 `offline.yml` 파일에는 기본적으로 `amd64`로 하드코딩 되어 있어 로컬 Nginx에서 파일을 찾지 못하는 것

Ansible 배포 중 아래와 같이 `etcd` 바이너리를 다운로드하지 못해(HTTP 404) FAILED가 발생하는 경우가 있다.

```bash
TASK [download : Download_file | Download item] **************************************
fatal: [k8s-node1]: FAILED! => {"msg": "Request failed", "response": "HTTP Error 404: Not Found", "url": "http://192.168.10.10/files/kubernetes/etcd/etcd-v3.5.26-linux-amd64.tar.gz"}
```


- 해결방안

```bash
# offline.yml 내의 amd64 문자열을 arm64로 치환
sed -i 's/amd64/arm64/g' inventory/mycluster/group_vars/all/offline.yml
```

# 6. 클러스터 배포 및 결과 검증

플레이북을 실행하여 배포를 시작

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml -e kube_version="1.34.3"
```

## 6.1 `kubectl` 세팅 및 자동완성 적용

사전에 `kubectl_localhost: true`를 주었기 때문에, 앤서블을 실행한 admin 노드에 자격증명(`kubeconfig`)과 `kubectl` 바이너리가 수집되어 있다.

```bash
# 바이너리 복사 및 버전 확인
cp inventory/mycluster/artifacts/kubectl /usr/local/bin/
kubectl version --client=true

# admin 권한 획득 (kubeconfig)
mkdir /root/.kube
scp k8s-node1:/root/.kube/config /root/.kube/
sed -i 's/127.0.0.1/192.168.10.11/g' /root/.kube/config

# kubectl 자동완성(Auto-completion) 및 Alias 설정
source <(kubectl completion bash)
alias k=kubectl
complete -F __start_kubectl k

# 영구 적용
echo 'source <(kubectl completion bash)' >> /etc/profile
echo 'alias k=kubectl' >> /etc/profile
echo 'complete -F __start_kubectl k' >> /etc/profile
```

## 6.2 오프라인 Registry 활용 검증

클러스터에 배포된 Pod들이 실제로 인터넷 망이 아닌, 우리가 구성한 로컬 사설 레지스트리(`192.168.10.10:35000`)에서 이미지를 정상적으로 Pull 받았는지 검증

```bash
kubectl get deploy,sts,ds -n kube-system -owide
```

- 컴포넌트의 IMAGES 출처가 `192.168.10.10:35000`으로 찍혀 있다면 완벽하게 폐쇄망 구축에 성공한 것이다.

```bash
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS       IMAGES                                                     
deployment.apps/coredns          2/2     2            2           4m9s   coredns          192.168.10.10:35000/coredns/coredns:v1.12.1                
deployment.apps/metrics-server   1/1     1            1           4m5s   metrics-server   192.168.10.10:35000/metrics-server/metrics-server:v0.8.0   

NAME                                     READY   AGE     CONTAINERS     IMAGES                                        
daemonset.apps/kube-flannel              0       4m25s   kube-flannel   192.168.10.10:35000/flannel/flannel:v0.27.3   
daemonset.apps/kube-proxy                1       4m43s   kube-proxy     192.168.10.10:35000/kube-proxy:v1.34.3
```



### NetworkManager와 CoreDNS 통신 원리 확인


```bash
ssh k8s-node2 cat /etc/NetworkManager/conf.d/dns.conf
# [global-dns-domain-*]
# servers = 10.233.0.3,192.168.10.10
```

Kubespray는 클러스터 내부의 DNS 해석을 위해 NetworkManager 설정에 CoreDNS IP(`10.233.0.3`)를 추가한다. 하지만 우리가 1일 차 네트워크 설정에서 `/etc/NetworkManager/conf.d/99-dns-none.conf`를 통해 NetworkManager가 DNS를 관리하지 못하도록 차단해 두었기 때문에, 실제 노드의 `resolv.conf`는 변경되지 않고 우리가 수동으로 지정한 `192.168.10.10`을 그대로 유지하고 있다. 이는 노드 자체에서 Kubernetes Service 명(예: `default.svc.cluster.local`)으로 도메인을 질의하는 것은 막고, K8s 내부의 Pod끼리만 CoreDNS를 통해 통신하도록 격리하는 깔끔한 아키텍처적 결과이다.