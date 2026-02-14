---
layout: post
tags:
  - Ansible
  - k8s
  - kubernetes
  - kubespray
title: "[K8S Deploy Study by Gasida] - Kubespray offline 설치 - 5 offline.yml 파일에 각 value에 대한 동작"
---
폐쇄망(Air-gapped) 환경에서 Kubespray를 성공적으로 배포하기 위한 가장 핵심적인 파일이 바로 **`offline.yml`**이다.

Kubespray는 기본적으로 수십 개의 바이너리와 수백 개의 컨테이너 이미지를 인터넷(GitHub, GCR, Quay, DockerHub 등)에서 다운로드하도록 설계되어 있다. `offline.yml`의 역할은 이러한 모든 외부 호출(Call)을 가로채서, 우리가 구축한 내부망의 웹 서버와 레지스트리로 방향을 틀어주는 역할을 한다.

# 1. 기본 인프라 주소  정의 (Base Endpoints)

```bash
http_server: "http://YOUR_HOST"
registry_host: "YOUR_HOST:35000"
```

- `http_server`
    - `Kubeadm`, `kubectl`, `cni` 등 정적 바이너리 파일과 OS 패키지(RPM/DEB)를 서빙하는 내부 `Nginx` 웹 서버 주소
    - `files_repo`나 `yum_repo` 변수에서 `{{ http_server }}` 형태로 치환되어 사용된다.
- `registry_host`
    - K8s 컴포넌트 이미지가 저장된 사설 컨테이너 레지스트리 주소(IP:Port)입니다.

# 2.  컨테이너 런타임 미러 설정 (Containerd Registries Mirrors)

이 값들은 타겟 노드(`k8s-node`)들의 `/etc/containerd/config.toml` 파일 내 `[plugins."io.containerd.grpc.v1.cri".registry.mirrors]` 섹션을 동적으로 생성한다.

```
containerd_registries_mirrors:
  - prefix: "{{ registry_host }}"
    mirrors:
      - host: "http://{{ registry_host }}"
        capabilities: ["pull", "resolve"]
        skip_verify: true
```

K8s의 기본 런타임인 Containerd는 외부 통신 시 HTTPS를 강제한다. 내부망에서 Self-signed를 쓰거나 HTTP 통신을 할 때 발생하는 `x509: certficate signed by unknown authority` 에러를 방지하기 위해 해당 레지스트리에 대해 `skip_verify: true` 를 설정하여 containerd의 `config.toml`에 주입하는 필수 설정이다. 




# 3. 다운로드 디렉터리 경로 매핑 (Repository Paths)

```bash
files_repo: "{{ http_server }}/files"
yum_repo: "{{ http_server }}/rpms"
ubuntu_repo: "{{ http_server }}/debs"
```

Nginx 서버 내부의 디렉터리 구조를 매핑한다.
- `files_repo`: `.tar.gz`, 바이너리 파일들이 위치한 경로.
- `yum_repo` / `ubuntu_repo`: OS별 패키지 매니저가 바라볼 경로.

# 4.  이미지 레지스트리 우회 (Registry Overrides) ⭐ 가장 중요 ⭐

앤서블이 각 노드에 배포할 Static Pod YAML (예: `/etc/kubernetes/manifests/kube-apiserver.yaml`)이나 DaemonSet/Deployment 매니페스트를 생성(Render)할 때 개입한다. 

예를 들어 원본 템플릿에 `image: {{ kube_image_repo }}/kube-apiserver:v1.34.3`라고 되어 있다면, 원래는 `registry.k8s.io`가 들어가야 할 자리를 이 변수들이 덮어씌워 `192.168.10.10:35000/kube-apiserver:v1.34.3`로 문자열을 강제 치환(Replace)한다.

```
kube_image_repo: "{{ registry_host }}"
gcr_image_repo: "{{ registry_host }}"
docker_image_repo: "{{ registry_host }}"
quay_image_repo: "{{ registry_host }}"
github_image_repo: "{{ registry_host }}"
```

Kubespray 플레이북 내부에는 구글(`gcr.io`), 레드햇(`quay.io`), 도커허브(`docker.io`), 깃허브(`ghcr.io`) 등 다양한 퍼블릭 레지스트리가 하드코딩 되어 있다. 이 변수들을 모두 각자의 `{{ registry_host }}`로 덮어쓰기함으로써, 각 노드가 이미지를 Pull 할 때 인터넷으로 나가지 않고 로컬 레지스트리를 찌르도록 만든다.


```bash
local_path_provisioner_helper_image_repo
```

 기본 스토리지 프로비저너인 `local-path-provisioner`가 디렉터리를 만들거나 지울 때 임시로 띄우는 `busybox` 이미지의 다운로드 경로를 사설 레지스트리로 변경한다.


# 5.  컴포넌트별 바이너리 다운로드 URL (Download URLs)

Kubespray는 `roles/download/tasks/download_file.yml` 이라는 공통 다운로드 태스크를 사용해 노드에 필요한 바이너리를 가져온다. 이때 아래 변수들이 `get_url` (혹은 `curl`) 모듈의 `url` 파라미터로 직접 전달된다.

```bash
kubeadm_download_url: "{{ files_repo }}/kubernetes/v{{ kube_version }}/kubeadm"
cni_download_url: "{{ files_repo }}/kubernetes/cni/cni-plugins-linux-{{ image_arch }}-v{{ cni_version }}.tgz"
nerdctl_download_url: "{{ files_repo }}/nerdctl-{{ nerdctl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
# ... (기타 etcd, calicoctl, helm, containerd 등)
```

- `kubeadm_download_url`, `kubectl_download_url`, `kubelet_download_url`
        - 구글 저장소(`dl.k8s.io`) 대신 사내 Nginx(`/files/kubernetes/...`)에서 K8s 3대 핵심 바이너리를 다운로드하도록 앤서블에 지시한다. 다운로드 후 노드의 `/usr/local/bin`에 배치된다.
        
- `etcd_download_url`    
    - `etcd` 바이너리 압축 파일을 로컬 `Nginx`에서 다운로드하도록 지시한다. `{{ etcd_version }}` 변수와 결합하여 정확한 버전의 tar.gz를 받아 압축을 풉니다.
        
- `cni_download_url`, `crictl_download_url`
        -  파드 네트워크 대역 설정을 위한 CNI 플러그인 바이너리와, 컨테이너 런타임 디버깅 도구(crictl)를 Nginx에서 가져와 `/opt/cni/bin/` 및 `/usr/local/bin/`에 각각 복사한다.
        
- `calicoctl_...`, `ciliumcli_...`, `helm_...` 등
        - `k8s-cluster.yml`에서 해당 애드온(Calico, Cilium, Helm)을 `true`로 활성화했을 때만 작동한다. 활성화된 경우 앤서블은 퍼블릭 GitHub 릴리즈 페이지가 아닌, Nginx(`/files/...`)에서 해당 CLI 도구나 CRD(Custom Resource Definition) 매니페스트를 다운로드한다.
        
- `containerd_download_url`, `runc_download_url`, `nerdctl_download_url`
        - 컨테이너 런타임 스택(Containerd, runc, nerdctl)의 바이너리를 Nginx에서 가져와 설치한다. 타겟 노드가 인터넷이 안 되어 OS 패키지 매니저로 설치가 불가능할 때, 이 URL을 통해 다운로드한 정적 바이너리로 런타임 환경을 구성한다.

# 6. (선택) 접근 제어 및 보안 (Access Control & Vault)


```bash
files_repo_user: download
files_repo_pass: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          61663232...

files_repo: "https://{{ files_repo_user ~ ':' ~ files_repo_pass ~ '@' ~ files_repo_host ~ files_repo_path }}"
```

엔터프라이즈 환경에서는 내부망의 파일 서버나 레지스트리에도 계정 인증(Basic Auth)이 걸려있는 경우가 많다. 이때 평문으로 비밀번호를 적어두면 보안 취약점이 되므로, Ansible Vault를 사용해 비밀번호를 암호화(AES256)하여 저장한다. 플레이북 실행 시 Vault 패스워드를 입력하여 복호화된 정보로 안전하게 다운로드를 수행한다.

 - `offline.yml`에 이 값들이 세팅되면, 앤서블은 다운로드 태스크(`get_url`)를 실행할 때 `url_username`과 `url_password` 파라미터를 추가하여 Nginx의 Basic Auth 인증을 통과한다. 레지스트리의 경우, 타겟 노드의 Containerd 자격증명 파일에 이 계정 정보를 기록하여 로그인된 상태로 이미지를 Pull 하도록 설정한다.

# 7.  OS별 패키지 레포지토리 (OS Specific Repos)

```bash
# CentOS/Redhat/AlmaLinux/Rocky Linux
docker_rh_repo_base_url: "{{ yum_repo }}/docker-ce/$releasever/$basearch"
docker_rh_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"
```

- 앤서블의 `yum_repository` (RHEL 계열) 또는 `apt_repository` (Debian 계열) 모듈에 전달되는 값이다. 
- 노드의 `/etc/yum.repos.d/docker-ce.repo` (혹은 apt sources.list) 파일을 생성할 때, `baseurl` 항목을 `download.docker.com` 대신 `{{ yum_repo }}/docker-ce/...` (내부 Nginx 경로)로 강제 지정한다. 이를 통해 노드에서 `dnf install containerd.io` 명령이 실행될 때 인터넷이 아닌 내부 서버에서 RPM 패키지를 가져오게 만든다.


오프라인 배포 시 특정 컴포넌트(예: CNI나 메트릭 서버) 설치에서 타임아웃 에러가 난다면 `offline.yml`에 해당 컴포넌트의 Override URL이 누락되어 Ansible이 외부 인터넷을 향하고 있는지 가장 먼저 이 파일의 URL 매핑 상태를 점검하자. 