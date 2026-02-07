---
layout: post
title: "[K8S Deploy Study by Gasida] -  Kubespary HA & Upgrade - 4 ETCD 노드 추가  및 제거 해보기"
tags:
  - ETCD
  - kubespray
  - kubernetes
  - Ansible
---


>목표
>ETCD 노드를 추가 및 제거 실습을 진행한다



# ETCD 노드(k8s-node5) 추가

- `inventory.ini`를 수정하자. 
    - `etcd`노드만을 추가하기 위해서 아래와 같이 변경한다 .
- `k8s-node4`(삭제완료), 5는 워커노드로도 사용되지 않았던 노드이다.
- 만약에 워커노드나 컨트롤 플레인으로 사용되었던 노드라면 먼저 해당 노드를 제거해야한다.(`remove-node.yml`)

```bash
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13 etcd_member_name=etcd3

[etcd]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13 etcd_member_name=etcd3
k8s-node4 ansible_host=192.168.10.14 ip=192.168.10.14 etcd_member_name=etcd4
k8s-node5 ansible_host=192.168.10.15 ip=192.168.10.15 etcd_member_name=etcd5

EOF
```


- `cluster.yml` 실행
    -  실행하는 시점에서 `etcd` 노드는 짝수가 될 것이나 모든 기능은 정상적으로 작동해야하며 노드를 제거하기 전에 클러스터가 새로운 `etcd` 리더를 선출하려고 할때만 문제가 발생할 수 있다. (하지만 실행중 애플리케이션은 계속해서 사용가능)
    - 한번에 여러 ETCD 노드를 추가하는 경우에는 리트라이 횟수를 늘리는 옵션을 적용하자. 
        - `-e etcd_retries=10`

```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  --limit=etcd,kube_control_plane -e ignore_assert_errors=yes -e etcd_retries=10
```


- 모든 컨트롤 플레인에서 `kube-apiserver.yaml`을 편집한다. 
    - `etcd-servers=...`

```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

```bash
..
- --etcd-servers=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379, https://192.168.10.14:2379, https://192.168.10.15:2379
  ..
```

- 확인

```
ssh k8s-node1 cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd-servers

ssh k8s-node2 cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd-servers

ssh k8s-node3 cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd-servers
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T092337248Z.png)


- 확인 

```bash
for i in {1..5}; do echo ">> k8s-node$i <<"; ssh k8s-node$i etcdctl.sh endpoint status -w table; echo; done
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T092557374Z.png)

# ETCD 노드 삭제

ETCD 5대에서 다시 3대로 줄여보자 (홀수 유지)
- 노드가 꺼져있을 때에는 아래 옵션 추가
    - `-e node=k8s-node5 `
    - `-e reset_nodes=false`
    - `-e allow_ungraceful_removal=true`


- etcd 멤버에서 제거

>왜 인벤토리에 남겨둔 채로 실행하나? → Kubespray가 해당 노드에 접속해서 etcd 서비스 중지, 데이터 정리, 인증서 삭제 등 클린업을 수행해야 하기 때문

 
```bash
#k8s-node5 제거
ansible-playbook -i inventory/mycluster/inventory.ini remove-node.yml -e node=k8s-node5
```

```bash
#k8s-node4 제거
ansible-playbook -i inventory/mycluster/inventory.ini remove-node.yml -e node=k8s-node4
```


- 인벤토리 수정

```bash
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13 etcd_member_name=etcd3

[etcd]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13 etcd_member_name=etcd3
EOF
```


- 나머지 노드 설정 재생성

```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  --limit=etcd,kube_control_plane -e ignore_assert_errors=yes -e etcd_retries=10
```


- 컨트롤 플레인에서 etcd 주소 변경

```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```


```bash
- --etcd-servers=https://192.168.10.11:2379,https://192.168.10.12:2379,https://192.168.10.13:2379
```

- 확인

```
for i in {1..5}; do echo ">> k8s-node$i <<"; ssh k8s-node$i etcdctl.sh endpoint status -w table; echo; done
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T093933037Z.png)

>*중요*
인벤토리에서 먼저 삭제하면 `remove-node.yml`이 해당 노드에 접근할  수 없어 클린업이 안 된다. 반드시 제거 실행 후 인벤토리 삭제 순서를 지키자.