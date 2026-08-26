# Ansible + Nginx — практика

## 1. Виртуальное окружение

Ansible установлен в отдельном Python `venv`, чтобы не использовать глобальную установку.

Активация окружения:

```bash
source ~/ansible-env/bin/activate
```

После активации:

```text
(ansible-env) ansible-user@kirillPC:~$
```

---

## 2. Структура Ansible-проекта

Создал следующую структуру:

```text
ansible/
├── ansible.cfg
├── inventory/
│   └── hosts.yml
├── playbooks/
│   └── update.yml
└── roles/
    ├── nginx/
    │   ├── files/
    │   │   └── index.html
    │   ├── handlers/
    │   │   └── main.yml
    │   └── tasks/
    │       └── main.yml
    │
    └── update-packages/
        └── tasks/
            └── main.yml
```

---

# 3. SSH-доступ к серверу

Создал SSH-ключ:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_key
```

Создаются два файла:

```text
~/.ssh/ansible_key       # приватный ключ
~/.ssh/ansible_key.pub   # публичный ключ
```

Публичный ключ добавляется на удалённый сервер.

Проверить SSH-подключение:

```bash
ssh -i ~/.ssh/ansible_key ansible-user@<SERVER_IP>
```

---

# 4. Inventory

В `inventory/hosts.yml` описал сервер, которым должен управлять Ansible:

```yaml
all:
  hosts:
    devops-trening-serv:
      ansible_host: <SERVER_IP>
      ansible_user: ansible-user

  vars:
    ansible_ssh_private_key_file: ~/.ssh/ansible_key
```

Здесь:

- `devops-trening-serv` — имя сервера внутри Ansible;
- `ansible_host` — реальный IP сервера;
- `ansible_user` — пользователь для SSH;
- `ansible_ssh_private_key_file` — приватный SSH-ключ.

---

# 5. Playbook

Создал:

```text
playbooks/update.yml
```

Пример:

```yaml
---
- name: Обновление пакетов
  hosts: all
  become: true

  roles:
    - update-packages
    - nginx
```

`hosts: all` — выполнить playbook на всех серверах из inventory.

`become: true` — Ansible будет повышать привилегии через `sudo`.

---

# 6. Роль обновления пакетов

Файл:

```text
roles/update-packages/tasks/main.yml
```

```yaml
---
- name: Обновить все пакеты через APT
  ansible.builtin.apt:
    upgrade: yes
    update_cache: yes
```

Эта задача:

1. обновляет кэш APT;
2. обновляет установленные пакеты.

---

# 7. Роль Nginx

Файл:

```text
roles/nginx/tasks/main.yml
```

```yaml
---
- name: Установить nginx
  ansible.builtin.apt:
    name: nginx
    state: present

- name: Копировать кастомную страницу index.html
  ansible.builtin.copy:
    src: files/index.html
    dest: /var/www/html/index.html
  notify: Перезапустить nginx

- name: Включить и запустить nginx
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

Что происходит:

```text
Ansible
   ↓
Устанавливает nginx
   ↓
Копирует index.html
   ↓
/var/www/html/index.html
   ↓
Запускает nginx
```

---

# 8. Handler

Файл:

```text
roles/nginx/handlers/main.yml
```

```yaml
---
- name: Перезапустить nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

В задаче копирования страницы указано:

```yaml
notify: Перезапустить nginx
```

Если `index.html` изменился, Ansible вызывает handler и перезапускает Nginx.

Если файл не изменился — перезапуск не выполняется.

---

# 9. Запуск playbook

Запуск:

```bash
ansible-playbook playbooks/update.yml -i inventory/hosts.yml -k
```

После выполнения получил:

```text
ok=5
changed=0
unreachable=0
failed=0
```

`failed=0` — ошибок нет.

`unreachable=0` — Ansible смог подключиться к серверу.

`changed=0` — сервер уже находился в нужном состоянии, поэтому ничего менять не пришлось.

Это пример **идемпотентности Ansible**.

---

# 10. Проверка Nginx

Зашёл на удалённый сервер по SSH.

Проверил версию:

```bash
nginx -v
```

Получил:

```text
nginx version: nginx/1.24.0 (Ubuntu)
```

Проверил состояние сервиса:

```bash
sudo systemctl status nginx
```

Получил:

```text
Active: active (running)
```

Значит Nginx запущен.

---

# 11. Проверка HTML-страницы

На самом сервере:

```bash
curl http://localhost
```

Получил:

```text
HELLO WORLD
```

Значит цепочка работает:

```text
curl
 ↓
localhost:80
 ↓
Nginx
 ↓
/var/www/html/index.html
 ↓
HELLO WORLD
```

---

# 12. Проверка порта 80

Проверил, слушает ли Nginx порт `80`:

```bash
sudo ss -lntp | grep ':80'
```

Получил:

```text
0.0.0.0:80
[::]:80
```

`0.0.0.0:80` означает, что Nginx слушает TCP-порт `80` на всех IPv4-интерфейсах сервера.

---

# 13. Проверка UFW

Проверил локальный firewall Ubuntu:

```bash
sudo ufw status
```

Получил:

```text
Status: inactive
```

Значит UFW не блокирует подключения.

`UFW (Uncomplicated Firewall)` — простой инструмент Ubuntu для управления firewall.

---

# 14. Почему сайт не открывался из интернета

На самом сервере:

```bash
curl localhost
```

работал:

```text
HELLO WORLD
```

Но с моего компьютера:

```bash
curl http://<SERVER_IP>
```

сайт не открывался.

Проблема оказалась не в:

```text
Ansible ❌
Nginx ❌
Ubuntu ❌
UFW ❌
```

Проблема была в **Security Group облачного сервера**.

---

# 15. Security Group

Security Group — firewall на уровне облачной инфраструктуры.

Изначально было разрешено только:

```text
TCP | 22 | 0.0.0.0/0 | SSH
```

Поэтому SSH работал:

```bash
ssh ansible-user@<SERVER_IP>
```

Но HTTP использует порт:

```text
80
```

а он был закрыт.

Добавил входящее правило:

```text
TCP | 80 | 0.0.0.0/0 | HTTP
```

`0.0.0.0/0` означает — разрешить подключения с любых IPv4-адресов.

---

# 16. UFW и Security Group — разница

```text
Интернет
    ↓
Security Group
    ↓
Облачный сервер
    ↓
UFW / Linux firewall
    ↓
Nginx
```

### Security Group

Работает на уровне облачной инфраструктуры.

Может не пропустить пакет даже до Linux-сервера.

### UFW

Работает уже внутри Linux-сервера.

---

# 17. Финальная проверка

После открытия TCP-порта `80`:

```bash
curl -v http://<SERVER_IP>
```

Получил:

```text
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)

HELLO WORLD
```

`HTTP 200 OK` означает, что запрос успешно обработан сервером.

---

# Итоговая схема

```text
Мой компьютер
      │
      │ HTTP GET
      │ TCP :80
      ▼
   Интернет
      │
      ▼
Публичный IP сервера
      │
      ▼
Security Group
TCP/80 разрешён
      │
      ▼
Ubuntu Server
      │
      ▼
Nginx :80
      │
      ▼
/var/www/html/index.html
      │
      ▼
  HELLO WORLD
```

# Что было сделано

1. Ansible установлен в Python `venv`.
2. Создан отдельный пользователь для Ansible.
3. Создан SSH-ключ `ed25519`.
4. Настроено SSH-подключение к удалённому серверу.
5. Создан Ansible inventory.
6. Создан playbook.
7. Создана роль обновления пакетов.
8. Создана роль установки Nginx.
9. Через Ansible установлен Nginx.
10. Через Ansible скопирован `index.html`.
11. Nginx добавлен в автозапуск и запущен.
12. Создан handler для перезапуска Nginx.
13. Проверена работа Nginx через `curl localhost`.
14. Проверено прослушивание TCP-порта `80`.
15. Проверен UFW.
16. Найдена проблема с Security Group.
17. В Security Group открыт входящий TCP-порт `80`.
18. Страница стала доступна из интернета.
19. Получен успешный ответ `HTTP 200 OK`.