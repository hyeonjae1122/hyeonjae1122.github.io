---
layout: post
title: "[K8S Deploy Study by Gasida] - Ansible 기초  - Roles 구조 및 사용법"
categories:
  - "\bAnsible"
tags:
  - Ansible
  - k8s
---
# 롤

롤은 플레이북 내용을 기능 단위로 나누어 공통 부품으로 관리/재사용하기 위한 구조다.
- 플레이북에서 전달된 변수를 사용할 수 있다.
- 콘텐츠를 그룹화하여 코드를 다른 사용자와 쉽게 공유할 수 있다.
- 웹 서버, 데이터베이스 서버 또는 깃(Git) 리포지터리와 같은 시스템 유형의 필수 요소를 정의할 수 있다.
- 대규모 프로젝트를 쉽게 관리할 수 있다.
- 다른 사용자와 동시에 개발할 수 있다.
- 잘 작성한 롤은 앤서블 갤럭시를 통해 공유하거나 다른 사람이 공유한 롤을 가져올 수도 있다.

**앤서블 롤 구조** : 롤은 하위 디렉터리 및 파일의 표준화된 구조에 의해 정의된다.
- 최상위 디렉터리는 롤 자체의 이름을 의미하고, 그 안은 tasks 및 handlers 등 롤에서 목적에 따라 정의된 하위 디렉터리로 구성된다.


롤의 최상의 디렉터리 아래에 있는 하위 디렉터리의 이름과 기능

| **하위 디렉터리** | **기능**                                                                                                      |
| ----------- | ----------------------------------------------------------------------------------------------------------- |
| defaults    | 이 디렉터리의 main.yml 파일에는 롤이 사용될 때 덮어쓸 수 있는 롤 변수의 기본값이 포함되어 있다.<br>이러한 변수는 우선순위가 낮으며 플레이에서 변경할 수 있다.            |
| files       | 이 디렉터리에는 롤 작업에서 참조한 정적 파일이 있다.                                                                              |
| handlers    | 이 디렉터리의 main.yml 파일에는 롤의 핸들러 정의가 포함되어 있다.                                                                   |
| meta        | 이 디렉터리의 main.yml 파일에는 작성자, 라이센스, 플랫폼 및 옵션, 롤 종속성을 포함한 롤에 대한 정보가 들어 있다.                                      |
| tasks       | 이 디렉터리의 main.yml 파일에는 롤의 작업 정의가 포함되어 있다.                                                                    |
| templates   | 이 디렉터리에는 롤 작업에서 참조할 Jinja2 템플릿이 있다.                                                                         |
| tests       | 이 디렉터리에는 롤을 테스트하는 데 사용할 수 있는 인벤토리와 test.yml 플레이북이 포함될 수 있다.                                                 |
| vars        | 이 디렉터리의 main.yml 파일은 롤의 변수 값을 정의합니다. 종종 이러한 변수는 롤 내에서 내부 목적으로 사용된다.<br>또한 우선순위가 높으며, 플레이북에서 사용될 때 변경되지 않는다. |

# 롤 생성 
```bash
# 롤 서브 명령어 확인
ansible-galaxy role -h

# 롤 생성
ansible-galaxy role init my-role
```

롤 생성 및 디렉터리 구조 확인

```bash
tree ./my-role/
├── defaults
│   └── main.yml
├── files
│   └── index.html
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```


#### 롤 작성 수순

1. 메인 태스크 작성

```yml
---
# tasks file for my-role

- name: install service {{ service_title }}
  ansible.builtin.apt:
    name: "{{ item }}"
    state: latest
  loop: "{{ httpd_packages }}"
  when: ansible_facts.distribution in supported_distros

- name: copy conf file
  ansible.builtin.copy:
    src: "{{ src_file_path }}"
    dest: "{{ dest_file_path }}"
  notify: 
    - restart service
```

2. index.html 정적 파일 생성
   
```bash
echo "Hello! Ansible" > files/index.html
```

3. 핸들러 작성 : 특정 태스크가 끝나고 그 다음에 수행해야하는 태스크
   
```yaml
---
# handlers file for my-role

- name: restart service
  ansible.builtin.service:
    name: "{{ service_name }}"
    state: restarted   
```

3. defaults(가변변수) 작성 : 외부로부터 재정의 될 수 있는 가변변수
   

```bash
echo '**service_title**: "Apache Web Server"' >> defaults/main.yml
```

   
4. vars(불변변수) 작성: 한번 정의되면 외부로 부터 변수 값을 수정할 수 없음. 롤 내의 플레이북에서만 사용되는 변수로 정의

```yaml
---
# vars file for my-role

service_name: apache2
src_file_path: ../files/index.html
dest_file_path: /var/www/html
httpd_packages:
  - apache2
  - apache2-doc

supported_distros:
  - Ubuntu
```

   
5. Role 추가 (롤 실행을 위해 롤을 호출해주는 플레이북 필요)

플레이북에 롤을 추가하려면 2가지 방법이 존재한다.
- `ansible.builtin.import_role` (정적 - 고정된 롤을 추가)
- `ansible.builtin.include_role` (동적 - 반복문이나 조건문에 의해 롤이 변경될 수 있음)

`project/role-example.yml`

```
cd ..
pwd

---
- hosts: tnode1

  tasks:
    - name: Print start play
      ansible.builtin.debug:
        msg: "Let's start role play"

    - name: Install Service by role
      ansible.builtin.import_role:
        name: my-role
```

위에서 생성한 `role-example.yml`을 생성

```bash
ansible-playbook role-example.yml
```

확인 

```
curl tnode1
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260117T125327559Z.png)

index.html 정적 파일 변경 적용 후 실행해본다.

```bash
echo "안녕하세요! 정적파일을 변경합니다" > my-role/files/index.html
```

![](https://raw.githubusercontent.com/hyeonjae1122/hyeonjae1122.github.io/main/assets/20260117T125444049Z.png)



