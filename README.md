# Домашнее задание № 27 по теме: "Развёртываение веб-приложений". К курсу Administrator Linux. Professional

## Задание

Создать vagrant-стэнд с докером, с проброшенными портами на хост (8081, 8082, 8083), со следующими контейнерами

Front:
- Nginx 1.2.25.5

Back:
- Database MySQL 8.3.0

- PHP-fpm wordpress-6.5.2 (port 8083)
- node.js 22 (port 8082)
- Django 5.0.4, python-3.12.3 (port 8081)

## Выполнение

Конфигурация vagrant-стэнда:

```json
[
  {
    "name": "dynweb",
    "cpus": 2,
    "gui": false,
    "box": "centos/8",
    "private_network":
    [
      { "ip": "192.168.56.11", "adapter": 2, "netmask": "255.255.255.0" }
    ],
    "forward_port":
    [
      { "guest": 8081, "host": 8081, "auto_correct": true },
      { "guest": 8082, "host": 8082, "auto_correct": true },
      { "guest": 8083, "host": 8083, "auto_correct": true }
    ],
    "memory": 2048,
    "no_share": true
  }
]
```

Для развертывания docker использована роль geerlingguy.docker

```bash
ansible-galaxy role install geerlingguy.docker
```

Playbook:

```yaml
- name: Dynweb
  hosts: all
  become: yes

  vars:
    disabled: false

  tasks:

  - name: Accept login with password from sshd
    ansible.builtin.lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^PasswordAuthentication no$'
      line: 'PasswordAuthentication yes'
      state: present
    notify: Restart sshd

  - name: Set timezone
    community.general.timezone:
      name: Europe/Moscow

  - name: List all files in directory /etc/yum.repos.d/*.repo
    find:
      paths: "/etc/yum.repos.d/"
      patterns: "*.repo"
    register: repos
    when: (ansible_os_family == "RedHat" and disabled == false)

  - name: Comment mirrorlist /etc/yum.repos.d/CentOS-*
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^(mirrorlist=.+)'
      line: '#\1'
    with_items: "{{ repos.files }}"
    when: (ansible_os_family == "RedHat" and disabled == false)

  - name: Replace baseurl
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^#baseurl=http:\/\/mirror.centos.org(.+)'
      line: 'baseurl=http://vault.centos.org\1'
    with_items: "{{ repos.files }}"
    when: (ansible_os_family == "RedHat" and disabled == false)

  - name: Install base tools
    ansible.builtin.dnf:
      name:
        - vim
      state: present
      update_cache: true
    when: disabled == false

  handlers:
  - name: Restart sshd
    ansible.builtin.service:
      name: sshd
      state: restarted

- name: Docker setup
  hosts: dynweb
  become: true
  roles:
    - geerlingguy.docker

- name: After docker setup
  hosts: dynweb
  become: yes
  tasks:
  - name: Add remote "vagrant" user to "docker" group
    ansible.builtin.user:
      name: vagrant
      groups: "docker"
      append: yes

  - name: Copy project directory
    ansible.builtin.copy:
      src: project
      dest: /home/vagrant
      owner: vagrant
      group: vagrant
      mode: '0640'
    notify: Start docker compose

  handlers:

  - name: Start docker compose
    ansible.builtin.shell:
      cmd: 'docker compose -f docker-compose.yml up -d'
      chdir: /home/vagrant/project
```

docker-compose:

```yaml
services:

  database:
    image: mysql
    container_name: database
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - ./dbdata:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - app-network

  wordpress:
    image: wordpress:6.5.2-fpm-alpine
    container_name: wordpress
    restart: unless-stopped
    environment:
      WORDPRESS_DB_HOST: database
      WORDPRESS_DB_NAME: "${DB_NAME}"
      WORDPRESS_DB_USER: root
      WORDPRESS_DB_PASSWORD: "${DB_ROOT_PASSWORD}"
    volumes:
      - ./wordpress:/var/www/html
    networks:
      - app-network
    depends_on:
      - database

  nginx:
    image: nginx:1.25.5-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - 8081:8081
      - 8082:8082
      - 8083:8083
    volumes:
      - ./wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
    networks:
      - app-network
    depends_on:
      - wordpress
      - app
      - node

  app:
    build: ./python
    container_name: app
    restart: always
    env_file:
      - .env
    command:
      "gunicorn --workers=2 --bind=0.0.0.0:8000 mysite.wsgi:application"
    networks:
      - app-network

  node:
    image: node:22-alpine3.18
    container_name: node
    working_dir: /opt/server
    volumes:
      - ./node:/opt/server
    command: node test.js
    networks:
      - app-network


networks:
  app-network:
    driver: bridge
```

Конфиг nginx:

```
upstream django {
  server app:8000;
}

server {
   listen 8081;
   listen [::]:8081;
   server_name localhost;
  location / {
    try_files $uri @proxy_to_app;
  }
  location @proxy_to_app {
    proxy_pass http://django;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $server_name;
  }
}

server {
   listen 8082;
   listen [::]:8082;
   server_name localhost;
  location / {
    proxy_pass http://node:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $server_name;
  }
}

server {
        listen 8083;
        listen [::]:8083;
        server_name localhost;
        index index.php index.html index.htm;

        root /var/www/html;
        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }
        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }
        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location = /favicon.ico {
                log_not_found off; access_log off;
        }

        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}
```

Изменён Dockerfile для сервиса app (Django):

```
FROM python:3.12.3-alpine3.18
ENV APP_ROOT /src
ENV CONFIG_ROOT /config
RUN mkdir ${CONFIG_ROOT}
COPY requirements.txt ${CONFIG_ROOT}/requirements.txt
RUN pip install -r ${CONFIG_ROOT}/requirements.txt
RUN mkdir ${APP_ROOT}
RUN django-admin startproject mysite ${APP_ROOT}
WORKDIR ${APP_ROOT}
```

Проверка Django

```bash
curl -I http://localhost:8081
```

```
HTTP/1.1 200 OK
Server: nginx/1.25.5
Date: Tue, 30 Apr 2024 12:43:16 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 10629
Connection: keep-alive
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin
```

Проверка node.js

```bash
curl http://localhost:8082
```

```
Hello from node js server
```

Проверка php-fpm (wordpress)

```bash
curl -I http://localhost:8083
```

```
HTTP/1.1 200 OK
Server: nginx/1.25.5
Date: Tue, 30 Apr 2024 12:46:02 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Powered-By: PHP/8.2.18
Link: <http://localhost:8083/index.php?rest_route=/>; rel="https://api.w.org/"
```
