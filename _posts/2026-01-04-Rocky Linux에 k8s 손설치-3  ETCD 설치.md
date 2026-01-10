# 데이터 암호화 config 및 key생성

Kubernetes에서 Secret 리소스(민감 데이터)를 ETCD에 저장할 때 암호화하는 과정이다.

- key의 용도
	- ETCD에 저장되는 모든 Secret(비밀번호, API 토큰 등)을 암호화
	- ETCD의 데이터를 누군가 탈취해도 암호화되어 있어서 안전

```bash
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
echo $ENCRYPTION_KEY
JMnUP1PUUORZE9iadPdzYifnvPVIniSzOW6NUoMofVc=
```


```bash
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



# Server 노드(컨트롤플레인) 에 ETCD 설치

ETCD Systemd 서비스 파일 생성
```bash
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
  --initial-advertise-peer-urls http://127.0.0.1:2380 \\ # 다른 ETCD에 알릴 주소
  --listen-peer-urls http://127.0.0.1:2380 \\ # 다른 ETCD 통신용 포트
  --listen-client-urls http://127.0.0.1:2379 \\ # 클라이언트 통신 포트 (Kubernetes API)
  --advertise-client-urls http://127.0.0.1:2379 \\ # 클라이언트에 알릴 주소
  --initial-cluster-token etcd-cluster-0 \\ # 클러스터 생성 시 사용하는 토큰
  --initial-cluster ${ETCD_NAME}=http://127.0.0.1:2380 \\ # 클러스터 멤버 정보
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target # 시스템 부팅 시 자동 시작
EOF

cat units/etcd.service | grep server


scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```

-  포트 설명

|포트|용도|사용자|
|---|---|---|
|**2379**|클라이언트 통신|Kubernetes API 서버|
|**2380**|피어(Peer) 통신|다른 ETCD 서버 (단일 노드이므로 사용 안 함)|

- 디렉토리 용도

| 디렉토리             | 용도                   |
| ---------------- | -------------------- |
| `/etc/etcd/`     | ETCD 설정 및 TLS 인증서 저장 |
| `/var/lib/etcd/` | ETCD 데이터베이스 파일 저장    |


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
