
>목표
>1. k8s-node3, k8s-node5 삭제(초기화) 후 상태 확인
>2. k8s-node5를 컨트롤 플레인 노드로 추가
>3. k8s-node3을 워커 노드로 추가


#  컨트롤 플레인 노드(k8s-node3) / 워커노드(k8s-node5) 삭제


- 워커노드(k8s-node5) 삭제

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v remove-node.yml -e node=k8s-node5 -e skip_confirmation=true
```


- 컨트롤 플레인(k8s-node3) 제거 

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v remove-node.yml -e node=k8s-node3 -e skip_confirmation=true
```

- 확인

```bash
kubectl get node -owide

# 삭제 확인
ssh k8s-node5 tree /etc/kubernetes
ssh k8s-node5 tree /var/lib/kubelet
ssh k8s-node5 pstree -a

ssh k8s-node3 tree /etc/kubernetes
ssh k8s-node3 tree /var/lib/kubelet
ssh k8s-node3 pstree -a
ssh k8s-node3 tree /var/lib/etcd
```

- 워커노드(`k8s-node4`)의 nginx 파드에 삭제된 컨트롤 플레인(k8s-node3)이 아직 존재함을 확인하자. 
    -  k8s-node4의 nginx.conf에는 아직 삭제한 컨트롤 플레인(k8s-node3)의 ip주소가 있다.( 자동변경 아님)

```bash
kubectl get pod -n kube-system -l k8s-app=kube-nginx -owide

kubectl logs -n kube-system -l k8s-app=kube-nginx --tail=1

kc describe pod -n kube-system -l k8s-app=kube-nginx
ssh k8s-node4 cat /etc/nginx/nginx.conf | grep upstream -A5
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T050111670Z.png)


- inventory.ini 수정
    - k8s-node3 , k8s-node5를 제거

```bash
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2

[etcd:children]
kube_control_plane

[kube_node]
k8s-node4 ansible_host=192.168.10.14 ip=192.168.10.14
EOF
```
# 컨트롤 플레인(k8s-node3 →k8s-node5) 노드 추가

- 새 컨트롤 플레인 노드를 인벤토리에 추가 후 `cluster.yml` 실행
    - 새 컨트롤 플레인 노드를 추가할 때는 항상 인벤토리의`kube_control_plane` 그룹 맨 끝에 추가
    - 컨트롤 플레인 노드를 첫 번째 위치에 추가하는 것은 지원되지 않으며 플레이북이 실패

- inventory.ini 수정

```bash
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node5 ansible_host=192.168.10.15 ip=192.168.10.15 etcd_member_name=etcd5

[etcd:children]
kube_control_plane

[kube_node]
k8s-node4 ansible_host=192.168.10.14 ip=192.168.10.14
EOF
```


- 모니터링

```
# 모니터링
watch -d "ssh k8s-node1 etcdctl.sh member list"
watch -d kubectl get node
```


- k8s-node5를 컨트롤 플레인 노드로 추가

```
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml -e kube_version="1.32.9"
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T051518419Z.png)


- 확인

```bash
# 노드확인
kubectl get node

# etcd 확인
ssh k8s-node1 etcdctl.sh member list -w table

# nginx.conf 서버 정보 확인
ssh k8s-node4 cat /etc/nginx/nginx.conf | grep upstream -A5
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T051600689Z.png)


# 후속작업

##  `admin-lb`의 `HAPrxoy` 에 대상 백엔드 정보 변경 
    
- `k8s-node3/5`를 삭제하고 워커노드였던 `k8s-node5`를 컨트롤 플레인으로 새롭게 추가하였으므로 HAProxy가 k8s-node5를 컨트롤 플레인으로 인식하도록 변경해야한다. 

```bash
cat /etc/haproxy/haproxy.cfg | grep k8s-node3
```

- admin-lb입장에서 아직 k8s-node3이 컨트롤 플레인으로 되어있는것을 알 수있다.
![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T051916302Z.png)

- `k8s-node5`로 변경하기

```bash
sed -i 's|server k8s-node3 192.168.10.13|server k8s-node5 192.168.10.15|g' /etc/haproxy/haproxy.cfg
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T052041361Z.png)


- haproxy 재로드 및 확인

```bash
systemctl reload haproxy
systemctl status haproxy --no-pager
```

## apiserver → etcd endpoint 에 변경 순차 작업

- etcd 정보 확인

```bash
ssh k8s-node1 etcdctl.sh member list
```

- 192.168.10.15(k8s-node5)의 etcd 정보를 확인

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T052823975Z.png)

- etcd가 없는 192.168.13:2379가 존재한다. 

```bash
ssh k8s-node1 cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd-server
ssh k8s-node2 cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd-server
ssh k8s-node5 cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd-server
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T052730128Z.png)


- 현재 존재하지 않는 k8s-node3 apiserver 의 HOLDER도 아직도 남아 있음을 확인

```bash
kubectl get leases -n kube-system --show-labels
kubectl delete leases -n kube-system apiserver-syplgv2uz3ssgciixtnxs4xeza
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T052957951Z.png)


- k8s-node3 apiserver 의 HOLDER 는 삭제

```bash
kubectl delete leases -n kube-system apiserver-syplgv2uz3ssgciixtnxs4xeza
#lease.coordination.k8s.io "apiserver-syplgv2uz3ssgciixtnxs4xeza" deleted
```

- apiserver 설정 변경 작업 진행 (중요!)
    - `k8s-node5` → `k8s-node2` → `k8s-node1`
    - 모니터링 
        - `watch -d crictl ps`

- `ssh k8s-node5` 진입

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep 192.168.10.13
sed -i 's|192.168.10.13|192.168.10.15|g' /etc/kubernetes/manifests/kube-apiserver.yaml
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep 192.168.10.15
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T053516868Z.png)


- 위 과정과 동일하게 `ssh k8s-node2`, `ssh k8s-node1`에서도 진행해주자.


- (참고) kubeadm-config의 cm 정보를 확인
    - 처음 설치할때 그대로인것을 확인

```
kubectl get cm -n kube-system kubeadm-config -o yaml
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T053823671Z.png)

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T053800402Z.png)

<br>
<br>

- 기존 `k8s-node3` 정보는 `k8s-node5`로 수정. `k8s-api-srv.admin-lb.com`는 추가
- 하지만 이후 kubespary `cluster.yml` 실행해도 변경된 값으로 적용되지는 않는다.

```bash
kubectl edit cm -n kube-system kubeadm-config
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T054151046Z.png)


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T054210702Z.png)


# 워커 노드(k8s-node5 → k8s-node3) 추가

- `inventory.ini` 수정

```bash
cat << EOF > /root/kubespray/inventory/mycluster/inventory.ini
[kube_control_plane]
k8s-node1 ansible_host=192.168.10.11 ip=192.168.10.11 etcd_member_name=etcd1
k8s-node2 ansible_host=192.168.10.12 ip=192.168.10.12 etcd_member_name=etcd2
k8s-node5 ansible_host=192.168.10.15 ip=192.168.10.15 etcd_member_name=etcd5

[etcd:children]
kube_control_plane

[kube_node]
k8s-node4 ansible_host=192.168.10.14 ip=192.168.10.14
k8s-node3 ansible_host=192.168.10.13 ip=192.168.10.13
EOF
```


- 워커노드 (k8s-node3)을 추가

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v scale.yml --limit=k8s-node3 -e kube_version="1.32.9"
```


- 노드 확인

```bash
kubectl get node
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260207T054646558Z.png)





