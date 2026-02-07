
>TL;DR


# 노드 추가

 기존 클러스터는 건드리지 않고 새로 추가된 노드만 단계적으로 합류 시킨다. 
 
- `/root/kubespray/scale.yml`
- `/root/kubespray/playbook/scale.yml`

## 새로운 워커노드 노드 추가하기 

- inventory.ini 수정

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


- 워커노드 추가수행

```bash
ANSIBLE_FORCE_COLOR=true ansible-playbook -i inventory/mycluster/inventory.ini -v scale.yml --limit=k8s-node5 -e kube_version="1.32.9" | tee kubespray_add_worker_node.log
```

- 확인 

```
kubectl get node -owide
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
EOF
```


# 비정상 노드 강제 삭제

위에서 추가 후 삭제했던 `k8s-node5`를 다시 추가하고 이 노드를 비정상 상태로 만든다.

- `inventory.ini` 수정

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

- 워커노드 추가 수행

```bash
ANSIBLE_FORCE_COLOR=true ansible-playbook -i inventory/mycluster/inventory.ini -v scale.yml --limit=k8s-node5 -e kube_version="1.32.9" | tee kubespray_add_worker_node.log
```


- 샘플파드 분배 

```bash
kubectl get pod -owide
kubectl scale deployment webpod --replicas 2
```

## 비정상 노드 만들기

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

