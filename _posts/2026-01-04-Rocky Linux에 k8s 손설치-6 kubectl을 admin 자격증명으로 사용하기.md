

# jumpbox 노드에서 kubectl 을 admin 자격증명으로 사용을 위한 설정하기

- `vagrant ssh jumpbox`

```bash
curl -s -k --cacert ca.crt https://server.kubernetes.local:6443/version | jq


kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443

kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key

kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin

kubectl config use-context kubernetes-the-hard-way

# 위 명령어를 실행한 결과 kubectl 명령줄 도구에서 사용하는 기본 위치 ~/.kube/config에 kubectl 파일이 생성됩니다. 
# 이는 또한 구성을 지정하지 않고도 kubectl 명령어를 실행할 수 있음을 의미합니다.

kubectl version
kubectl get nodes -v=6
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T171026667Z.png)



```bash
cat /root/.kube/config
kubectl get nodes -owide
kubectl get pod -A
```


