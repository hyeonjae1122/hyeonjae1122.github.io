---
layout: post
title:  "글 쓰는 방법"
---

# 실리움 시작하기전에 TL;DR

#### 실리움 아키텍처
![img.png](../assets/1week-arch/img.png)

1️⃣ Cilium Agent runs on each node(DaemonSet)
실리움 에이전트는 각 노드에서 데몬셋을 사용하여 실행된다.



2️⃣ Cilium agent listens to k8s events and API calls to understand the cluster's networking and security requirements.
클러스터의 네트워킹 및 보안 요구사항을 파악하기 위해 kubernetes events와 API call을 모니터링한다.



3️⃣ Cilium agent manages eBPF programs
실리움 에이전트는 eBPF 프로그램을 관리, 로딩을 담당하여 적절한 인터페이스에 연결되도록한다.


4️⃣ Cilium Operator manages cluster-wide tasks.
실리움 오퍼레이터는 클러스터 전반의 작업을 관리한다.



5️⃣ Envoy proxy handles all L7 traffic (eBPF can only work with L3/L4)
엔보이 프락시는 각 노드당 하나씩 배치 되며 모든 L7트래픽을 처리한다.(eBPF는 모든 L3, L4를 처리한다)



6️⃣ Envoy proxy can be deployed within the Cilium agent pods or as a standalone pod.
엔보이 프락시는 실리움 에이전트 파드 내 또는 독립 파드로 배포될 수 있다.



7️⃣ Cilium uses kubernetes CRDs as a data store to propagate state between agents.
또한 Cilium은 k8s CRDs를 데이터 스토어로 사용하여 에이전트 간의 상태를 전파한다.



8️⃣ The Hubble server runs on each node and captures traffic flows
허블 서버는 각 노드에서 실행되며 트래픽 흐름을 캡처한다.



9️⃣ The Hubble Relay componets aggregates flow data from multiple Hubble servers to provide cluster-wide visibility.
허블 릴레이 구성 요소는 여러 허블 서버에서 데이터 플로우를 집계하여 클러스터 전체의 가시성을 제공한다.



🔟 Hubble CLI is command line tool used to query the Hubble Relay for flow data.
허블 CLI는 허블 릴레이에서 데이터흐름을 쿼리하는데 사용되는 CLI이다.



1️⃣1️⃣ Hubble UI utilizes relay-based visibility to provide a graphical service dependency and connectivity map
Hubble UI는 릴레이 기반 가시성을 사용하여 그래픽 서비스 의존성 및 연결맵을 제공한다.



1️⃣2️⃣ Sidecar-based service mesh requires a sidecar proxy for each application
사이드카 기반 서비스메시는 각 어플레케이션 마다 사이드카 프록시를 필요로한다.



1️⃣3️⃣ Sidecar model leads to higher complexity, higher resource usage, higher latency, longer start times.
사이드카 모델은 높은 복잡성, 높은 리소스 사용량, 높은 지연, 긴 시작 시간을 초래한다.



1️⃣4️⃣ Cilium uses a sidecarless model where eBPF programs handle all L3/L4 logic, and an Envoy Proxy (one per node) handles L7.
실리움은 사이드카리스 모델을 사용하는데 eBPF가  L3/L4 로직을 핸들링하며 각 노드당 배치된 엔보이 프락시가 L7를 담당한다.



1️⃣5️⃣ Sidecarless model requires less resources, less complexity, and lower latency.
사이드카리스 모델은 보다 적은 리소스 사용량, 적은 복잡도, 낮은 지연을 요구한다.


# 서비스 메시를 구현하는 두가지 모델 : Sidecar vs Sidecarless(Cilium의 접근방식)
서비스 메시를 구현하는 두가지 모델 : Sidecar vs Sidecarless ( Cilium의 접근 방식)
1. 전통적인 사이드카 (Sidecar) 모델
- 아키텍처: Envoy와 같은 전용 프록시가 모든 애플리케이션 파드 내부에 "사이드카" 컨테이너로 배포되며 애플리케이션을 오가는 모든 트래픽은 이 프록시를 통과
- 장점: 특정 프로그래밍 언어에 구애받지 않으며 애플리케이션 코드 변경이 필요 없음
- 단점:
  - 높은 리소스 사용량: 모든 파드에 추가 프록시 컨테이너가 필요하여 CPU 및 메모리 소비가 증가 
  - 높은 복잡성: 많은 프록시를 수동으로 구성하고 관리해야함
  - 지연 시간 증가: 모든 요청에 대해 추가적인 네트워크 홉이 발생
  - 느린 시작 시간: 애플리케이션은 사이드카 프록시가 시작될 때까지 기다려야 함


2. Cilium의 사이드카리스 (Sidecar-less) 모델
아키텍처: Cilium은 eBPF와 노드별 프록시를 활용하여 더 효율적인 모델을 제공

- eBPF: 모든 레이어 3 및 레이어 4 네트워킹과 보안을 커널에서 직접 처리하여 해당 트래픽에 대한 프록시 필요성을 없앰.
- Envoy 프록시: 고급 레이어 7 트래픽 관리(예: HTTP 인식 정책)를 위해 Cilium은 파드별이 아닌 노드별로 실행되는 단일 Envoy 프록시를 사용한다. eBPF 프로그램은 필요한 L7 트래픽만 이 공유 프록시로 지능적으로 리디렉션함
- 장점:
  - 적은 리소스: 프록시 수를 대폭 줄여 CPU와 메모리를 절약 
  - 낮은 복잡성: 관리할 부분이 적어 관리가 단순해짐
  - 더 나은 성능 및 낮은 지연 시간: L3/L4 트래픽에 대한 추가적인 네트워크 홉을 피함
