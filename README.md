### Подготовить стенд на Vagrant и используя Ansible развернуть nginx.

Подготовить стенд на Vagrant как минимум с одним сервером. На этом
сервере используя Ansible необходимо развернуть nginx со следующими
условиями:
- необходимо использовать модуль yum/apt
- конфигурационные файлы должны быть взяты из шаблона jinja2 с
переменными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify для старта nginx после установки
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible.


Создаём каталог Ansible и положим в него Vagrantfile из методички и запустим ВМ `vagrant up`.
Проверяем параметры для подключения с помощью команды `vagrant ssh-config`:
```
Host nginx
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile .vagrant/machines/nginx/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
  PubkeyAcceptedKeyTypes +ssh-rsa
  HostKeyAlgorithms +ssh-rsa
```

Используя эти данные создаём inventory файл:
```
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```

Проверяем может ли Ansible связаться с нашим хостом:
```
feelins@FeeLinS-PC:~/OTUS/lesson2-ansible/ansible$ ansible nginx -i inventory -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

В текущем каталоге создадим файл ansible.cfg:
```
[defaults]
inventory = inventory
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
```

Установим пакет epel-release на наш хост:
```
ansible nginx -m yum -a "name=epel-release state=present" -b
```

Добавим в каталог templates шаблон для конфига NGINX и модуля, который будет
копировать этот шаблон на хост:
```
# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}
```

Создаём Playbook nginx.yml:
```
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes
    
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```

Пробуем его запустить `ansible-playbook nginx.yml`:
```
feelins@FeeLinS-PC:~/OTUS/lesson2-ansible/ansible$ ansible-playbook nginx.yml

PLAY [NGINX | Install and configure NGINX] *************************************

TASK [Gathering Facts] *********************************************************
ok: [nginx]

TASK [NGINX | Install EPEL Repo package from standart repo] ********************
ok: [nginx]

TASK [NGINX | Install NGINX package from EPEL Repo] ****************************
changed: [nginx]

TASK [NGINX | Create NGINX config file from template] **************************
changed: [nginx]

RUNNING HANDLER [reload nginx] *************************************************
changed: [nginx]

PLAY RECAP *********************************************************************
nginx                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
Как видим, всё прошло успешно, ok=2, changed=2. Теперь можно перейти по адресу `http://192.168.56.50:8080/`:

![Скрин](https://github.com/FeeLinS9/Olesson2/blob/master/picture.png)


