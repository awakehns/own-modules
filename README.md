# Домашнее задание к занятию 6 `«Создание собственных модулей»` - `Демин Герман`

### Создание и запуск модуля

`cd lib/ansible/modules`

`nano my_own_module.py`

```
#!/usr/bin/python

from __future__ import (absolute_import, division, print_function)
__metaclass__ = type

DOCUMENTATION = r'''
---
module: my_own_module
short_description: Create text file with given content
version_added: "1.0.0"
description: This module creates a text file at the specified path with provided content.
options:
    path:
        description: Path to the file that will be created.
        required: true
        type: str
    content:
        description: Content to write into the file.
        required: true
        type: str
author:
    - Your Name (@yourGitHubHandle)
'''

EXAMPLES = r'''
- name: Create hello.txt with content
  my_own_module:
    path: /tmp/hello.txt
    content: "Hello from custom module"
'''

RETURN = r'''
changed:
    description: Whether the file was created or modified.
    type: bool
    returned: always
path:
    description: Path of the file.
    type: str
    returned: always
content:
    description: Content written to the file.
    type: str
    returned: always
'''

from ansible.module_utils.basic import AnsibleModule
import os


def run_module():
    module_args = dict(
        path=dict(type='str', required=True),
        content=dict(type='str', required=True),
    )

    result = dict(
        changed=False,
        path='',
        content='',
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    path = module.params['path']
    content = module.params['content']

    result['path'] = path
    result['content'] = content

    if module.check_mode:
        module.exit_json(**result)

    # проверяем, есть ли файл и совпадает ли контент
    if os.path.exists(path):
        with open(path, 'r') as f:
            existing_content = f.read()
        if existing_content == content:
            module.exit_json(**result)

    # создаём/перезаписываем файл
    try:
        with open(path, 'w') as f:
            f.write(content)
        result['changed'] = True
    except Exception as e:
        module.fail_json(msg=str(e), **result)

    module.exit_json(**result)


def main():
    run_module()


if __name__ == '__main__':
    main()
```

`ansible localhost -m my_own_module -a "path=/tmp/test.txt content='Hello Ansible Module'" -c local`


`ERROR: No module named 'jinja2'`


Устанавливаю модуль

`pip install jinja2`

Снова

`ansible localhost -m my_own_module -a "path=/tmp/test.txt content='Hello Ansible Module'" -c local`

```
ERROR: 'NoneType' object is not callable

Traceback (most recent call last):
  File "/home/drvidjet/Work/Demin/ansible/ansible/lib/ansible/cli/__init__.py", line 93, in <module>
    from ansible import constants as C
  File "/home/drvidjet/Work/Demin/ansible/ansible/lib/ansible/constants.py", line 18, in <module>
    config = ConfigManager()
  File "/home/drvidjet/Work/Demin/ansible/ansible/lib/ansible/config/manager.py", line 343, in __init__
    self._base_defs = self._read_config_yaml_file(defs_file or ('%s/base.yml' % os.path.dirname(__file__)))
                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/drvidjet/Work/Demin/ansible/ansible/lib/ansible/config/manager.py", line 415, in _read_config_yaml_file
    return yaml_load(config_def) or {}
           ~~~~~~~~~^^^^^^^^^^^^
TypeError: 'NoneType' object is not callable
```

Делаю вывод, что отсутсвует модуль для обработки yaml, либо установлена неподходящая под задачу версия

`pip install pyYAML`

Снова

`ansible localhost -m my_own_module -a "path=/tmp/test.txt content='Hello Ansible Module'" -c local`


На этот раз все прошло успешно

![start-module1](/img/start-module1.png)

Пробуем в плейбуке

`nano playbook.yml`

```
---
- hosts: localhost
  tasks:
    - name: Create file with my own module
      my_own_module:
        path: /tmp/hello.txt
        content: "Hello from custom module"
```

`ansible-playbook -c local playbook.yml`

Все прошло успешно

![start-module2](/img/start-module2.png)

Запускаем второй раз

![start-module3](/img/start-module3.png)

Идемпотентность подтвердждена


### Создание роли с модулем

`deactivate`

`ansible-galaxy collection init my_own_namespace.yandex_cloud_elk`

`mkdir my_own_namespace/yandex_cloud_elk/plugins/modules`

`cp lib/ansible/modules/my_own_module.py my_own_namespace/yandex_cloud_elk/plugins/modules/`

`ansible-galaxy role init my_own_role --init-path my_own_namespace/yandex_cloud_elk/roles`

`nano my_own_namespace/yandex_cloud_elk/roles/my_own_role/defaults/main.yml`

```
---
path: /tmp/default.txt
content: "Hello from role default"
```

`nano my_own_namespace/yandex_cloud_elk/roles/my_own_role/tasks/main.yml`

```
---
- name: Create file using my own module
  my_own_module:
    path: "{{ path }}"
    content: "{{ content }}"
```

`nano my_own_namespace/yandex_cloud_elk/role_playbook.yml`

```
---
- hosts: localhost
  roles:
    - my_own_role
```

`nano my_own_namespace/yandex_cloud_elk/galaxy.yml`

```
namespace: my_own_namespace
name: yandex_cloud_elk
version: 1.0.0
readme: README.md
authors:
  - "awakehns awake1394hns@gmail.com"
description: "Collection that contains a custom module to create text files and a role to use it."
license:
  - GPL-3.0-or-later
tags:
  - files
  - example
  - custom-module
```

`nano my_own_namespace/yandex_cloud_elk/README.md`

```
# my_own_namespace.yandex_cloud_elk

Collection: `my_own_namespace.yandex_cloud_elk`

Короткое описание:
This collection contains a custom module `my_own_module` that writes a text file at `path` with `content`, and a role `my_own_role` that wraps it.

## Установка (локально)
```bash
ansible-galaxy collection install my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz -p ./collections
```

Создаю тег

```
git remote set-url origin git@github.com:awakehns/my_own_collection.git

git add .

git commit -m "Add custom module and role"

git tag 1.0.0

git push origin main --tags
```

Собираю архив

`ansible-galaxy collection build`

Тест

```
mkdir test_collection && cd test_collection

cp ../my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz ./

cp ../role_playbook.yml ./

ansible-galaxy collection install my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
```

Установка прошла успешно

![install](/img/install.png)

Запуск финального playbook

`cd ..`

`ansible-playbook -c local role_playbook.yml`

Все прошло успешно

![playbook](/img/playbook.png)


### Ссылка на репозиторий с коллекцией:

#### https://github.com/awakehns/my_own_collection

### Коллеция

#### https://github.com/awakehns/my_own_collection/tree/main/my_own_namespace/yandex_cloud_elk

### tar.gz внутри тега

#### https://github.com/awakehns/my_own_collection/releases/tag/1.0.0

### tar.gz в репо

#### https://github.com/awakehns/my_own_collection/blob/main/my_own_namespace/yandex_cloud_elk/my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
