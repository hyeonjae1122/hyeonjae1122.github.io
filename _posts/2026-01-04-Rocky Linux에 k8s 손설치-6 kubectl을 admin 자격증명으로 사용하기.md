

# jumpbox 노드에서 kubectl 을 admin 자격증명으로 사용을 위한 설정하기

- `vagrant ssh jumpbox` 접속

- 클러스터 접근 가능 확인 (CA 인증서로 API 서버에 접근해서 정상 작동 확인)
```bash
curl -s -k --cacert ca.crt https://server.kubernetes.local:6443/version | jq
```


## kubectl 설정 파일(~/.kube/config) 생성

- 클러스터 정보 등록
```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443
```

- 사용자(admin) 인증 정보 등록

```bash
kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key
```

- Context(클러스터 + 사용자 조합) 생성

```bash
kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin
```

- 기본 Context 설정 (`kubectl` 명령어 실행 시 자동으로 이 context 사용)

```bash
kubectl config use-context kubernetes-the-hard-way
```


```bash
kubectl version
kubectl get nodes -v=6
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T171026667Z.png)


- 설정 확인
```bash
cat /root/.kube/config
kubectl get nodes -owide
kubectl get pod -A
```


