

# CA 설정 및 TLS 인증서 생성

- 총 9개의 항목에 대해 개인키 및 인증서를 생성한다. 

| 항목                      | 개인키                         | CSR                         | 인증서                         | 참고 정보                                                                      | X509v3 Extended Key Usage                              |
| ----------------------- | --------------------------- | --------------------------- | --------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------ |
| Root CA                 | ca.key                      | X                           | ca.crt                      |                                                                            |                                                        |
| admin                   | admin.key                   | admin.csr                   | admin.crt                   | CN = admin, O = system:masters                                             | TLS **Web Client** Authentication                      |
| node-0                  | node-0.key                  | node-0.csr                  | node-0.crt                  | CN = system:node:node-0, O = system:nodes                                  | TLS **Web** ==**Server**== **/ Client** Authentication |
| node-1                  | node-1.key                  | node-1.csr                  | node-1.crt                  | CN = system:node:node-1, O = system:nodes                                  | TLS **Web** ==**Server**== **/ Client** Authentication |
| kube-proxy              | kube-proxy.key              | kube-proxy.csr              | kube-proxy.crt              | CN = system:kube-proxy, O = system:node-proxier                            | TLS **Web** ==Server== / **Client** Authentication     |
| kube-scheduler          | kube-scheduler.key          | kube-scheduler              | kube-scheduler.crt          | CN = system:kube-scheduler, O = system:kube-scheduler                      | TLS **Web** ==Server== / **Client** Authentication     |
| kube-controller-manager | kube-controller-manager.key | kube-controller-manager.csr | kube-controller-manager.crt | CN = system:kube-controller-manager, O = system:kube-controller-manager    | TLS **Web** ==Server== / **Client** Authentication     |
| kube-api-server         | kube-api-server.key         | kube-api-server.csr         | kube-api-server.crt         | CN = kubernetes, SAN: IP(127.0.0.1, ==**10.32.0.1**==), DNS(kubernetes,..) | TLS **Web** ==**Server**== **/ Client** Authentication |
| service-accounts        | service-accounts.key        | service-accounts.csr        | service-accounts.crt        | CN = service-accounts                                                      | TLS **Web Client** Authentication                      |
|                         |                             |                             |                             |                                                                            |                                                        |


| 항목                  | 네트워크 대역 or IP     |
| ------------------- | ----------------- |
| **clusterCIDR**     | 10.200.0.0/16     |
| → node-0 PodCIDR    | 10.200.0.0/24     |
| → node-1 PodCIDR    | 10.200.1.0/24     |
| **ServiceCIDR**     | **10.32.0.0/24**  |
| → **api clusterIP** | ==**10.32.0.1**== |

`ca.conf`  내용 살펴보기

| 구분                          | 역할                   |
| --------------------------- | -------------------- |
| `[req]`                     | OpenSSL 요청 기본 동작     |
| `[ca_*]`                    | CA 인증서               |
| `[admin]`                   | 관리자 (kubectl)        |
| `[service-accounts]`        | ServiceAccount 토큰 서명 |
| `[node-*]`                  | 워커 노드(kubelet)       |
| `[kube-proxy]`              | kube-proxy           |
| `[kube-controller-manager]` | 컨트롤러                 |
| `[kube-scheduler]`          | 스케줄러                 |
| `[kube-api-server]`         | API Server           |
| `[default_req_extensions]`  | 공통 CSR 옵션            |

```bash
[req]
distinguished_name = req_distinguished_name
prompt             = no                      # CSR 생성 시 대화형 입력 없음
x509_extensions    = ca_x509_extensions      # CA 인증서 생성 시 사용할 확장

[ca_x509_extensions]                         # CA 인증서 설정 (Root of Trust)
basicConstraints = CA:TRUE                   # CA 권한 인증서
keyUsage         = cRLSign, keyCertSign      # 다른 인증서를 서명 가능, Kubernetes 모든 인증의 신뢰 루트

[req_distinguished_name]
C   = US
ST  = Washington
L   = Seattle
CN  = CA                                     # 클러스터 CA

[admin]                                      # Admin 사용자 (kubectl)
distinguished_name = admin_distinguished_name
prompt             = no
req_extensions     = default_req_extensions

[admin_distinguished_name]
CN = admin                                   # [K8S] CN → user
O  = system:masters                          # [K8S] O → group , system:masters - Kubernetes 슈퍼유저 그룹, 모든 RBAC 인가 우회

# Service Accounts
#
# The Kubernetes Controller Manager leverages a key pair to generate
# and sign service account tokens as described in the
# [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/)
# documentation.

[service-accounts]                           # Service Account 서명자
distinguished_name = service-accounts_distinguished_name
prompt             = no
req_extensions     = default_req_extensions

[service-accounts_distinguished_name] 
CN = service-accounts                        # controller-manager가 사용하는 ServiceAccount 토큰 서명용 인증서 , apiserver에서 --service-account-key-file 로 사용

# Worker Nodes
#
# Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/)
# called Node Authorizer, that specifically authorizes API requests made
# by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet).
# In order to be authorized by the Node Authorizer, Kubelets must use a credential
# that identifies them as being in the `system:nodes` group, with a username
# of `system:node:<nodeName>`.

[node-0]                                    # Worker Node 인증서 (kubelet)
distinguished_name = node-0_distinguished_name
prompt             = no
req_extensions     = node-0_req_extensions

[node-0_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth  # clientAuth: apiserver → kubelet & serverAuth: kubelet HTTPS 서버(10250)
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Node-0 Certificate"
subjectAltName       = DNS:node-0, IP:127.0.0.1
subjectKeyIdentifier = hash

[node-0_distinguished_name]
CN = system:node:node-0                     # kubelet 사용자 , CN = system:node:<nodeName>
O  = system:nodes                           # Node Authorizer 그룹  ,O = system:nodes
C  = US
ST = Washington
L  = Seattle

[node-1]
distinguished_name = node-1_distinguished_name
prompt             = no
req_extensions     = node-1_req_extensions

[node-1_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Node-1 Certificate"
subjectAltName       = DNS:node-1, IP:127.0.0.1
subjectKeyIdentifier = hash

[node-1_distinguished_name]
CN = system:node:node-1
O  = system:nodes
C  = US
ST = Washington
L  = Seattle


# Kube Proxy Section
[kube-proxy]                                # kube-proxy
distinguished_name = kube-proxy_distinguished_name
prompt             = no
req_extensions     = kube-proxy_req_extensions

[kube-proxy_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Kube Proxy Certificate"
subjectAltName       = DNS:kube-proxy, IP:127.0.0.1
subjectKeyIdentifier = hash

[kube-proxy_distinguished_name]
CN = system:kube-proxy
O  = system:node-proxier                    # system:node-proxier ClusterRoleBinding 존재 , 서비스 네트워크 제어 가능
C  = US
ST = Washington
L  = Seattle


# Controller Manager
[kube-controller-manager]
distinguished_name = kube-controller-manager_distinguished_name
prompt             = no
req_extensions     = kube-controller-manager_req_extensions

[kube-controller-manager_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Kube Controller Manager Certificate"
subjectAltName       = DNS:kube-controller-manager, IP:127.0.0.1
subjectKeyIdentifier = hash

[kube-controller-manager_distinguished_name] # 클러스터 상태 관리, Node, ReplicaSet, SA 토큰 등 관리
CN = system:kube-controller-manager
O  = system:kube-controller-manager
C  = US
ST = Washington
L  = Seattle


# Scheduler
[kube-scheduler]
distinguished_name = kube-scheduler_distinguished_name
prompt             = no
req_extensions     = kube-scheduler_req_extensions

[kube-scheduler_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Kube Scheduler Certificate"
subjectAltName       = DNS:kube-scheduler, IP:127.0.0.1
subjectKeyIdentifier = hash

[kube-scheduler_distinguished_name]
CN = system:kube-scheduler
O  = system:kube-scheduler                  # Pod 스케줄링 전용 권한
C  = US
ST = Washington
L  = Seattle


# API Server
#
# The Kubernetes API server is automatically assigned the `kubernetes`
# internal dns name, which will be linked to the first IP address (`10.32.0.1`)
# from the address range (`10.32.0.0/24`) reserved for internal cluster
# services.

[kube-api-server]                           # API Server 인증서
distinguished_name = kube-api-server_distinguished_name
prompt             = no
req_extensions     = kube-api-server_req_extensions

[kube-api-server_req_extensions]
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth, serverAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client, server
nsComment            = "Kube API Server Certificate"
subjectAltName       = @kube-api-server_alt_names
subjectKeyIdentifier = hash

[kube-api-server_alt_names]                 # SAN (Subject Alternative Name) : 모든 내부/외부 접근 주소
IP.0  = 127.0.0.1
IP.1  = 10.32.0.1
DNS.0 = kubernetes
DNS.1 = kubernetes.default
DNS.2 = kubernetes.default.svc
DNS.3 = kubernetes.default.svc.cluster
DNS.4 = kubernetes.svc.cluster.local
DNS.5 = server.kubernetes.local
DNS.6 = api-server.kubernetes.local

[kube-api-server_distinguished_name]
CN = kubernetes
C  = US
ST = Washington
L  = Seattle


[default_req_extensions]                    # 공통 CSR 확장 : 대부분 클라이언트 인증서 -> kubelet / apiserver만 serverAuth 추가
basicConstraints     = CA:FALSE
extendedKeyUsage     = clientAuth
keyUsage             = critical, digitalSignature, keyEncipherment
nsCertType           = client
nsComment            = "Admin Client Certificate"
subjectKeyIdentifier = hash
```

## 각 컴포넌트별 개인키 , CSR, 인증서 생성

```bash
# Root CA 개인키
openssl genrsa -out ca.key 4096
openssl req -x509 -new -sha512 -noenc \
  -key ca.key -days 3653 \
  -config ca.conf \
  -out ca.crt
  
# Create Client and Server Certificates : admin
openssl genrsa -out admin.key 4096
openssl req -new -key admin.key -sha256 \
  -config ca.conf -section admin \
  -out admin.csr

openssl x509 -req -days 3653 -in admin.csr \
  -copy_extensions copyall \
  -sha256 -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out admin.crt  
  
# Create Client and Server Certificates: 나머지
certs=( "node-0" "node-1" "kube-proxy" "kube-scheduler" "kube-controller-manager" "kube-api-server" "service-accounts" )

# 확인 
echo ${certs[*]}

# 개인키 생성, csr 생성, 인증서 생성 
for i in ${certs[*]}; do
openssl genrsa -out "${i}.key" 4096

openssl req -new -key "${i}.key" -sha256 \
-config "ca.conf" -section ${i} \
-out "${i}.csr"

openssl x509 -req -days 3653 -in "${i}.csr" \
-copy_extensions copyall \
-sha256 -CA "ca.crt" \
-CAkey "ca.key" \
-CAcreateserial \
-out "${i}.crt"
done

```

생성된 개인키 , csr, 인증서 확인

```bash
ls -1 *.crt *.key *.csr

admin.crt
admin.csr
admin.key
ca.crt
ca.key
kube-api-server.crt
kube-api-server.csr
kube-api-server.key
kube-controller-manager.crt
kube-controller-manager.csr
kube-controller-manager.key
kube-proxy.crt
kube-proxy.csr
kube-proxy.key
kube-scheduler.crt
kube-scheduler.csr
kube-scheduler.key
node-0.crt
node-0.csr
node-0.key
node-1.crt
node-1.csr
node-1.key
service-accounts.crt
service-accounts.csr
service-accounts.key
```

인증서 / 개인키 , csr 정보 를 확인

```bash
# 각종 key 확인
openssl rsa -in ca.key -text -noout
openssl rsa -in admin.key -text -noout
openssl rsa -in node-0.key -text -noout
openssl rsa -in node-1.key -text -noout
openssl rsa -in kube-proxy.key -text -noout
openssl rsa -in kube-scheduler.key -text -noout
openssl rsa -in kube-controller-manager.key -text -noout
openssl rsa -in kube-api-server.key -text -noout
openssl rsa -in service-accounts.key -text -noout

#각종 csr 확인
# Root CA 인증서 생성하였으므로(Self-Signed certificate) 없음
openssl req -in admin.csr -text -noout
openssl req -in node-0.csr -text -noout
openssl req -in node-1.csr -text -noout
openssl req -in kube-proxy.csr -text -noout
openssl req -in kube-scheduler.csr -text -noout
openssl req -in kube-controller-manager.csr -text -noout
openssl req -in kube-api-server.csr -text -noout
openssl req -in service-accounts.csr -text -noout

# 각종 인증서 확인
openssl x509 -in ca.crt -text -noout
openssl x509 -in admin.crt -text -noout
openssl x509 -in node-0.crt -text -noout
openssl x509 -in node-1.crt -text -noout
openssl x509 -in kube-proxy.crt -text -noout
openssl x509 -in kube-scheduler.crt -text -noout
openssl x509 -in kube-controller-manager.crt -text -noout
openssl x509 -in kube-api-server.crt -text -noout
openssl x509 -in service-accounts.crt -text -noout
```

## node-0 , node-1에 클라이언트 및 서버 증명서 전달

```bash
for host in node-0 node-1; do
ssh root@${host} mkdir /var/lib/kubelet/

scp ca.crt root@${host}:/var/lib/kubelet/
scp ${host}.crt \
 root@${host}:/var/lib/kubelet/kubelet.crt

scp ${host}.key \
  root@${host}:/var/lib/kubelet/kubelet.key
done

# 확인
ssh node-0 ls -l /var/lib/kubelet 
ssh node-1 ls -l /var/lib/kubelet

# 서버머신에 적절한 개인키 및 증명서를 전달
scp \ 
  ca.key ca.crt \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  root@server:~  
  
# 확인
ssh server ls -l /root
  
```

## API Server와 통신을 위한 Client 인증 설정 파일 작성

- `kubelet`  k8s config 파일

```bash
# apiserver 파드 args 정보
kubectl describe pod -n kube-system kube-apiserver-myk8s-control-plane
    Command:
      kube-apiserver
      --authorization-mode=Node,RBAC  


# Generate a kubeconfig file for the node-0 and node-1 worker nodes

# config set-cluster
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=node-0.kubeconfig && ls -l node-0.kubeconfig && cat node-0.kubeconfig

kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=node-1.kubeconfig && ls -l node-1.kubeconfig && cat node-1.kubeconfig

# config set-credentials
kubectl config set-credentials system:node:node-0 \
  --client-certificate=node-0.crt \
  --client-key=node-0.key \
  --embed-certs=true \
  --kubeconfig=node-0.kubeconfig && cat node-0.kubeconfig

kubectl config set-credentials system:node:node-1 \
  --client-certificate=node-1.crt \
  --client-key=node-1.key \
  --embed-certs=true \
  --kubeconfig=node-1.kubeconfig && cat node-1.kubeconfig
  
# set-context : default 추가
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:node-0 \
  --kubeconfig=node-0.kubeconfig && cat node-0.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:node-1 \
  --kubeconfig=node-1.kubeconfig && cat node-1.kubeconfig


# use-context : current-context 에 default 추가
kubectl config use-context default \
  --kubeconfig=node-0.kubeconfig

kubectl config use-context default \
  --kubeconfig=node-1.kubeconfig


#
ls -l *.kubeconfig
-rw------- 1 root root 10157 Jan  3 14:55 node-0.kubeconfig
-rw------- 1 root root 10068 Jan  3 14:50 node-1.kubeconfig
```

- `kube-proxy` k8s config 파일

```bash
# Generate a kubeconfig file for the kube-proxy service
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.crt \
  --client-key=kube-proxy.key \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default \
  --kubeconfig=kube-proxy.kubeconfig

# 확인
cat kube-proxy.kubeconfig
```

- `kube-controller-manager` k8s config 파일

```bash
# Generate a kubeconfig file for the kube-controller-manager service
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.crt \
  --client-key=kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default \
  --kubeconfig=kube-controller-manager.kubeconfig

# 확인
cat kube-controller-manager.kubeconfig
```

- `kube-scheduler` k8s config 파일

```bash
# Generate a kubeconfig file for the kube-scheduler service
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.crt \
  --client-key=kube-scheduler.key \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default \
  --kubeconfig=kube-scheduler.kubeconfig

# 확인
cat kube-scheduler.kubeconfig
```

- `admin` k8s config 파일

```bash
# Generate a kubeconfig file for the admin user
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default \
  --kubeconfig=admin.kubeconfig

# 확인
cat admin.kubeconfig
```

## API Server와 통신을 위한 Client 인증 설정 파일을 서버 및 노드에 전달

- `kubelet` and `kube-proxy` 의 kubeconfig를 `node-0` 및 `node-1`에 복사

```bash
#
ls -l *.kubeconfig


# kubelet and kube-proxy kubeconfig 파일을 node-0 및 node-1에 복사
for host in node-0 node-1; do
  ssh root@${host} "mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig \
    root@${host}:/var/lib/kube-proxy/kubeconfig \

  scp ${host}.kubeconfig \
    root@${host}:/var/lib/kubelet/kubeconfig
done

# 확인
ssh node-0 ls -l /var/lib/*/kubeconfig
ssh node-1 ls -l /var/lib/*/kubeconfig

```


- `kube-controller-manager` 및 `kube-scheduler`의  kubeconfig 파일을 server에 복사

```bash
# kube-controller-manager and kube-scheduler kubeconfig 파일을 server에 복사
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  root@server:~/

# 확인
ssh server ls -l /root/*.kubeconfig
```


# 데이터 암호화 config 및 key생성

```bash
# The Encryption Key

# Generate an encryption key
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
echo $ENCRYPTION_KEY
JMnUP1PUUORZE9iadPdzYifnvPVIniSzOW6NUoMofVc=


# The Encryption Config File

# Create the encryption-config.yaml encryption config file
# (참고) 실제 etcd 값에 기록되는 헤더 : k8s:enc:aescbc:v1:key1:<ciphertext>
cat configs/encryption-config.yaml
kind: EncryptionConfiguration           # kube-apiserver가 etcd에 저장할 리소스를 어떻게 암호화할지 정의
apiVersion: apiserver.config.k8s.io/v1  # --encryption-provider-config 플래그로 참조
resources:
  - resources:
      - secrets                         # 암호화 대상 Kubernetes 리소스 : 여기서는 Secret 리소스만 암호화
    providers:                          # 지원 providers(위 부터 적용됨) : identity, aescbc, aesgcm, kms v2, secretbox
      - aescbc:                         # etcd에 저장될 Secret을 AES-CBC 방식으로 암호화
          keys:
            - name: key1                # 키 식별자 (etcd 데이터에 기록됨)
              secret: ${ENCRYPTION_KEY}
      - identity: {}                    # 암호화하지 않음 (Plaintext), 주로 하위 호환성 / 점진적 암호화 목적
# aescbc를 첫 번째에, identity를 두 번째에 배치하는 것은 "새로운 데이터는 무조건 암호화해서 저장하되, 이전에 평문으로 저장되었던 데이터도 문제없이 읽어 들이겠다"는 하위 호환성 전략을 의미
# 설명 출처 : https://yyeon2.medium.com/bootstrap-kubernetes-the-hard-way-48644e868550

envsubst < configs/encryption-config.yaml > encryption-config.yaml
cat encryption-config.yaml

# Copy the encryption-config.yaml encryption config file to each controller instance:
scp encryption-config.yaml root@server:~/
ssh server ls -l /root/encryption-config.yaml

```



# Server 노드에 etcd 서비스 기동


```bash
# Prerequisites

# hostname 변경 : controller -> server
# http 평문 통신!
# Each etcd member must have a unique name within an etcd cluster. 
# Set the etcd name to match the hostname of the current compute instance:
cat units/etcd.service | grep controller

ETCD_NAME=server
cat > units/etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --initial-advertise-peer-urls http://127.0.0.1:2380 \\
  --listen-peer-urls http://127.0.0.1:2380 \\
  --listen-client-urls http://127.0.0.1:2379 \\
  --advertise-client-urls http://127.0.0.1:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ${ETCD_NAME}=http://127.0.0.1:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
cat units/etcd.service | grep server

# Copy etcd binaries and systemd unit files to the server machine
scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```


```bash
ssh root@server
-------------------------------------------------------------------
# Bootstrapping an etcd Cluster

# Install the etcd Binaries
# Extract and install the etcd server and the etcdctl command line utility
pwd
mv etcd etcdctl /usr/local/bin/

# Configure the etcd Server
mkdir -p /etc/etcd /var/lib/etcd
chmod 700 /var/lib/etcd
cp ca.crt kube-api-server.key kube-api-server.crt /etc/etcd/

# Create the etcd.service systemd unit file:
mv etcd.service /etc/systemd/system/
tree /etc/systemd/system/

# Start the etcd Server
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd

# 확인
systemctl status etcd --no-pager
ss -tnlp | grep etcd
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T150247881Z.png)



```bash

# etcd 클러스터 멤버를 리스트화 하여 확인
etcdctl member list
etcdctl member list -w table
etcdctl endpoint status -w table

exit
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T150359780Z.png)

# k8s 컨트롤 플레인 부트스트래핑

## server 노드에 ‘api server, scheduler, kcm 서비스 기동

| 항목               | 네트워크 대역 or IP     |
| ---------------- | ----------------- |
| **clusterCIDR**  | **10.200.0.0/16** |
| → node-0 PodCIDR | 10.200.0.0/24     |
| → node-1 PodCIDR | 10.200.1.0/24     |
| **ServiceCIDR**  | **10.32.0.0/24**  |
| → api clusterIP  | 10.32.0.1         |

### kube-apiserver.service 기동

```bash
# Prerequisites

# kube-apiserver.service 수정 : service-cluster-ip-range 추가
# https://github.com/kelseyhightower/kubernetes-the-hard-way/issues/905
# service-cluster-ip 값은 ca.conf 에 설정한 [kube-api-server_alt_names] 항목의 Service IP 범위
cat ca.conf | grep '\[kube-api-server_alt_names' -A2
[kube-api-server_alt_names]
IP.0  = 127.0.0.1
IP.1  = 10.32.0.1

cat units/kube-apiserver.service
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


```bash
cat configs/kube-apiserver-to-kubelet.yaml ; echo
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


# kube-scheduler
cat units/kube-scheduler.service ; echo
cat configs/kube-scheduler.yaml ; echo


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

```
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
clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created
clusterrolebinding.rbac.authorization.k8s.io/system:kube-apiserver created

# 확인
kubectl get clusterroles system:kube-apiserver-to-kubelet --kubeconfig admin.kubeconfig
kubectl get clusterrolebindings system:kube-apiserver --kubeconfig admin.kubeconfig

---------------------------------------------------------------
```
# node-0
```bash
ssh root@node-0
# EPEL 저장소 활성화 bridge-utils 설치를 위해
sudo dnf -y install epel-release 

# 
dnf -y install socat conntrack ipset kmod psmisc bridge-utils

# Disable Swap : Verify if swap is disabled:
swapon --show

#
mkdir -p \
 /etc/cni/net.d \ 
 /opt/cni/bin \
 /var/lib/kubelet \ 
 /var/lib/kube-proxy \
 /var/lib/kubernetes \
 /var/run/kubernetes
```


갑자기 되었는데?
```bash
# node-1에서
sudo systemctl status firewalld
getenforce

# 만약 활성화되어 있으면
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo setenforce 0


# 재시작

sudo systemctl restart containerd
sudo systemctl restart kubelet
sudo systemctl restart kube-proxy

sleep 5

# 상태 확인
sudo systemctl status kubelet --no-pager | tail -10
```