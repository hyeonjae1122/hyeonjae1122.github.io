---
layout: post
title: "[K8S Deploy Study by Gasida] - Kubernetes kubeadm 구현 상세 정리"
categories:
  - "\bAnsible"
tags:
  - k8s
  - kubeadm
---
# 1. 핵심 설계 원칙
## 보안성
- `RBAC` (역할 기반 접근 제어) 강제
- `Node Authorizer` 사용
- 컨트롤 플레인 컴포넌트 간 안전한 통신
- `API 서버`와 `kubelet` 간 안전한 통신
- kubelet API 잠금
- `kube-proxy`, `CoreDNS` 등 시스템 컴포넌트 접근 제한
- `Bootstrap Token` 접근 제한
## 사용자 편의성

- 사용자는 아래의 몇가지의 최소한의 명령만 실행하면 된다.

```bash
kubeadm init

export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl apply -f <network-plugin>.yaml

kubeadm join --token <token> <endpoint>:<port>
```

## 확장성
- 특정 네트워크 제공자를 강제하지 않음
- 설정 파일을 통해 다양한 매개변수 커스터마이징 가능

# 2.  주요 디렉터리 및 파일

kubeadm은 복잡성을 줄이고 이를 기반한 상위 수준 도구의 개발을 단순화하기 위해 잘 알려진 경로 및 파일 이름에 대해 제한된 수의 상수 값을 사용한다.

## 주요 디렉터리

- `/etc/kubernetes` :  k8s 기본 디렉터리 ( 대부분의 경우 명확하게 지정된 경로이며 직관적임)
- `/etc/kubernetes/manifests` : Static Pod manifests 위치
- `/etc/kubernetes/pki` : 인증서 및 키 파일 저장

## Static Pod Manifest 파일

```bash
/etc/kubernetes/manifests/
├── etcd.yaml
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
└── kube-scheduler.yaml
```

## kubeconfig 파일

```bash
/etc/kubernetes/
├── bootstrap-kubelet.conf    (TLS 부트스트랩 중)
├── kubelet.conf              (kubelet용)
├── controller-manager.conf   (컨트롤러 매니저용)
├── scheduler.conf            (스케줄러용)
├── admin.conf                (클러스터 관리자용)
└── super-admin.conf          (비상용 수퍼 관리자)
```


## 인증서 파일

```bash
/etc/kubernetes/pki/
├── ca.crt, ca.key                          # CA 인증서
├── apiserver.crt, apiserver.key            # API 서버
├── apiserver-etcd-client.crt, .key         # API 서버 ↔ etcd
├── apiserver-kubelet-client.crt, .key      # API 서버 ↔ kubelet
├── sa.pub, sa.key                          # ServiceAccount
├── front-proxy-ca.crt, .key                # 프론트 프록시 CA
├── front-proxy-client.crt, .key            # 프론트 프록시 클라이언트
└── etcd/
    ├── ca.crt, ca.key
    ├── server.crt, server.key
    ├── peer.crt, peer.key
    └── healthcheck-client.crt, .key
```



# 3. kubeadm init 워크플로우 상세

## Phase 1 Preflight Checks(사전점검)

`--ignore-preflight-errors` 플래그를 사용함으로써 건너뛸 수 있음

- k8s 시스템 요구사항
    - 커널 버전확인
    - cgroups 서브시스템 설정 확인
    - CRI 엔드포인트 응답 확인
    - root 사용자 확인
    - 호스트명이 유효한 DNS subdomain인지 확인
    - 호스트명이 네트워크를 통해 reach 가능한지 확인
    - kubelet 버전 호환성 확인
    - firewalld 활성화 확인 
    - 필요한 포트 사용 가능 확인
    - swap 비활성화 확인
- etcd 관련 점검
    - `2379` 포트 사용가능 확인
    - `/var/lib/etcd` 디렉터리가 비어있는지 확인

## Phase2 인증서 생성

`/etc/kubernetes/pki`에 저장

인증서 
- `ca.crt`  `ca.key` : 클러스터 CA
- `apiserver.crt` :  API Server (CN: `kubernetes`)
- `apiserver-kubelet-client.crt` : API Server 에서 kubelet (CN : `system:masters`)
- `sa.key`, `sa.pub` : ServiceAccount 서명 
- `front-proxy-ca.crt` / `front-proxy-ca.crt`: 프론트 프록시 CA
- `front-proxy-client.crt` / `front-proxy-client.key`


**API 서버 인증서에 포함되어야 할 SAN (Subject Alternative Name):**

```
- kubernetes                     # DNS
- kubernetes.default             # DNS
- kubernetes.default.svc         # DNS
- kubernetes.default.svc.cluster.local  # DNS
- 10.96.0.1                     # Service CIDR 첫 번째 IP
- <apiserver-advertise-address> # 예: 192.168.10.11
- <node-name>                   # 예: k8s-ctr1
- <user-custom-names>           # 사용자 커스텀
```
## Phase 3 kubeconfig 파일 생성

| 파일                      | 용도                | CN                            |
| ----------------------- | ----------------- | ------------------------------ |
| bootstrap-kubelet.conf  | kubelet TLS 부트스트랩 | system:node:{hostname}         |
| controller-manager.conf | 컨트롤러 매니저          | system:kube-controller-manager |
| scheduler.conf          | 스케줄러              | system:kube-scheduler          |
| admin.conf              | 클러스터 관리자          | kubernetes-admin               |
| super-admin.conf        | 비상용 (RBAC 우회)     | kubernetes-super-admin         |
**⚠️ 주의:** 이 파일들은 인증 키를 포함하므로 기밀 취급 필요


## Phase4 Static Pod Manifest 생성

공통 속성 

```
- namespace: kube-system
- hostNetwork: true            # 컨트롤 플레인 조기 시작을 위해 필요
- tier: control-plane
- component: <component-name>
- priorityClassName: system-node-critical
```

### API Server

```
매개변수:
- --apiserver-advertise-address=<IP>
- --apiserver-bind-port=6443
- --service-cluster-ip-range=10.96.0.0/16  # Service CIDR
- --etcd-servers=https://127.0.0.1:2379    # 로컬 etcd 사용

강제 설정:
- --insecure-port=0           # 안전하지 않은 연결 차단
- --enable-bootstrap-token-auth=true
- --allow-privileged=true
- --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname

인증서 설정:
- --client-ca-file=/etc/kubernetes/pki/ca.crt
- --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
- --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

### Controller Manager

```bash
# --pod-network-cidr가 지정되면:
- --allocate-node-cidrs=true
- --cluster-cidr=<CIDR>
- --node-cidr-mask-size=<size>

# 강제 설정:
- --controllers=*,bootstrapsigner,tokencleaner
- --use-service-account-credentials=true
```

### 스케쥴러 설정

사용자 매개변수에 영향을 받지 않음

### Local etcd 설정

```bash
--listen-client-urls=https://127.0.0.1:2379,https://<advertise-ip>:2379
--advertise-client-urls=https://<advertise-ip>:2379
--listen-peer-urls=https://<advertise-ip>:2380
--initial-advertise-peer-urls=https://<advertise-ip>:2380
--data-dir=/var/lib/etcd
```

## Phase 5 kubeadm-config ConfigMap 저장

목적 : 향후 kubeadm upgrade 등에서 클러스터 상태 파악하기 위함이다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubeadm-config
  namespace: kube-system
data:
  ClusterConfiguration: |
    apiVersion: kubeadm.k8s.io/v1beta4
    kind: ClusterConfiguration
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controlPlaneEndpoint: 192.168.10.10:6443
    ...
```

## Phase 6 Control Plane 노드 마킹

```yaml
# 라벨 추가:
- node-role.kubernetes.io/control-plane: ""

# 테인트 추가:
- node-role.kubernetes.io/control-plane:NoSchedule
```
## Phase 7 Bootstrap Token 설정

생성되는 시크릿 

```yaml
name: bootstrap-token-<token-id>
namespace: kube-system
```

특징 : 
- 기본 유효 기간: 24시간
- 그룹: `system:bootstrappers:kubeadm:default-node-token`

## Phase 8 자동 승인 설정

생성되는 ClusterRoleBinding

```
# 1. kubeadm:kubelet-bootstrap
   - 역할: system:node-bootstrapper
   - 그룹: system:bootstrappers:kubeadm:default-node-token
   
2. kubeadm:node-autoapprove-bootstrap
   - 역할: system:certificates.k8s.io:certificatesigningrequests:nodeclient
   - 그룹: system:bootstrappers:kubeadm:default-node-token
   
3. kubeadm:node-autoapprove-certificate-rotation
   - 역할: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
   - 그룹: system:nodes
```
## Phase 9 Add-on 설치
## Phase 10

