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
