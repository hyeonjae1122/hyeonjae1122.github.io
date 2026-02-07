
>목표
>첫 번째 컨트롤 플레인 노드를 교체한다.


# 첫번째 컨트롤 플레인 및 etcd-master 추가 및 제거

## inventory.ini 수정

- 만약 `k8s-node1` 노드를 제거하고 싶다면 아래와 같이 kube_control_plane의 순서를 변경한다.

```bash 
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13 etcd_member_name=etcd3
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1

[etcd:children]
kube_control_plane

[kube_node]
k8s-node4 ansible_host=192.168.10.14 ip=192.168.10.14
EOF
```

## 클러스터 업그레이드

- `upgrade-cluster.yml`이나 `cluster.yml`을 수행한다. 

```bash
ANSIBLE_FORCE_COLOR=true ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml -e kube_version="1.32.9" | tee kubespray_install.log
```

## 기존의 컨트롤 플레인을 제거

- `remove-node.yml`은 기존 노드가 인벤토리에 남아있는 상태에서 실행한다.
- 제거하려는 노드에 대해서만 실행되도록 플레이북에 옵션을 전달해야함. 
    - `-e node=k8s-node1`
    - (optional) `reset_nodes=false` , `allow_ungraceful_removal=true` 제거노드가 오프라인 인경우

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v remove-node.yml -e node=k8s-node1 -e skip_confirmation=true
```

## configmap 수정

- `kube-public` 네임스페이스에서 `cluster-info`의 `configmap`을 편집한다  
- 기존 `kube_control_plane` 노드의 IP 주소를 현재 `kube_control_plane` 노드의 IP 주소로 변경한다. 
- 또한 `certificate-authority-data` 인증서를 변경한 경우 해당 필드도 업데이트한다.

```bash
kubectl  edit cm -n kube-public cluster-info
```

- 확인
```
kubectl get cm -n kube-public cluster-info -o yaml | grep server
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T083505539Z.png)

## 새로운 컨트롤 플레인 추가

- (optional) 필요하다면 인벤토리를 업데이트한다.
- `--limit=kube_controle_plane`옵션과 함께 `cluster.yml`을 실행한다.

```bash
ANSIBLE_FORCE_COLOR=true ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml -e kube_version="1.32.9" --limit=kube_control_plane | tee kubespray_install.log
```


# 확인

- etcd 그룹 첫 번째 확인

```bash
ansible -i inventory/mycluster/inventory.ini etcd \
  -m debug -a "var=groups['etcd'][0]"
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T084753510Z.png)

- kube_control_plane 그룹 첫 번째 확인 

```bash
ansible -i inventory/mycluster/inventory.ini kube_control_plane \
  -m debug -a "var=groups['kube_control_plane'][0]"
```
![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T084812715Z.png)