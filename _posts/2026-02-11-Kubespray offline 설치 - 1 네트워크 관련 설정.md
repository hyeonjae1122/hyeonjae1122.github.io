---
layout: post
tags:
  - Ansible
  - k8s
  - kubernetes
  - kubespray
title: "[K8S Deploy Study by Gasida] - Kubespray offline 설치 - 1 네트워크 관련 설정"
---

# 네트워크 게이트웨이(k8s-node의 외부 통신 끊고 내부망으로 설정)

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


# NTP 서버  (admin은 한국 공용 NPT서버로, k8s-node는 admin을 NTP서버로 동기화)

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

# DNS 서버 (NetworkManager에서 DNS 관리 끄기)

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