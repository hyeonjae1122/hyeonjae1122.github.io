---
layout: post
tags:
  - Ansible
  - k8s
  - kubernetes
  - kubespray
title: "[K8S Deploy Study by Gasida] - Kubespray offline 설치 - 1 네트워크 관련 설정"
---
# 개요

오프라인(폐쇄망) 환경의 쿠버네티스 클러스터를 구축하기 위해, `k8s-node`들은 외부 인터넷과 직접 통신할 수 없도록 격리해야 한다. 대신 패키지 다운로드나 외부 통신이 임시로 필요할 때, 인터넷과 연결된 `admin` 노드가 일종의 **라우터(NAT 게이트웨이)** 역할을 수행하도록 구성한다.

- `net.ipv4.ip_forward = 1`
    - 리눅스 커널이 자신의 IP가 아닌 다른 목적지로 가는 패키지를 폐기하지 않고 전달(Forwarding)하도록 허용
- `MASQUERADE`: 내부망(`enp0s9`)에서 출발한 패킷이 외부망(`enp0s8`)으로 나갈 때, 출발지 IP를 `admin` 노드의 외부 IP로 변조하여 통신이 가능하게 해줍니다.


# NTP 서버 동기화와 쿠버네티스 HA의 관계

- **분산 시스템에서 시간 동기화(NTP)가 필수적인 이유** 
    - 단순한 서버 시간 맞춤을 넘어, 쿠버네티스에서 시간 동기화는 클러스터의 안정성과 직결된다. 특히 쿠버네티스의 상태 데이터를 저장하는 **etcd**는 노드 간의 시간이 미세하게라도 어긋나면 리더 선출(Leader Election) 로직에 문제가 생겨 잦은 재시작이 발생하거나 고가용성(HA) 클러스터 전체에 심각한 장애를 유발할 수 있다.
    - 또한 추후 Keycloak과 같은 인증 시스템을 클러스터에 올리고 연동할 때, 서버 간 시간이 맞지 않으면 토큰(JWT) 발행 및 만료 시간에 오차가 생겨 인증 장애로 직결됩니다. 따라서 `k8s-node`들이 `admin` 노드를 바라보도록 철저히 동기화해야 한다.

# DNS 설정과 NetworkManager 제어

 - NetworkManager DNS 관리를 비활성화(`dns=none`)하는 이유
     - Rocky Linux를 비롯한 RHEL 기반 OS에서는 기본적으로 `NetworkManager`가 네트워크 인터페이스가 재시작되거나 시스템이 부팅될 때마다 `/etc/resolv.conf` 파일을 자동으로 덮어쓴다. 만약 이를 방치하면 우리가 기껏 설정해 둔 내부 DNS(`192.168.10.10`)가 초기화되어 버린다. 쿠버네티스의 내부 DNS인 **CoreDNS**는 기본적으로 호스트 노드의 `/etc/resolv.conf`를 참조하여 외부 도메인을 질의한다. 이 파일이 임의로 변경되는 것을 막는 것은 향후 발생할 수 있는 CoreDNS의 이름 해석 지연(Lag)이나 통신 실패 같은 장애를 예방하는 핵심 사전 작업이다.

# (실습) 네트워크 게이트웨이(k8s-node의 외부 통신 끊고 내부망으로 설정)

## admin

- 라우팅 설정

```bash
sysctl -w net.ipv4.ip_forward=1  # sysctl net.ipv4.ip_forward
cat <<EOF | tee /etc/sysctl.d/99-ipforward.conf
net.ipv4.ip_forward = 1
EOF
sysctl --system
```

- NAT 설정

```
iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE
iptables -t nat -S
iptables -t nat -L -n -v
```

## k8s-node

### enp0s8(외부망) -> enp0s9(내부망)


- `enp0s8` 확인 명령어

```bash
cat /etc/NetworkManager/system-connections/enp0s8.nmconnection
```

- `enp0s8` 연결 내리기

```bash
nmcli connection down enp0s8
nmcli connection modify enp0s8 connection.autoconnect no #재부팅 이후에도 자동연결내리기
```


- 외부 통신을 위해 `enp0s9` 에 디폴트 라우팅 추가 : 우선순위 200설정

```bash
nmcli connection modify enp0s9 +ipv4.routes "0.0.0.0/0 192.168.10.10 200"
```

- 설정적용

```bash
nmcli connection up enp0s9
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T064323086Z.png)

- 각종 확인 명령어

```bash
# 외부 통신 확인
ping -w 1 -W 1 8.8.8.8
curl www.google.com
curl: (6) Could not resolve host: www.google.com

# DNS Nameserver 정보 확인
cat /etc/resolv.conf
cat << EOF > /etc/resolv.conf
nameserver 168.126.63.1
nameserver 8.8.8.8
EOF

```

```bash
curl www.google.com


# (참고) 디폴트 라우팅 제거 시
nmcli connection modify enp0s9 -ipv4.routes "0.0.0.0/0 192.168.10.10 200"
nmcli connection up enp0s9
ip route
```


# (실습) NTP 서버  (admin은 한국 공용 NPT서버로, k8s-node는 admin을 NTP서버로 동기화)

## admin

- 현재 ntp 서버와 타임 동기화 설정 및 상태 확인
```bash
systemctl status chronyd.service --no-pager
grep "^[^#]" /etc/chrony.conf
```


-  chrony가 어떤 NTP 서버들을 알고 있고, 그중 어떤 서버를 기준으로 시간을 맞추는지를 보여준다.

```bash
chronyc sources -v
dig +short 2.rocky.pool.ntp.org
```

 - chrony 설정

```
cp /etc/chrony.conf /etc/chrony.bak
```

- 외부 한국 공용 NTP 서버 설정
```bash
cat << EOF > /etc/chrony.conf

server pool.ntp.org iburst
server kr.pool.ntp.org iburst

# 내부망(192.168.10.0/24)에서 이 서버에 접속하여 시간 동기화 허용
allow 192.168.10.0/24

# 외부망이 끊겼을 때도 로컬 시계를 기준으로 내부망에 시간 제공 (선택 사항)
local stratum 10

# 로그
logdir /var/log/chrony
EOF
```

- 재시작 및 상태확인
```bash
systemctl restart chronyd.service
systemctl status chronyd.service --no-pager
```

- 상태 확인

```bash
timedatectl status
chronyc sources -v
```

- `k8s-node` 시간 동기화 후 자신의 NTP Server를 사용하는 클라이언트 확인

```bash
chronyc clients
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T065523961Z.png)

## k8s-node

- 상태 확인

```bash
timedatectl status
chronyc sources -v
```


```bash
cp /etc/chrony.conf /etc/chrony.bak
cat << EOF > /etc/chrony.conf
server 192.168.10.10 iburst
logdir /var/log/chrony
EOF
```

- 상태확인
```bash
systemctl restart chronyd.service
systemctl status chronyd.service --no-pager
```


- 상태 확인

```bash
timedatectl status
chronyc sources -v
```

#  (실습) DNS 서버 (NetworkManager에서 DNS 관리 끄기)

## admin 

-  bind 설치

```bash

dnf install -y bind bind-utils
```


-  `named.conf` 설정

```bash
cp /etc/named.conf /etc/named.bak
cat <<EOF > /etc/named.conf
options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { 127.0.0.1; 192.168.10.0/24; };
        allow-recursion { 127.0.0.1; 192.168.10.0/24; };

        forwarders {
                168.126.63.1;
                8.8.8.8;
        };

        recursion yes;

        dnssec-validation auto;  # https://sirzzang.github.io/kubernetes/Kubernetes-Kubespray-08-01-06/#troubleshooting-dnssec-%EA%B2%80%EC%A6%9D-%EC%8B%A4%ED%8C%A8

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
EOF
```

- 문법 오류 확인 (아무 메시지 없으면 정상)

```
named-checkconf /etc/named.conf
```

- 서비스 활성화 및 시작

```
systemctl enable --now named
```


- DMZ 서버 자체 DNS 설정 (자기 자신 사용)

```bash
cat /etc/resolv.conf
echo "nameserver 192.168.10.10" > /etc/resolv.conf
```

- 확인

```bash
dig +short google.com @192.168.10.10
dig +short google.com
```

-  NetworkManager에서 DNS 관리 끄기 : 미적용 시, admin 서버 재부팅 시 NetworkManager 가 초기 설정값 덮어쓰움.

```bash
cat /etc/NetworkManager/conf.d/99-dns-none.conf
cat << EOF > /etc/NetworkManager/conf.d/99-dns-none.conf
[main]
dns=none
EOF
```


```
systemctl restart NetworkManager
```



## k8s-node

```bash
cat /etc/NetworkManager/conf.d/99-dns-none.conf
```

```bash
cat << EOF > /etc/NetworkManager/conf.d/99-dns-none.conf
[main]
dns=none
EOF
```

```bash
systemctl restart NetworkManager
```


# 네트워크 설정 최종 점검 체크리스트

  다음 단계인 K8s 설치로 넘어가기 전, 아래 항목들이 정상적으로 동작하는지 확인한다.
-  `k8s-node`에서 `ping 8.8.8.8` (IP 통신)은 정상 작동하는가?
-   `k8s-node`에서 `curl google.com` 시, `admin`에 구축한 Bind DNS를 통해 IP를 정상적으로 받아오는가?
- `k8s-node`에서 `chronyc sources -v` 명령어 입력 시, `admin` 노드의 IP(`192.168.10.10`) 앞에 `^*` 기호(정상 동기화 중임을 의미)가 표시되는가?