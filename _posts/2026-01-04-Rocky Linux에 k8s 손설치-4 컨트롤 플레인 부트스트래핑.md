
# k8s 컨트롤 플레인 기동

## server 노드에 api server, scheduler, kcm 서비스 기동

| 항목               | 네트워크 대역 or IP     |
| ---------------- | ----------------- |
| **clusterCIDR**  | **10.200.0.0/16** |
| → node-0 PodCIDR | 10.200.0.0/24     |
| → node-1 PodCIDR | 10.200.1.0/24     |
| **ServiceCIDR**  | **10.32.0.0/24**  |
| → api clusterIP  | 10.32.0.1         |

```bash
pwd 
# /root/kubernetes-the-hard-way

# 각종 서비스 및 yaml 확인
cat units/kube-apiserver.service
cat units/kube-scheduler.service ; echo
cat units/kube-controller-manager.service ; echo
cat configs/kube-apiserver-to-kubelet.yaml
cat configs/kube-scheduler.yaml ; echo
```


- `units/kube-apiserver.service` 수정
	- --service-cluster-ip-range=10.32.0.0/24 
	

```bash
cat ca.conf | grep '\[kube-api-server_alt_names' -A2
[kube-api-server_alt_names]
IP.0  = 127.0.0.1
IP.1  = 10.32.0.1

cat << EOF > units/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --allow-privileged=true \\
  --apiserver-count=1 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.crt \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-servers=http://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \\
  --kubelet-client-certificate=/var/lib/kubernetes/kube-api-server.crt \\
  --kubelet-client-key=/var/lib/kubernetes/kube-api-server.key \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-accounts.crt \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-accounts.key \\
  --service-account-issuer=https://server.kubernetes.local:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kube-api-server.crt \\
  --tls-private-key-file=/var/lib/kubernetes/kube-api-server.key \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
cat units/kube-apiserver.service
```

- `configs/kube-apiserver-to-kubelet.yaml`  확인

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"  # Kubernetes가 업그레이드 시 자동 관리
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""                                               # Core API group (v1) : Node 관련 서브리소스는 core group에 속함
    resources:                                           # 아래 처럼, kubelet API 대부분을 포괄
      - nodes/proxy                                      ## apiserver → kubelet 프록시 통신
      - nodes/stats                                      ## 노드/파드 리소스 통계 (cAdvisor)
      - nodes/log                                        ## metrics-server / top 명령
      - nodes/spec                                       ## kubectl logs
      - nodes/metrics                                    ## metrics-server / top 명령
    verbs:
      - "*"                                              # 대상은 “nodes 하위 리소스”로 한정 + 모든 동작 허용 (get, list, watch, create, proxy 등)
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver                            # 누가 이 권한을 쓰는가? → kube-apiserver 자신
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes                         # 사용자 kubernetes ,이 사용자는 kube-apiserver가 사용하는 클라이언트 인증서의 CN

# api-server : Subject CN 확인
openssl x509 -in kube-api-server.crt -text -noout
        Subject: CN = kubernetes,

# api -> kubelet 호출 시 Flow
kube-apiserver (client)
  |
  | (TLS client cert, CN=kubernetes)
  ↓
kubelet API Server 역할 (/stats, /log, /metrics)
  |
  ↓
RBAC 평가:
  User = kubernetes
  → ClusterRoleBinding system:kube-apiserver 매칭
  → ClusterRole system:kube-apiserver-to-kubelet 권한 부여
```

- `units/kube-scheduler.service` , `configs/kube-scheduler.yaml`

```
# kube-controller-manager : cluster-cidr 는 POD CIDR 포함하는 대역, service-cluster-ip-range 는 apiserver 설정 값 동일 설정.
cat units/kube-controller-manager.service ; echo
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --bind-address=0.0.0.0 \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \
  --cluster-signing-key-file=/var/lib/kubernetes/ca.key \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --root-ca-file=/var/lib/kubernetes/ca.crt \
  --service-account-private-key-file=/var/lib/kubernetes/service-accounts.key \
  --service-cluster-ip-range=10.32.0.0/24 \
  --use-service-account-credentials=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```


server(컨트롤 플레인)으로 아래 리스트를 복사한다.
- kube-apiserver 바이너리 파일
- kube-controller-manager 바이너리 파일
- kube-scheduler 바이너리파일
- kubectl 바이너리 파일
- kube-apiserver.service
- kube-controller-manager.service
- kube-scheduler.service
- kube-scheduler.yaml
- kube-apiserver-to-kubelet.yaml : kube-apiserver가 kubelet(Node)에 접근할 수 있도록 허용하는 '시스템 내부용 RBAC' 설정 파일


```bash
scp \
  downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  units/kube-apiserver.service \
  units/kube-controller-manager.service \
  units/kube-scheduler.service \
  configs/kube-scheduler.yaml \
  configs/kube-apiserver-to-kubelet.yaml \
  root@server:~/

# 확인
ssh server ls -l /root

```

### server(컨트롤 플레인)에서 kubectl 확인

```bash
ssh root@server
---------------------------------------------------------------
# Create the Kubernetes configuration directory:
pwd
mkdir -p /etc/kubernetes/config


# Install the Kubernetes binaries:
mv kube-apiserver \
  kube-controller-manager \
  kube-scheduler kubectl \
  /usr/local/bin/
ls -l /usr/local/bin/kube-*


# Configure the Kubernetes API Server
mkdir -p /var/lib/kubernetes/
mv ca.crt ca.key \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  encryption-config.yaml \
  /var/lib/kubernetes/
ls -l /var/lib/kubernetes/

## Create the kube-apiserver.service systemd unit file:
mv kube-apiserver.service \
  /etc/systemd/system/kube-apiserver.service
tree /etc/systemd/system


# Configure the Kubernetes Controller Manager

## Move the kube-controller-manager kubeconfig into place:
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

## Create the kube-controller-manager.service systemd unit file:
mv kube-controller-manager.service /etc/systemd/system/


# Configure the Kubernetes Scheduler

## Move the kube-scheduler kubeconfig into place:
mv kube-scheduler.kubeconfig /var/lib/kubernetes/

## Create the kube-scheduler.yaml configuration file:
mv kube-scheduler.yaml /etc/kubernetes/config/

## Create the kube-scheduler.service systemd unit file:
mv kube-scheduler.service /etc/systemd/system/


# Start the Controller Services : Allow up to 10 seconds for the Kubernetes API Server to fully initialize.
systemctl daemon-reload
systemctl enable kube-apiserver kube-controller-manager kube-scheduler
systemctl start  kube-apiserver kube-controller-manager kube-scheduler

# 확인
ss -tlp | grep kube


systemctl is-active kube-apiserver
systemctl status kube-apiserver --no-pager
journalctl -u kube-apiserver --no-pager

systemctl status kube-scheduler --no-pager
systemctl status kube-controller-manager --no-pager

# Verify this using the kubectl command line tool:
kubectl cluster-info dump --kubeconfig admin.kubeconfig
kubectl cluster-info --kubeconfig admin.kubeconfig
Kubernetes control plane is running at https://127.0.0.1:6443

kubectl get node --kubeconfig admin.kubeconfig
kubectl get pod -A --kubeconfig admin.kubeconfig

kubectl get service,ep --kubeconfig admin.kubeconfig

# clusterroles 확인
kubectl get clusterroles --kubeconfig admin.kubeconfig
kubectl describe clusterroles system:kube-scheduler --kubeconfig admin.kubeconfig


# kube-scheduler subject 확인
kubectl get clusterrolebindings --kubeconfig admin.kubeconfig
kubectl describe clusterrolebindings system:kube-scheduler --kubeconfig admin.kubeconfig
---------------------------------------------------------------
```

### Kubelet 인가를 위한 RBAC

- Kubernetes API 서버가 각 작업자 노드에서 Kubelet API에 액세스할 수 있도록 RBAC 권한을 구성
- Kubelet API에 대한 액세스 권한은 메트릭, 로그를 검색하고 포드에서 명령을 실행하는 데 필요

```bash
ssh root@server # 이미 server 에 ssh 접속 상태
---------------------------------------------------------------
# api -> kubelet 접속을 위한 RBAC 설정
# Create the system:kube-apiserver-to-kubelet ClusterRole with permissions to access the Kubelet API and perform most common tasks associated with managing pods:
cat kube-apiserver-to-kubelet.yaml
kubectl apply -f kube-apiserver-to-kubelet.yaml --kubeconfig admin.kubeconfig

# 확인
kubectl get clusterroles system:kube-apiserver-to-kubelet --kubeconfig admin.kubeconfig
kubectl get clusterrolebindings system:kube-apiserver --kubeconfig admin.kubeconfig

---------------------------------------------------------------
```


### jumpbox 서버에서 k8s controlplane 정상동작 확인

- `vagrant ssh jumpbox`

접속이 불가할 경우 방화벽 중지 후 curl 테스트. 

```bash
# 방화벽 중단 및 비활성화
sudo systemctl stop firewalld
sudo systemctl disable firewalld

curl -s -k --cacert ca.crt https://server.kubernetes.local:6443/version | jq
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T163806078Z.png)
