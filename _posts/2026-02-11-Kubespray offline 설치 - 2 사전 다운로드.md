---
layout: post
tags:
  - Ansible
  - k8s
  - kubernetes
  - kubespray
title: "[K8S Deploy Study by Gasida] - Kubespray offline 설치 2 사전 다운로드"
---
# 리눅스 / 파이썬 패키지 및 이미지 저장소 구성


## 미러 YUM/DNF 리포지토리
### admin

```bash
dnf install -y dnf-plugins-core createrepo nginx
```


```
mkdir -p /data/repos/rocky/10
cd /data/repos/rocky/10
```


-  BaseOS, AppStream, Extras 동기화

```bash
dnf repolist
```

```bash
dnf reposync --repoid=baseos    --download-metadata -p /data/repos/rocky/10
du -sh /data/repos/rocky/10/baseos/

dnf reposync --repoid=appstream --download-metadata -p /data/repos/rocky/10

dnf reposync --repoid=extras    --download-metadata -p /data/repos/rocky/10
```


(Optional) 테스트용

```bash
# 내부 배포용 웹 서버 설정 (nginx)
cat <<EOF > /etc/nginx/conf.d/repos.conf
server {
    listen 80;
    server_name repo-server;

    location /rocky/10/ {
        autoindex on;                 # 디렉터리 목록 표시
        autoindex_exact_size off;     # 파일 크기 KB/MB/GB 단위로 보기 좋게
        autoindex_localtime on;       # 서버 로컬 타임으로 표시
        root /data/repos;
    }
}
EOF
```

```bash
systemctl enable --now nginx
systemctl status nginx.service --no-pager
ss -tnlp | grep nginx

# 접속 테스트
curl http://192.168.10.10/rocky/10/
open http://192.168.10.10/rocky/10/baseos/
```

- 삭제 하기 

```
systemctl disable --now nginx && dnf remove -y nginx
```

### k8s-node


```bash
tree /etc/yum.repos.d/
mkdir /etc/yum.repos.d/backup
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup/
```

- 로컬 레포 파일 생성
    - 서버 IP는 Repo 서버의 IP로 수정
    - NF 클라이언트는 HTTP, HTTPS, FTP, file(로컬 파일시스템 직접 접근) 프로토콜로 저장소에 접근할 수 있다.

```bash
cat <<EOF > /etc/yum.repos.d/internal-rocky.repo
[internal-baseos]
name=Internal Rocky 10 BaseOS
baseurl=http://192.168.10.10/rocky/10/baseos
enabled=1
gpgcheck=0

[internal-appstream]
name=Internal Rocky 10 AppStream
baseurl=http://192.168.10.10/rocky/10/appstream
enabled=1
gpgcheck=0

[internal-extras]
name=Internal Rocky 10 Extras
baseurl=http://192.168.10.10/rocky/10/extras
enabled=1
gpgcheck=0
EOF
```

- 내부 서버 repo 정상 동작 확인

```bash
dnf clean all
dnf repolist
dnf makecache
```


- 패키지 인스톨 정상 실행 확인

```bash
dnf install -y nfs-utils
```

- 패키지 정보에 repo 확인

```
dnf info nfs-utils | grep -i repo
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T072839723Z.png)

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T072847465Z.png)


## 프라이빗 이미지 레지스트리 구성

- podman은 기본적으로 설치되어있다.  아래 명령어로 확인해본다.

```bash
# podman 설치 : 기본 설치 되어 있음
dnf install -y podman
dnf info podman | grep repo
From repo    : appstream

# podman 확인
which podman
podman --version
podman info
cat /etc/containers/registries.conf
cat /etc/containers/registries.conf.d/000-shortnames.conf
```

### admin

- Registry 이미지 받기

```bash
podman pull docker.io/library/registry:latest
podman images
```

- Registry 데이터 저장 디렉터리 준비

```bash
mkdir -p /data/registry
chmod 755 /data/registry
```

- 컨테이너 기동

```bash
podman run -d --name local-registry -p 5000:5000 -v /data/registry:/var/lib/registry --restart=always docker.io/library/registry:latest

```

- 확인 

```bash
podman ps
ss -tnlp | grep 5000
pstree -a
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T073144394Z.png)

- 레지스트리 정상동작 확인

```bash
curl -s http://localhost:5000/v2/_catalog | jq
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T073230250Z.png)


- 컨테이너 이미지 저장소 Docker Registry 에 이미지 push하기

```bash
podman pull alpine
```


```bash
cat /etc/containers/registries.conf.d/000-shortnames.conf | grep alpine
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T073559288Z.png)

```
podman images
podman tag alpine:latest 192.168.10.10:5000/alpine:1.0
```

- 이미지 푸시

```bash
podman push 192.168.10.10:5000/alpine:1.0
```

- 기본적으로 컨테이너 엔진들은 HTTPS를 요구합니다. 내부망에서 HTTP로 테스트하려면 Registry 주소를 '안전하지 않은 저장소'로 등록해야 한다.

```bash
cp /etc/containers/registries.conf /etc/containers/registries.bak
cat <<EOF >> /etc/containers/registries.conf
[[registry]]
location = "192.168.10.10:5000"
insecure = true
EOF
grep "^[^#]" /etc/containers/registries.conf
```

- 이미지 재푸시

```bash
podman push 192.168.10.10:5000/alpine:1.0
```


```
curl -s 192.168.10.10:5000/v2/_catalog | jq
curl -s 192.168.10.10:5000/v2/alpine/tags/list | jq
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T073653276Z.png)


(정리)

```bash
podman rm -f local-registry
mv /etc/containers/registries.bak /etc/containers/registries.conf
```

### k8s-node


```bash
# registries.conf 설정
cp /etc/containers/registries.conf /etc/containers/registries.bak
cat <<EOF >> /etc/containers/registries.conf
[[registry]]
location = "192.168.10.10:5000"
insecure = true
EOF
grep "^[^#]" /etc/containers/registries.conf
```


- 이미지 가져오기

```bash
podman pull 192.168.10.10:5000/alpine:1.0
podman images
```

(정리)

```bash
mv /etc/containers/registries.bak /etc/containers/registries.conf
```
