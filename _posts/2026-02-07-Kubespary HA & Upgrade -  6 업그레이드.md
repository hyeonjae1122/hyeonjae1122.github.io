---
layout: post
tags:
  - Ansible
  - kubespray
  - kubernetes
  - k8s
title:
categories:
  - kubernetes
---

# 사전작업 Flannel CNI Plugin 업그레이드

- flannel 관련 변수 검색

```bash
grep -Rni "flannel" inventory/mycluster/ playbooks/ roles/ --include="*.yml" -A2 -B1

```


```bash
#roles/kubespray_defaults/defaults/main/download.yml:115:flannel_version: 0.27.3
#roles/kubespray_defaults/defaults/main/download.yml:116:flannel_cni_version: 1.7.1-flannel1
```


- 정보확인
-  flannel 설정 수정 

```bash
cat << EOF >> inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml flannel_version: 0.27.4 
EOF
```

```bash
grep "^[^#]" inventory/mycluster/group_vars/k8s_cluster/k8s-net-flannel.yml
```


```bash
# flannel_interface: enp0s9
# flannel_version: 0.27.4
```

- 모니터링

```bash
watch -d "ssh k8s-node3 crictl ps"
```


```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v upgrade-cluster.yml --tags "flannel" --list-tasks

ansible-playbook -i inventory/mycluster/inventory.ini -v upgrade-cluster.yml --tags "flannel" --limit k8s-node3 -e kube_version="1.32.9"
```


- flannel 은 데몬셋 이므로 특정 대상 노드로 수행 불가 
    - 민감한 클러스터 환경이라면 cni plugin 은 kubespary 와 별도 배포 관리 후 특정 노드별 순차 적용 해야 한다.
```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v upgrade-cluster.yml --tags "flannel" -e kube_version="1.32.9"
```


- 확인

```bash
kubectl get ds -n kube-system -owide
ssh k8s-node1 crictl images
kubectl get pod -n kube-system -l app=flannel -owide
```


