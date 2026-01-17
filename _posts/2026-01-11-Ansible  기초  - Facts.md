---
layout: post
title: "[K8S Deploy Study by Gasida] - Ansible 기초  - Facts"
categories:
  - "\bAnsible"
tags:
  - Ansible
  - k8s
---
# Facts

- 앤서블이 관리 호스트에서 자동으로 검색한 변수(자동 예약 변수)
- 팩트에는 플레이, 조건문, 반복문 또는 관리 호스트에서 수집한 값에 의존하는 기타 명령문의 일반 변수처럼 사용 가능한 호스트별 정보가 포함되어 있다.
- 관리 호스트에서 수집된 일부 팩트에는 다음 내용들이 포함될 수 있다.
    -  호스트 이름
    - 커널 버전
    - 네트워크 인터페이스 이름
    - 운영체제 버전
    - CPU 개수
    - 사용 가능한 메모리
    -  스토리지 장치의 크기 및 여유 공간

# Facts 사용하기

기본 활성화로, 플레이북을 실행할 때 자동으로 팩트가 수집된다. 

`project/facts.yaml` 생성

```bash
---

- hosts: db

  tasks:
  - name: Print all facts
    ansible.builtin.debug:
      var: ansible_facts
```

실행하기 

```bash
ansible-playbook facts.yml
```


특정값만 추출해보기

`project/facts1.yaml` 생성

```bash
---

- hosts: db

  tasks:
  - name: Print all facts
    ansible.builtin.debug:
      msg: >
        The default IPv4 address of {{ ansible_facts.hostname }}
        is {{ ansible_facts.default_ipv4.address }}
```

```실행하기
ansible-playbook facts1.yml
```


변수로 사용할 수 있는 앤서블 팩트

|팩트|ansible_facts.* 표기법|
|---|---|
|호스트명|ansible_facts.**hostname**|
|도메인 기반 호스트명|ansible_facts.**fqdn**|
|기본 IPv4 주소|ansible_facts.**default_ipv4.address**|
|네트워크 인터페이스 이름 목록|ansible_facts.**interfaces**|
|/dev/vda1 디스크 파티션 크기|ansible_facts.**device.vda.partitions.vda1.size**|
|DNS 서버 목록|ansible_facts.**dns.nameservers**|
|현재 실행 중인 커널 버전|ansible_facts.**kernel**|
|운영체제 종류|ansible_facts.**distribution**|

구 표기법은 비권장

|==ansible_* 표기법==|ansible_fa**cts.*** 표기법|
|---|---|
|ansible_hostname|ansible_facts.**hostname**|
|ansible_fqdn|ansible_facts.**fqdn**|
|ansible_default_ipv4.address|ansible_facts.**default_ipv4.address**|
|ansible_interfaces|ansible_facts.**interfaces**|
|ansible_device.vda.partitions.vda1.size|ansible_facts.**device.vda.partitions.vda1.size**|
|ansible_dns.nameservers|ansible_facts.**dns.nameservers**|
|ansible_kernel|ansible_facts.**kernel**|
|ansible_distribution|ansible_facts.**distribution**|

- 현재 위 두 개의 표기법 모두 인지한다.
- 이는 앤서블 환경 설정 파일인 ansible.cfg `[defaults]` 섹션에 있는 `inject_facts_as_vars` 매개 변수 기본 설정 값이 `true`이기 때문이며, `false`로 설정하면 `ansible_*` 표기법을 비활성화 할 수 있다.

`project/facts2.yml`

```bash
---

- hosts: db

  tasks:
  - name: Print all facts
    ansible.builtin.debug:
      msg: >
        The node's host name is {{ ansible_hostname }}
        and the ip is {{ ansible_default_ipv4.address }}
```


ansible.cfg 수정후 실행
-  `ansible_*` 표기법을 비활성화 하기 위하여 `ansible.cfg` 수정

```
# ansible.cfg 파일 편집
[defaults]
inventory = ./inventory
remote_user = root
ask_pass = false
inject_facts_as_vars = false

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false

# 실행
ansible-playbook facts2.yml
```

# ansible.builtin.setup 모듈

팩트 수집 기능 비활성화
- 사용자가 팩트 수집으로 인해 호스트에 부하가 걸리는 것을 원치 않을 수도 있다. 이런 경우에는 팩트 수집 기능을 비활성화 할 수 있다.

`project/fact3.yaml`

```bash
---

- hosts: db
  gather_facts: no

  tasks:
  - name: Print all facts
    ansible.builtin.debug:
      msg: >
        The default IPv4 address of {{ ansible_facts.hostname }}
        is {{ ansible_facts.default_ipv4.address }}
```

실행하기

```
ansible-playbook facts3.yml
```


수동으로 팩트수집 기능 활성화
- `project/fact4.yaml`

```bash
---

- hosts: db
  gather_facts: no

  tasks:
  - name: Manually gather facts
    ansible.builtin.setup:

  - name: Print all facts
    ansible.builtin.debug:
      msg: >
        The default IPv4 address of {{ ansible_facts.hostname }}
        is {{ ansible_facts.default_ipv4.address }}
```


실행하기

```
ansible-playbook facts4.yml
```


### Gathering Facts
- **facts** 정보와 **inventory** 정보는 기본적으로 default 캐시 플러그인인 **memory** 플러그인을 사용하고 있다.
- **캐싱**하여 **영구 저장**을 위해서 **파일** 혹은 **데이터베이스**를 사용 할 수 있다.

Gathering Facts 캐싱 설정
- `ansible.cfg`

```bash
[defaults]
inventory = ./inventory
remote_user = root
ask_pass = false
gathering = smart
fact_caching = jsonfile
fact_caching_connection = myfacts

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false
```


### 사용자 정의 팩트 만들기

사용자에 의해 정의된 팩트를 이용하여 환경 설정 파일의 일부 항목을 구성하거나 조건부 작업을 진행할 수 있다.
- 사용자 지정 팩트는 관리 호스트의 로컬에 있는 `/etc/ansible/facts.d` 디렉터리 내에 `*.fact`로 저장되어야만 앤서블이 플레이북을 실행할 때 자동으로 팩트를 수집할 수 있다.

```bash
mkdir /etc/ansible/facts.d

cat <<EOT > /etc/ansible/facts.d/my-custom.fact
[packages]
web_package = httpd
db_package = mariadb-server

[users]
user1 = ansible
user2 = gasida
EOT

cat /etc/ansible/facts.d/my-custom.fact
```


`project/facts5.yml`

```bash
---

- hosts: localhost

  tasks:
  - name: Print all facts
    ansible.builtin.debug:
      var: ansible_local
```

실행하기 

```
ansible-playbook facts5.yml
```