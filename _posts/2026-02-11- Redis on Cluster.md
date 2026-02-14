---
layout: post
tags:
  - redis
  - k8s
title: 레디스 다중화 구성하기
aliases:
---
참고글
https://tech.kakaopay.com/post/kakaopaysec-redis-on-kubernetes/


# 


## Hostnetwork

- HostNetwork는 Pod이 호스트의 네트워크 인터페이스를 공유할 수 있도록 허용하는 Kubernetes의 네트워크 설정

- 장점
    - 네트워크 성능 향상 : 레디스 클러스터에서는 파드간 통신을 위한 네트워크 성능이 매우 중요하다. Hostnetwork를 사용하면 파드간 통신시 Pod IP 주소를 직접 사용하기 때문에 네트워크 오버헤드를 줄이고 레디스 클러스터 간의 통신 속도를 크게 향상할 수 있다.


## Pod간 직접 통신을 통한 단순화

- 레디스 클러스터에는 데이터가 여러 노드에 Sharding 되어 분산 저장 되어 있다. 클라이언트에서는 특정 데이터를 요청할 때 요청된 데이터가 저장된 클러스터 구성 정보(Topology)를 리디렉션 응답으로 클라이언트에게 리턴하여 클라이언트가 원하는 데이터가 저장된 노드로 다시 연결할 수 있도록 처리한다.


## 내/외부 네트워크 엔드포인트

-
```bash
kubectl get svc -n redis-cluster
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260211T094447585Z.png)


- Headless Service는 일반서비스와 같이 Cluster IP를 갖지 않고 대신 DNS를 갖고 있다. Headless Service의 DNS를 Lookup 하게 되면 Redis Cluster의 모든 Pod IP가 조회된다. 


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260211T094643386Z.png)

