---
layout: post
tags:
  - Ansible
  - kubespray
  - kubernetes
  - k8s
title: 2026-02-07-Kubespary HA & Upgrade - 1 노드 관리
---

> kubespray를 활용하여 새로운 노드를 추가하고 삭제한다. 
 


# 노드 추가

 기존 클러스터는 건드리지 않고 새로 추가된 노드만 단계적으로 합류 시킨다. 
 
- `/root/kubespray/scale.yml`
- `/root/kubespray/playbook/scale.yml`

## 새로운 워커노드 노드 추가하기 

- inventory.ini 수정 (k8s-node5 추가)

```bash
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13 etcd_member_name=etcd3

[etcd:children]
kube_control_plane

[kube_node]
k8s-node4 ansible_host=192.168.10.14 ip=192.168.10.14
k8s-node5 ansible_host=192.168.10.15 ip=192.168.10.15
EOF
```

- 확인 및 모니터링

```bash
ansible-inventory -i /root/kubespray/inventory/mycluster/inventory.ini --graph

# ansible 연결 확인
ansible -i inventory/mycluster/inventory.ini k8s-node5 -m ping

# 모니터링
watch -d kubectl get node
```


- 워커노드 추가수행

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v scale.yml --list-tasks

ANSIBLE_FORCE_COLOR=true ansible-playbook -i inventory/mycluster/inventory.ini -v scale.yml --limit=k8s-node5 -e kube_version="1.32.9" | tee kubespray_add_worker_node.log
```

- 확인 

```bash
kubectl get node -owide
kubectl get pod -n kube-system -owide |grep k8s-node5

# 변경 정보 확인
ssh k8s-node5 tree /etc/kubernetes
ssh k8s-node5 tree /var/lib/kubelet
ssh k8s-node5 pstree -a
```

- 샘플 파드 분배

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webpod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webpod
  template:
    metadata:
      labels:
        app: webpod
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - sample-app
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: webpod
        image: traefik/whoami
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webpod
  labels:
    app: webpod
spec:
  selector:
    app: webpod
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30003
  type: NodePort
EOF
```

```bash
kubectl get pod -owide
kubectl scale deployment webpod --replicas 1
kubectl get pod -owide
kubectl scale deployment webpod --replicas 2
```

# 노드 삭제 

- `/root/kubespray/remove-node.yml`
- `/root/kubespray/playbooks/remove_node.yml

## 새로 추가된 워커노드 삭제하기

### 워커 노드 삭제 순서
1) `inventory.ini` 수정
2) webpod 디플로이먼트에 pdb 설정
3) 노드삭제 수행
4) 노드삭제 실패 확인
5) webpod 디플로이먼트 pdb 삭제
6) 노드삭제 다시 수행
7) 노드삭제 성공 확인
8) `inventory.ini` 복구

#### inventory.ini 수정

```bash
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13 etcd_member_name=etcd3

[etcd:children]
kube_control_plane

[kube_node]
k8s-node4 ansible_host=192.168.10.14 ip=192.168.10.14
k8s-node5 ansible_host=192.168.10.15 ip=192.168.10.15
EOF
```
#### webpod deployment에 pdb 설정

-  해당 정책은 최소 2개의 Pod가 Ready상태이어야함
-  drain / eviction 시 단 하나의 Pod도 축출 불가


```bash
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webpod
  namespace: default
spec:
  maxUnavailable: 0
  selector:
    matchLabels:
      app: webpod
EOF
```

#### 노드 삭제 수행

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v remove-node.yml -e node=k8s-node5
```


 #### 실패확인

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T034637733Z.png)


#### pdb삭제 

```bash
kubectl delete pdb webpod
```


#### 노드 삭제 수행

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v remove-node.yml -e node=k8s-node5
```

#### 노드 삭제 성공 확인

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T034954830Z.png)

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T035012610Z.png)



#### inventory.ini 원래대로 돌려 놓기

```bash
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13 etcd_member_name=etcd3

[etcd:children]
kube_control_plane

[kube_node]
k8s-node4 ansible_host=192.168.10.14 ip=192.168.10.14
k8s-node5 ansible_host=192.168.10.15 ip=192.168.10.15
EOF
```

- 워커 노드 추가 수행 : 3분 정도 소요

```bash
ANSIBLE_FORCE_COLOR=true ansible-playbook -i inventory/mycluster/inventory.ini -v scale.yml --limit=k8s-node5 -e kube_version="1.32.9" | tee kubespray_add_worker_node.log
```

- 확인 및 샘플 파드 재분배
```bash
kubectl get node -owide
```

- 파드 배포

```bash
kubectl get pod -owide
kubectl scale deployment webpod --replicas 1
kubectl get pod -owide
kubectl scale deployment webpod --replicas 2
```


# 비정상 노드 강제 삭제

먼저 실습을 위해 `k8s-node5` 노드를 비정상 상태로 만든다.
- `kubelet`과 `containerd` 를중단

```bash
ssh k8s-node5 systemctl stop kubelet
ssh k8s-node5 systemctl stop containerd
```


- 모니터링 확인

```bash
watch -d kubectl get node
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T040459368Z.png)


- taint 추가 확인

```bash
kc describe node k8s-node5

#Taints:              
# node.kubernetes.io/unreachable:NoExecute                # node.kubernetes.io/unreachable:NoSchedule
```


- 삭제시도

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v remove-node.yml -e node=k8s-node5 -e skip_confirmation=true
```

- 노드 drain에서 삭제 실패

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T073150812Z.png)

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T073759238Z.png)

- role 내에 defaults 에 변수 확인

```bash
cat roles/remove_node/pre_remove/defaults/main.yml
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T073825470Z.png)


#### 강제로 삭제하기

- `reset_nodes=false`
    - 클러스터 제어부에서 메타데이터만 정리, 노드에 삭제 시도 안함
        - `kubeadm reset` ❌
        - 서비스 정리 ❌ 
        - SSH 접속 ❌

- `allow_ungraceful_removal=true`
    - drain / 정상 종료 못 해도 그냥 제거 허용, Pod eviction 실패 무시, kubelet 응답 없어도 계속 진행


- 위의 옵션을 주어 강제로 삭제 진행

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v remove-node.yml -e node=k8s-node5 -e reset_nodes=false -e allow_ungraceful_removal=true -e skip_confirmation=true
```

(참고):  노드 제거 실행 TASK: `cat roles/remove-node/post-remove/tasks/main.yml`

- 노드확인 

```
kubectl get node
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T074809413Z.png)

- `k8s-node5` 상태 확인 
    - 아직 이전 상태로 남아 있음

```bash
ssh k8s-node5 systemctl status kubelet --no-pager
ssh k8s-node5 tree /etc/kubernetes
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T074838862Z.png)


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T074833006Z.png)



# k8s-node5 초기화

## 제거 대상 노드에서 kubeadm reset

- `ssh k8s-node5`에 진입

```bash
kubeadm reset -f

# 아래 업데이트 필요
# 디렉터리/파일 삭제
rm -rf /etc/cni/net.d
rm -rf /etc/kubernetes/
rm -rf /var/lib/kubelet

# iptables 정리
iptables -t nat -S
iptables -t filter -S
iptables -F
iptables -t nat -F
iptables -t mangle -F
iptables -X

# 서비스 중지
systemctl status containerd --no-pager 
systemctl status kubelet --no-pager

# kubelet 서비스 중지 -> 제거해보자
systemctl stop kubelet && systemctl disable kubelet

# contrainrd 서비스 중지 -> 제거해보자
systemctl stop containerd && systemctl disable containerd

# reboot
reboot
```
## host OS에서 k8s-node5 제거후 새로 생성

```bash
# vagrant 로 k8s-node5 제거
vagrant destroy -f k8s-node5
vagrant status

# vagrant 로 k8s-node5 다시 생성(프로비저닝, 스크립트 반영)
vagrant up k8s-node5
```

## admin-lb에서 워커노드 추가 실행

```bash
# k8s-node5 ping 통신 확인
ping -c 1 192.168.10.15
PING 192.168.10.15 (192.168.10.15) 56(84) bytes of data.
64 bytes from 192.168.10.15: icmp_seq=1 ttl=64 time=0.461 ms

# 실패!
ansible -i inventory/mycluster/inventory.ini k8s-node5 -m ping

# 성공!
sshpass -p 'qwe123' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.10.15
nsible -i inventory/mycluster/inventory.ini k8s-node5 -m ping


# 워커 노드 추가 수행 : 3분 정도 소요
ANSIBLE_FORCE_COLOR=true ansible-playbook -i inventory/mycluster/inventory.ini -v scale.yml --limit=k8s-node5 -e kube_version="1.32.9" | tee kubespray_add_worker_node.log


# 확인
kubectl get node -owide

# 변경 정보 확인
ssh k8s-node5 tree /etc/kubernetes
ssh k8s-node5 tree /var/lib/kubelet
ssh k8s-node5 pstree -a


# 샘플 파드 분배
kubectl get pod -owide
kubectl scale deployment webpod --replicas 1
kubectl get pod -owide
kubectl scale deployment webpod --replicas 2
```