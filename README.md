# Домашнее задание к занятию 5 `«Тестирование roles»` - `Демин Герман`

### Molecule

```
cd roles/clickhouse

molecule test -s ubuntu_xenial
```

Результат теста

![clickhouse-test](/img/clickhouse-test.png)

Тест не отработал, так как синтаксис molecule/ubuntu_xenial/molecule.yml устаревший и в текущих версиях является недействительным

```
cd ../vector-role

molecule init scenario --driver-name docker
```

Команда не отработала, так как в новых версиях молекулы немного поменялся синтаксис команд

Создаю сценарий командой

`molecule init scenario docker`

`nano molecule/docker/molecule.yml` 

```
driver:
  name: docker


ansible:
  cfg:
    defaults:
      host_key_checking: false
      verbosity: 1
      allow_broken_conditionals: True
    ssh_connection:
      pipelining: true
  env:
    ANSIBLE_FORCE_COLOR: "1"
    ANSIBLE_LOAD_CALLBACK_PLUGINS: "1"

  executor:
    backend: ansible-playbook
    args:
      ansible_playbook:
        - --diff
        - --force-handlers

platforms:
  - name: ubuntu
    image: docker.io/library/ubuntu:22.04
    privileged: true
    pre_build_image: true
    override_options:
      container_options:
        detach: true
    ansible_connection: docker
    ansible_user: root

  - name: oraclelinux
    image: docker.io/oraclelinux:9
    privileged: true
    pre_build_image: true
    override_options:
      container_options:
        detach: true
    ansible_connection: docker
    ansible_user: root

scenario:
  test_sequence:
    - dependency
    - create
    - converge
    - idempotence
    - verify
    - cleanup
    - destroy
```

`nano molecule/docker/converge.yml` 

```
---
- name: Converge
  hosts: all
  become: true
  gather_facts: true

  roles:
    - role: vector-role
```



`molecule test --scenario-name docker`

После чего тест написал мне, что пакет vector отсутсвует в репозиториях

Добавляю таски на установку напрямую с репо вектора и необходимых пакетов

```
- name: Install packages on ubuntu
  apt:
    name:
      - python3
      - sudo
      - python3-apt
      - software-properties-common
      - curl
    state: present
  when: ansible_os_family == "Debian"

- name: Download Vector DEB package on Ubuntu
  get_url:
    url: "https://packages.timber.io/vector/latest/vector_latest-1_amd64.deb"
    dest: "/tmp/vector.deb"
  when: ansible_os_family == "Debian"

- name: Install Vector DEB package on Ubuntu
  apt:
    deb: "/tmp/vector.deb"
  when: ansible_os_family == "Debian"

- name: Install packages on oracle
  dnf:
    name:
      - python3
      - sudo
      - python3-dnf
      - curl
    state: present
  when: ansible_os_family == "RedHat"

- name: Download Vector RPM package on OracleLinux
  get_url:
    url: "https://packages.timber.io/vector/latest/vector-latest-1.x86_64.rpm"
    dest: "/tmp/vector.rpm"
  when: ansible_os_family == "RedHat"

- name: Install Vector RPM package on Oracle"
  yum:
    name: "/tmp/vector.rpm"
    state: present
    disable_gpg_check: yes
  when: ansible_os_family == "RedHat"
```

Замечаю таск на проверку сервиса и комменчу, так как внутри докера нет systemd

```
#- name: Enable and start Vector service
#  systemd:
#    name: vector
#    enabled: yes
#    state: started
#  tags:
#    - "3.0"
```

Также отключил по этой причине хендлер на рестарт сервиса и, узнав об этом из ошибки, убрал из tasks/main.yml

`notify: restart vector`

`molecule test --scenario-name docker`

После всех махинаций тест наконец стал успешным

![vector-test1](/img/vector-test1.png)

Добавляю проверки

`nano molecule/default/verify.yml`

```
---
- name: Verify
  hosts: all
  gather_facts: false
  become: true

  tasks:
    - name: Проверка наличия конфигурационного файла vector-role
      ansible.builtin.stat:
        path: /etc/vector/vector.toml
      register: vector_config

    - name: Assert, что конфигурационный файл существует
      ansible.builtin.assert:
        that:
          - vector_config.stat.exists
        fail_msg: "Файл конфигурации vector не найден!"
        success_msg: "Файл конфигурации vector найден."
```

Проверки на сервис я по вышеописанным причинам не добавлял

`molecule test --scenario-name docker`

Проверки также прошли успешно

![vector-test2](/img/vector-test2.png)

Делаю тег

```
git add .

git commit -m "Fix Molecule scenario for Docker; Vector role working"

git tag -a v1.0.1 -m "Working Molecule scenario for vector-role"

git tag

v1.0.1

git push origin v1.0.1
```



### Tox


Добавил файлы tox.ini и tox-requirements.txt в корень роли

`docker run --privileged=True -v /home/drvidjet/Work/Demin/ansible/ansible-playbook-roles:/opt/vector-role -w /opt/vector-role -it aragast/netology:latest /bin/bash`

`tox`

Выполнилась установка модулей

`mkdir molecule/lite`

`dnf install nano`

`nano molecule/lite/molecule.yml`


```
---
driver:
  name: podman

platforms:
  - name: ubuntu
    image: docker.io/library/ubuntu:22.04
    ansible_connection: podman

  - name: oraclelinux
    image: docker.io/oraclelinux:9
    ansible_connection: podman

provisioner:
  name: ansible
  playbooks:
    converge: ../../tasks/main.yml

scenario:
  test_sequence:
    - create
    - converge
    - verify
    - destroy
```

`molecule test --scenario-name lite`

Сценарий не отработал, выдав ошибку модулей

```
[DEPRECATION WARNING]: Ansible will require Python 3.8 or newer on the 
controller starting with Ansible 2.12. Current version: 3.6.8 (default, Jan 14 
2022, 11:04:20) [GCC 8.5.0 20210514 (Red Hat 8.5.0-7)]. This feature will be 
removed from ansible-core in version 2.12. Deprecation warnings can be disabled
 by setting deprecation_warnings=False in ansible.cfg.
/usr/local/lib/python3.6/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 3.6 is no longer supported by the Python core team. Therefore, support for it is deprecated in cryptography. The next release of cryptography will remove support for Python 3.6.
  from cryptography.exceptions import InvalidSignature
Traceback (most recent call last):
  File "/usr/local/bin/molecule", line 5, in <module>
    from molecule.__main__ import main
  File "/usr/local/lib/python3.6/site-packages/molecule/__main__.py", line 24, in <module>
    from molecule.shell import main
  File "/usr/local/lib/python3.6/site-packages/molecule/shell.py", line 29, in <module>
    from molecule import command, logger
  File "/usr/local/lib/python3.6/site-packages/molecule/command/__init__.py", line 46, in <module>
    from molecule.command.init import init  # noqa
  File "/usr/local/lib/python3.6/site-packages/molecule/command/init/init.py", line 25, in <module>
    from molecule.command.init import role, scenario
  File "/usr/local/lib/python3.6/site-packages/molecule/command/init/role.py", line 30, in <module>
    from molecule.command.init import base
  File "/usr/local/lib/python3.6/site-packages/molecule/command/init/base.py", line 27, in <module>
    import cookiecutter.main
  File "/usr/local/lib/python3.6/site-packages/cookiecutter/main.py", line 19, in <module>
    from cookiecutter.repository import determine_repo_dir
  File "/usr/local/lib/python3.6/site-packages/cookiecutter/repository.py", line 11, in <module>
    from cookiecutter.zipfile import unzip
  File "/usr/local/lib/python3.6/site-packages/cookiecutter/zipfile.py", line 9, in <module>
    import requests
  File "/usr/local/lib/python3.6/site-packages/requests/__init__.py", line 43, in <module>
    import urllib3
  File "/usr/lib/python3.6/site-packages/urllib3/__init__.py", line 8, in <module>
    from .connectionpool import (
  File "/usr/lib/python3.6/site-packages/urllib3/connectionpool.py", line 11, in <module>
    from .exceptions import (
  File "/usr/lib/python3.6/site-packages/urllib3/exceptions.py", line 2, in <module>
    from .packages.six.moves.http_client import (
ModuleNotFoundError: No module named 'urllib3.packages.six'
```

В tox.ini я изменил строчки

```
deps =
    -r tox-requirements.txt
    ansible210: ansible<3.0
    ansible30: ansible<3.1
commands =
    {posargs:molecule test -s compatibility --destroy always}
```

На

```
deps =
    -r tox-requirements.txt
commands =
    molecule test --scenario-name lite --destroy always
```

Убрав конфликтующие версии, поправив синтаксис команды и подставив название своего сценария

Снова пробую команду

`tox`

На этот раз она отработала корректно

![vector-test3](/img/vector-test3.png)

`exit`

Делаю тег

```
git add .

git tag -a v0.1.2 -m "Working lite scenario for molecule with podman driver"

git push origin v0.1.2
```

Ссылка на репозиторий, в котором проводилась работа с molecule и tox

https://github.com/awakehns/ansible-playbook-roles
