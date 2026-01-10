# k8s 워커노드 부트스트래핑

## kubelet-config 파일 작성 및 워커노드에 전달

- 설정 파일 확인
```bash
cat configs/10-bridge.conf | jq
cat configs/kubelet-config.yaml | yq 
```

- 노드별 설정 파일 생성 및 전송

```bash
for HOST in node-0 node-1; do
  SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" \
    configs/10-bridge.conf > 10-bridge.conf

  sed "s|SUBNET|$SUBNET|g" \
    configs/kubelet-config.yaml > kubelet-config.yaml

  scp 10-bridge.conf kubelet-config.yaml \
  root@${HOST}:~/
done

# 확인
ssh node-0 ls -l /root
ssh node-1 ls -l /root

```


## 바이너리 및 설정 파일 전송

- 전송 할 파일 목록

```bash
downloads/worker/*           # kubelet, kube-proxy, crictl, runc 등 바이너리
downloads/client/kubectl     # kubectl 명령어 도구
downloads/cni-plugins/*      # flannel, vxlan 등 CNI 플러그인
configs/99-loopback.conf     # 루프백 네트워크 설정
configs/containerd-config.toml # 컨테이너 런타임 설정
configs/kube-proxy-config.yaml # 프록시 설정
units/containerd.service     # systemd 서비스 파일
units/kubelet.service        # systemd 서비스 파일
units/kube-proxy.service     # systemd 서비스 파일
```


- 모든 노드에 병렬 전송

```bash
for HOST in node-0 node-1; do
  scp \
    downloads/worker/* \
    downloads/client/kubectl \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@${HOST}:~/
done

for HOST in node-0 node-1; do
  scp \
    downloads/cni-plugins/* \
    root@${HOST}:~/cni-plugins/
done

# 확인
ssh node-0 ls -l /root
ssh node-1 ls -l /root
ssh node-0 ls -l /root/cni-plugins
ssh node-1 ls -l /root/cni-plugins
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T165500061Z.png)


## 워커노드 프로비저닝

- node-0 접속 후 실행

```bash
ssh root@node-0
# EPEL 저장소 활성화
sudo dnf -y install epel-release 
dnf -y install socat conntrack ipset kmod psmisc bridge-utils

# Disable Swap : Verify if swap is disabled:
swapon --show

# 필요한 디렉토리 생성
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes

# 바이너리 설치
mv crictl kube-proxy kubelet runc /usr/local/bin/
mv containerd containerd-shim-runc-v2 containerd-stress /bin/
mv cni-plugins/* /opt/cni/bin/


# CNI 네트워크 설정
mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
cat /etc/cni/net.d/10-bridge.conf 

# br-netfilter 모듈 활성화
# Pod 간 통신이 iptables 방화벽 규칙을 우회하지 않도록 강제
lsmod | grep netfilter
modprobe br-netfilter
echo "br-netfilter" >> /etc/modules-load.d/modules.conf
lsmod | grep netfilter

# iptables가 브릿지 트래픽을 처리하도록 설정
echo "net.bridge.bridge-nf-call-iptables = 1"  >> /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.d/kubernetes.conf
sysctl -p /etc/sysctl.d/kubernetes.conf


# containerd 설정
mkdir -p /etc/containerd/
mv containerd-config.toml /etc/containerd/config.toml
mv containerd.service /etc/systemd/system/
cat /etc/containerd/config.toml ; echo

```


- containerd-config.toml 주요 설정
```bash
cat /etc/containerd/config.toml ; echo
version = 2
[plugins."io.containerd.grpc.v1.cri"]               # CRI 플러그인 활성화 : kubelet은 이 플러그인을 통해 containerd와 통신
  [plugins."io.containerd.grpc.v1.cri".containerd]  # containerd 기본 런타임 설정
    snapshotter = "overlayfs"                       # 컨테이너 파일시스템 레이어 관리 방식 : Linux표준/성능최적
    default_runtime_name = "runc"                   # 기본 OCI 런타임 : 파드가 별도 지정 없을 경우 runc 사용
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]  # runc 런타임 상세 설정
    runtime_type = "io.containerd.runc.v2"                        # containerd 최신 runc shim
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]  # runc 옵션
    SystemdCgroup = true                                                  # containerd가 cgroup을 systemd로 관리 
[plugins."io.containerd.grpc.v1.cri".cni]           # CNI 설정
  bin_dir = "/opt/cni/bin"                          # CNI 플러그인 바이너리 위치
  conf_dir = "/etc/cni/net.d"                       # CNI 네트워크 설정 파일 위치
```



``` bash
# kubelet ↔ containerd 연결 Flow
kubelet
  ↓ CRI (gRPC)
containerd.sock(unix:///var/run/containerd/containerd.sock)
  ↓
containerd CRI plugin
  ↓
runc
  ↓
Linux namespaces / cgroups


# kubelet 설정
mv kubelet-config.yaml /var/lib/kubelet/
mv kubelet.service /etc/systemd/system/


# Configure the Kubernetes Proxy
mv kube-proxy-config.yaml /var/lib/kube-proxy/
mv kube-proxy.service /etc/systemd/system/


# Start the Worker Services
systemctl daemon-reload
systemctl enable containerd kubelet kube-proxy
systemctl start containerd kubelet kube-proxy


# 확인
systemctl status kubelet --no-pager
systemctl status containerd --no-pager
systemctl status kube-proxy --no-pager

exit
-----------------------------------------------------------

# jumpbox 에서 server 접속하여 kubectl node 정보 확인
ssh server "kubectl get nodes node-0 -o yaml --kubeconfig admin.kubeconfig" | yq
ssh server "kubectl get nodes -owide --kubeconfig admin.kubeconfig"
ssh server "kubectl get pod -A --kubeconfig admin.kubeconfig"
```


Rocky Linux판 `v1.34.2` 의 워커노드 접속 성공

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T165958538Z.png)

node-1도 동일한 과정을 반복

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260110T170450633Z.png)
