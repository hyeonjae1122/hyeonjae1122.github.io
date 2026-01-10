
이번에는 TLS 인증서를 어떻게 생성하고 배포하는지를 설명한다. 

# Kubernetes의 보안 기반

Kubernetes 클러스터는 여러 컴포넌트들이 서로 통신하는데 이 통신이 안전해야 한다. kubectl 클라이언트가 API 서버와 통신할 때, kubelet이 API 서버와 통신할 때, 그리고 다양한 컨트롤 플레인 컴포넌트들이 서로 통신할 때 모두 암호화되고 인증된 연결이 필요하다. 

# CA 설정 및 TLS 인증서 생성

Kubernetes 클러스터를 운영하려면 총 9개의 인증서가 필요하다. 먼저 Root CA (인증 기관)를 만들어야 한다. Root CA는 다른 모든 인증서에 서명하는 신뢰의 근원으로 작동한다.

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

인증서를 만들 때 OpenSSL이라는 도구를 사용하는데 이 도구에게 어떤 인증서를 만들어야 하는지 알려주기 위해 ca.conf라는 설정 파일을 작성한다. 이 파일은 각 인증서의 특성을 정의하고있다.

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
scp ca.key ca.crt kube-api-server.key kube-api-server.crt service-accounts.key service-accounts.crt root@server:~

# 확인
ssh server ls -l /root
  
```

## API Server와 통신을 위한 Client 인증 설정 파일 작성

인증서를 만들었다면, 각 컴포넌트가 이 인증서를 사용해서 API 서버에 접근할 수 있도록 **kubeconfig 파일**을 만들어야 한다. Kubeconfig는 어떤 클러스터에 접근할 것인지, 어떤 자격증명을 사용할 것인지, 그리고 어떤 컨텍스트를 사용할 것인지를 정의한다.


### Kubeconfig 파일을 만드는 과정

첫 번째로 클러스터 정보를 설정한. API 서버의 주소(예: https://server.kubernetes.local:6443), CA 인증서의 경로, 그리고 인증서를 파일 내에 포함시킬지(embed-certs=true)를 지정한다.

두번째로 사용자 자격증명을 설정한다. 여기서는 인증서와 개인키의 경로를 지정하며. kubelet의 경우 "system:node:node-0" 같은 형식의 사용자 이름을 사용한다.

세 번째로 컨텍스트를 설정하고 기본 컨텍스트로 지정합니다. 컨텍스트는 클러스터, 사용자, 네임스페이스를 조합한 것이다. 

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

생성한 인증서와 kubeconfig 파일들을 적절한 위치에 배포환다.

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



# 