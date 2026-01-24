---
layout: post
title: "[K8S Deploy Study by Gasida] -  k8s 컨트롤플레인 HA환경 구성 따라해보기 - kubeadm 으로 업그레이드"
categories:
  - kubernetes
tags:
  - k8s
  - kubeadm
---

# k8s upgrade by kubeadm

- `api-server`파드는 3대 모두가 Active로 동작
- `etcd`는 1대가 리더역할 하며 Write처리
- `kube-controller-manager`와 `kube-scheduler` 역시 1대가 리더역할

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260124T143141713Z.png)


## 컨트롤 플레인 3대인 환경에서 1대가 장애 발생시

- `admin-lb(HAProxy)`에 백엔드 대상 `k8s-ctr1`이 헬스 체크 실패하니 제외하고 나머지 대상으로 요청을 전달한다.
- 정상 노드 2대 중 1대의` etcd` 는 리더 역할이 되어, Write 처리함.
- 기존 `kube-controller-manger` 과 `kube-scheduler` 는 리더가 k8s-ctr1 이였는데, 장애 발생 중이여서 나머지 노드에서 리더 역할을 가져감.

결국은 1대가 장애가 발생하더라도 k8s api 호출 처리에는 문제가 없다.








# 



k8s-ctr1에서 인증서 갱신

