---
layout: post
tags:
  - Ansible
  - k8s
  - kubernetes
  - kubespray
title: "[K8S Deploy Study by Gasida] - Kubespray offline 설치 3 k8s 설치"
---


오프라인(폐쇄망) 환경에서 Kubespray를 사용해 Kubernetes 클러스터를 설치하려면, 설치에 필요한 바이너리/컨테이너 이미지/Python 패키지/RPM(deb) 패키지까지 사전에 모두 다운로드해 내부에 제공할 수 있어야 한다.

#  준비사항

## 디스크 용량 확보

- 기본 60G에서 120G로 증설

```bash
lsblk
df -hT /
```

#  repo clone & 전체 다운로드 (약 3.3GB / 17분)

오프라인 설치에 필요한 파일/이미지/패키지/리포 구성을 전부 `outputs/` 아래로 만들어 둔다.

```bash
git clone https://github.com/kubespray-offline/kubespray-offline
cd kubespray-offline

```

- 변수 정보 확인 
```bash
source ./config.sh
echo -e "kubespary $KUBESPRAY_VERSION"
echo -e "runc $RUNC_VERSION"
echo -e "containerd $CONTAINERD_VERSION"
echo -e "nercdtl $NERDCTL_VERSION"
echo -e "cni $CNI_VERSION"
echo -e "nginx $NGINX_VERSION"
echo -e "registry $REGISTRY_VERSION"
echo -e "registry_port: $REGISTRY_PORT"
echo -e "Additional container registry hosts: $ADDITIONAL_CONTAINER_REGISTRY_LIST"
echo -e "cpu arch: $IMAGE_ARCH"

```


```bash
./download-all.sh
```

- `download-all.sh `의 흐름 :  여러 스크립트를 순차 실행한다.
    - `config.sh` →` target-scripts/config.sh` : 버전 변수 로드
    - `precheck.sh` : 환경점검
    - `prepare-pkg.sh` : 의존 패키지 설치
    - `prepare-py.sh ` : python venv + requirements 설치
    - `get-kubespray.sh` : kubespray tarball 확보 및 패치
    - `pypi-mirror.sh` pip 패키지 미러 생성
    - `download-kubespray-files.sh` : kubesprary offline 목록생성 + 파일 / 이미지 다운로드    - 
    - `download-additional-containers.sh` : 추가 이미지 다운로드 (nignx/registry등)
    - `create-repo.sh`  : RPM 리포 생성(createrepo)  
    - `copy-target-scripts.sh` : outputs에 설치 스크립트 복사

버전이 바뀌면 대부분 **이 단계에서 다운로드 대상이 달라지므로**, 버전 변수는 `target-scripts/config.sh` 에서 관리하는 게 핵심이다. 즉 버전 변경 시엔 config에서 바꾸고 다시 download-all을 해야한다. 

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T081145409Z.png)


## precheck / 패키지 / python venv

### precheck.sh

- `podman` 외 런타임 사용 시 설치 여부 확인   
- RHEL7 + SELinux enforcing이면 중단(미지원)
    
### prepare-pkgs.sh

```bash
dnf install -y rsync gcc libffi-devel createrepo git podman createrepo_c
```
RHEL 계열 
- `rsync gcc libffi-devel createrepo git podman ...`
- RHEL 10에서 `createrepo_c` 관련 처리
    
### prepare-py.sh → venv.sh

```bash
pip install -U pip setuptools & pip install -r requirements.txt
```

- python venv 생성    
- `pip install -U pip setuptools`
- `pip install -r requirements.txt`
    
venv 경로는 기본으로:
- `~/.venv/<python버전>/`

## kubespray 소스 확보(get-kubespray.sh)

`KUBESPRAY_VERSION`이 커밋 해시라면: git clone 후 해당 커밋 checkout한다.
- `master` 또는 `release-...` 일 경우 : `branch clone`    
- 일반 버전이라면: `v${KUBESPRAY_VERSION}.tar.gz` 다운로드
    

다운로드 후
- `outputs/files/kubespray-<ver>.tar.gz` 로 저장
   - `cache/kubespray-<ver>/` 로 extract 후 필요하면 patch 적용

## PyPI 미러 만들기(pypi-mirror.sh)

오프라인 노드에서 pip를 쓰려면 wheel/sdist를 내부에 모아두고 index를 만들어야 한다.

`pypi-mirror.sh`의 흐름
- `python-pypi-mirror` 설치
- `kubespray requirements`
    - binary 패키지(3.11, 3.12 대상) 다운로드
    - `source` 패키지 다운로드
    - `pip/setuptools/wheel` 등 추가 다운로드        
- `pypi-mirror create` 로 HTML 인덱스 생성

## kubespray offline 목록 생성 + 파일/이미지 다운로드

`download-kubespray-files.sh`는 kubespray repo 안의`contrib/offline/generate_list.sh`
를 실행해서 `files.list`, `images.list` 를 만든다.
- `outputs/files/files.list`
    - `kubeadm`/`kubelet`/`kubectl`,`etcd`, `cni`, `runc`, `containerd`, `nerdctl` 등 URL 목록    
- `outputs/images/images.list` 
    - 필요한 컨테이너 이미지 목록   
- `files.list`
    - 등록된 URL들을 curl로 다운로드    
- `images.list`
    - `download-images.sh`로 pull & save(압축) 수행
-  `download-additional-containers.sh`가 `imagelists/*.txt` 기반으로 nginx/registry 같은 추가 이미지도 다운로드
    

## 로컬 repo 생성(create-repo.sh)

RHEL일 경우 
- `scripts/create-repo-rhel.sh` 실행
    - `dnf download --resolve --alldeps ...`
    - `outputs/rpms/local/` 에 RPM 복사    
- `createrepo` 실행해 repodata를 생성한다.
    

> 참고: `/bin/rm outputs/rpms/local/*.i686.rpm` 같은 메시지는  
> i686 RPM이 없어서 “지울 게 없다”는 뜻이라 보통 무시해도 됨.

## download-all 결과물 확인

다운로드 완료 후 `outputs/` 용량은 약 3.3GB정도이다. 

`outputs/` 의 구조
- `files/`
    - kubeadm/kubelet/kubectl, containerd 등 바이너리
 - `images/` 
     - 컨테이너 이미지 tar.gz
- `pypi/` 
    - pip 패키지 (index.html, *.whl, *.tar.gz)
- `rpms/`
    - dnf repo    
- `playbook/`
    - 오프라인 repo 설정용 role/playbook    
- 기타 설치 스크립트
    - `setup-*`, `start-*`, `install-containerd.sh` 등
    



# outputs로 이동 후 containerd 설치 + 이미지 로드

```bash
cd /root/kubespray-offline/outputs
./setup-container.sh
```

`setup-container.sh`의 역할

- `install-containerd.sh` 실행 (로컬 files에서 runc/containerd/nerdctl/cni 설치)    
- nginx/registry 이미지 tar.gz를 nerdctl로 load
    

- 확인

```bash
which runc && runc --version
which containerd && containerd --version
which nerdctl && nerdctl --version
nerdctl images
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T090201554Z.png)

#  nginx 기동 (files/images/pypi/rpms 제공)

```bash
./start-nginx.sh
```


nginx는 `--network host` 로 뜨고 `outputs/` 전체를 `/usr/share/nginx/html`로 마운트해서 웹서버로 제공한다.

디렉터리 목록이 보이게 하려면 `nginx-default.conf`의 아래 옵션을 추가
- `autoindex on;`
- `autoindex_exact_size off;`
- `autoindex_localtime on;`

```bash
cat << EOF > nginx-default.conf 
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        # index  index.html index.htm;

        autoindex on;                 # 디렉터리 목록 표시
        autoindex_exact_size off;     # 파일 크기 KB/MB/GB 단위로 보기 좋게
        autoindex_localtime on;       # 서버 로컬 타임으로 표시
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # Force sendfile to off
    sendfile off;     
}
EOF

```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T090250060Z.png)

# 오프라인 repo / pip mirror 시스템 설정

```bash
./setup-offline.sh
```

RHEL 계열

- `/etc/yum.repos.d/*.repo` 를 `.original`로 바꿔 비활성화
- `/etc/yum.repos.d/offline.repo` 생성
    - `baseurl=http://localhost/rpms/local/`
        

pip
- `~/.config/pip/pip.conf`
    - `index-url=http://localhost/pypi/`
    - `trusted-host=localhost`
        

- 설정 확인

```bash
dnf repolist 
cat /etc/yum.repos.d/offline.repo # offline.repo 파일 확인
cat ~/.config/pip/pip.conf # pypi mirror 설정 확인
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T090418865Z.png)

# Python 설치로 offline repo 동작 검증

```bash
./setup-py.sh
```

Rocky 10에서는 python 버전이 3.12로 자동 설정되는 구조
- (예: `pyver.sh`에서 10* → PY=3.12).

오프라인 설치 옵션
- `--disablerepo=* --enablerepo=offline-repo`


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T090525082Z.png)




# registry 기동 + 이미지 push

## registry 기동

- 기본 포트 :  `REGISTRY_PORT=35000`   
- 저장 경로 :  `/var/lib/registry` (volume mount)

```bash
./start-registry.sh
```
    
- 확인

```bash
nerdctl ps
ss -tnlp | grep registry
```

- 현재는 registry 서버 내부에 저장된 (컨테이너) 이미지 없는 상태
```bash
tree /var/lib/registry/
```

```
curl 192.168.10.10:5001/metrics
curl 192.168.10.10:5001/debug/pprof/
```
`
## 이미지 load & push

```bash
./load-push-all-images.sh
```


```bash
vi load-push-all-images.sh
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T090921285Z.png)

```bash
image might be filtered out (Hint: set --platform=... or --all-platforms)
```

ARM64 환경에서 tar에 멀티 플랫폼이 섞여 있으면 발생할 수 있어서, load 시에`nerdctl load --all-platforms -i <image.tar.gz>`를 쓰도록 스크립트를 수정해서 해결한다.

정상적으로 push 후에는 `localhost:35000/<repo>:<tag>` 형태로 레지스트리에 올라간다.
    
- kube-proxy 컨테이너가 localhost:35000 과 registry.k8s.io 로 push 된 상태 확인

```bash
nerdctl images | grep -i kube-proxy
nerdctl images | grep localhost
nerdctl images | grep localhost | wc -l
```


![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T091306360Z.png)


- 카탈로그 / 태그 확인

```bash
curl -s http://localhost:35000/v2/_catalog | jq
curl -s http://localhost:35000/v2/kube-apiserver/tags
curl -s http://localhost:35000/v2/kube-apiserver/tags/list | jq
curl -s http://localhost:35000/v2/kube-apiserver/manifests/v1.34.3 | jq
tree /var/lib/registry/ -L 5
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T091335260Z.png)
![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T091357016Z.png)

#  kubespray tarball 압축 해제

```bash
`./extract-kubespray.sh`
```

- `files/kubespray-<ver>.tar.gz`를 풀어 `kubespray-<ver>/` 생성    
- 버전별 patch 디렉터리가 있으면 적용


- `kubespary` 저장소 압축 해제된 파일들 확인

```
tree kubespray-2.30.0/ -L 1
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260214T091525152Z.png)