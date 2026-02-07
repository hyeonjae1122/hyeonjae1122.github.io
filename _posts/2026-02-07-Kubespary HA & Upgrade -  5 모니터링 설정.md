---
layout: post
---

>목표
>모니터링을 위해 ETCD 메트릭을 수집하도록 설정

# NFS subdir external provisioner 설치

-  NFS subdir external provisioner 설치 : admin-lb 에 NFS Server(/srv/nfs/share) 설정 되어 있음

```bash
kubectl create ns nfs-provisioner
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner -n nfs-provisioner \
    --set nfs.server=192.168.10.10 \
    --set nfs.path=/srv/nfs/share \
    --set storageClass.defaultClass=true
```


스토리지 클래스 확인

```bash
kubectl get sc

# 파드 확인
kubectl get pod -n nfs-provisioner -owide
```



# kube-prometheus-stack 설치, 대시보드 추가

- helm 추가

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```bash
# 파라미터 파일 생성
cat <<EOT > monitor-values.yaml
prometheus:
  prometheusSpec:
    scrapeInterval: "20s"
    evaluationInterval: "20s"
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
    additionalScrapeConfigs:
      - job_name: 'haproxy-metrics'
        static_configs:
          - targets:
              - '192.168.10.10:8405'
    externalLabels:
      cluster: "myk8s-cluster"
  service:
    type: NodePort
    nodePort: 30001

grafana:
  defaultDashboardsTimezone: Asia/Seoul
  adminPassword: prom-operator
  service:
    type: NodePort
    nodePort: 30002

alertmanager:
  enabled: false
defaultRules:
  create: false
kubeProxy:
  enabled: false
prometheus-windows-exporter:
  prometheus:
    monitor:
      enabled: false
EOT
```

-  배포

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 80.13.3 \
-f monitor-values.yaml --create-namespace --namespace monitoring
```

- 확인

```bash
helm list -n monitoring
kubectl get pod,svc,ingress,pvc -n monitoring
kubectl get prometheus,servicemonitors,alertmanagers -n monitoring
kubectl get crd | grep monitoring
```

-  각각 웹 접속 실행 : NodePort 접속

```bash
open http://192.168.10.14:30001 # prometheus
open http://192.168.10.14:30002 # grafana : 접속 계정 admin / prom-operator

# 프로메테우스 버전 확인
kubectl exec -it sts/prometheus-kube-prometheus-stack-prometheus -n monitoring -c prometheus -- prometheus --version

# 그라파나 버전 확인
kubectl exec -it -n monitoring deploy/kube-prometheus-stack-grafana -- grafana --version

```


- Grafana Dashboard
    - `15661`, `12693` 
        - [https://github.com/dotdc/grafana-dashboards-kubernetes](https://github.com/dotdc/grafana-dashboards-kubernetes)

```bash
curl -o 12693_rev12.json https://grafana.com/api/dashboards/12693/revisions/12/download
curl -o 15661_rev2.json https://grafana.com/api/dashboards/15661/revisions/2/download
curl -o k8s-system-api-server.json https://raw.githubusercontent.com/dotdc/grafana-dashboards-kubernetes/refs/heads/master/dashboards/k8s-system-api-server.json

# sed 명령어로 uid 일괄 변경 : 기본 데이터소스의 uid 'prometheus' 사용
sed -i -e 's/${DS_PROMETHEUS}/prometheus/g' 12693_rev12.json
sed -i -e 's/${DS__VICTORIAMETRICS-PROD-ALL}/prometheus/g' 15661_rev2.json
sed -i -e 's/${DS_PROMETHEUS}/prometheus/g' k8s-system-api-server.json

# my-dashboard 컨피그맵 생성 : Grafana 포드 내의 사이드카 컨테이너가 grafana_dashboard="1" 라벨 탐지!
kubectl create configmap my-dashboard --from-file=12693_rev12.json --from-file=15661_rev2.json --from-file=k8s-system-api-server.json -n monitoring
kubectl label configmap my-dashboard grafana_dashboard="1" -n monitoring

# 대시보드 경로에 추가 확인
kubectl exec -it -n monitoring deploy/kube-prometheus-stack-grafana -- ls -l /tmp/dashboards
```


# ETCD 메트릭 수집 될 수 있게 설정

```bash
ssh k8s-node1 ss -tnlp | grep etcd
ssh k8s-node1 ps -ef | grep etcd
cat roles/etcd/templates/etcd.env.j2 | grep -i metric
```


```
cat << EOF >> inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
etcd_metrics: true
etcd_listen_metrics_urls: "http://0.0.0.0:2381"
EOF
```


```
tail -n 5 inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

- 모니터링
    - `k8s-node1`

```bash
watch -d "etcdctl.sh member list -w table"
```

- `admin-lb`

```bash
while true; do echo ">> k8s-node1 <<"; ssh k8s-node1 etcdctl.sh endpoint status -w table; echo; echo ">> k8s-node2 <<"; ssh k8s-node2 etcdctl.sh endpoint status -w table; echo ">> k8s-node3 <<"; ssh k8s-node3 etcdctl.sh endpoint status -w table; sleep 1; done
```

- etcd 재시작

```bash
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml --tags "etcd" --list-tasks
ansible-playbook -i inventory/mycluster/inventory.ini -v cluster.yml --tags "etcd" --limit etcd -e kube_version="1.32.9"
```


- 확인

```bash
ssh k8s-node1 etcdctl.sh member list -w table
for i in {1..3}; do echo ">> k8s-node$i <<"; ssh k8s-node$i etcdctl.sh endpoint status -w table; echo; done

```


- 백업 확인

```bash
for i in {1..3}; do echo ">> k8s-node$i <<"; ssh k8s-node$i tree /var/backups; echo; done
```


```
ssh k8s-node1 ss -tnlp | grep etcd

curl -s http://192.168.10.11:2381/metrics
curl -s http://192.168.10.12:2381/metrics
curl -s http://192.168.10.13:2381/metrics
```


- 스크래핑 설정 추가

```bash
cat <<EOF > monitor-add-values.yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'etcd'
        metrics_path: /metrics 
        static_configs:
          - targets:
              - '192.168.10.11:2381'
              - '192.168.10.12:2381'
              - '192.168.10.13:2381'
EOF
```


- 헬름 업그레이드로 적용

```bash
helm get values -n monitoring kube-prometheus-stack
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 80.13.3 \
--reuse-values -f monitor-add-values.yaml --namespace monitoring
```


- 확인

```bash
helm get values -n monitoring kube-prometheus-stack
```

-  (옵션) 불필요 servicemonitor etcd 제거 : 반영에 다소 시간 소요

```bash
kubectl get servicemonitors.monitoring.coreos.com -n monitoring kube-prometheus-stack-kube-etcd -o yaml
kubectl delete servicemonitors.monitoring.coreos.com -n monitoring kube-prometheus-stack-kube-etcd
```