# Ansible Playbook для настройки двух виртуальных машин на Debian

## Описание задания
Я выполнила задачу по автоматизации настройки двух виртуальных машин (VM) с помощью Ansible. Вот основные шаги, которые я реализовала:

1. **Обновление пакетов и установка утилит**: `net-tools`, `curl`, `nginx`.
2. **Замена файла `index.html`**:
   - Для первой машины страница возвращает `Hello World1 !`.
   - Для второй машины — `Hello World2 !`.
3. **Создание пользователей**:
   - На первой машине — пользователь `User1`.
   - На второй машине — пользователь `User2`.
   - Пароли зашифрованы с помощью Ansible Vault.
4. **Настройка брандмауэра (UFW)**:
   - Разрешены только входящие подключения по SSH и HTTP.
   - Остальные подключения запрещены.

## Подготовка окружения

### 1. Настройка виртуальных машин
Я создала две виртуальные машины с Debian и установила на них Ansible:
```bash
sudo apt update
sudo apt install -y ansible
```

### 2. Настройка SSH-доступа
Чтобы Ansible мог подключаться к виртуальным машинам, я настроила SSH:
1. Сгенерировала ключ на своей рабочей машине:
   ```bash
   ssh-keygen
   ```
2. Скопировала ключ на обе виртуальные машины:
   ```bash
   ssh-copy-id <ваш_пользователь>@<IP_ВМ>
   ```

### 3. Создание структуры проекта
Я организовала проект следующим образом:
```plaintext
ansible_project/
├── group_vars/
│   └── all.yml        # Общие переменные
├── hosts              # Инвентарь
├── playbooks/
│   └── site.yml       # Основной playbook
├── templates/
│   ├── index_vm1.html.j2
│   └── index_vm2.html.j2
└── vars/              # Дополнительные переменные
```

## Реализация задания

### 1. Настройка инвентаря
В файле `hosts` я указала IP-адреса виртуальных машин:
```ini
[debian]
vm1 ansible_host=192.168.56.101
vm2 ansible_host=192.168.56.102

[debian:vars]
ansible_user=my_user
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### 2. Определение переменных
В файле `group_vars/all.yml` я создала переменные для пакетов, пользователей и их паролей:
```yaml
packages:
  - net-tools
  - curl
  - nginx

index_pages:
  vm1: "Hello World1 !"
  vm2: "Hello World2 !"

users:
  vm1:
    name: User1
    password: "{{ 'password1' | password_hash('sha512') }}"
  vm2:
    name: User2
    password: "{{ 'password2' | password_hash('sha512') }}"
```

### 3. Создание шаблонов для страниц
В директории `templates/` я создала два шаблона:
- `index_vm1.html.j2`:
  ```html
  <html>
  <body>
      <h1>{{ index_pages.vm1 }}</h1>
  </body>
  </html>
  ```
- `index_vm2.html.j2`:
  ```html
  <html>
  <body>
      <h1>{{ index_pages.vm2 }}</h1>
  </body>
  </html>
  ```

### 4. Написание playbook
Файл `playbooks/site.yml`:
```yaml
- hosts: debian
  become: yes
  vars_files:
    - ../group_vars/all.yml

  tasks:
    - name: Update and install packages
      apt:
        name: "{{ packages }}"
        state: present
        update_cache: yes

    - name: Deploy custom index.html
      template:
        src: "templates/index_{{ inventory_hostname }}.html.j2"
        dest: /var/www/html/index.html
      notify: restart nginx

    - name: Create users with encrypted passwords
      user:
        name: "{{ users[inventory_hostname].name }}"
        password: "{{ users[inventory_hostname].password }}"
        shell: /bin/bash

    - name: Allow SSH and HTTP only
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      with_items:
        - ssh
        - http

    - name: Deny all other incoming traffic
      ufw:
        rule: deny
        from: any
        to: any
        proto: all

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

### 5. Шифрование паролей
Я зашифровала файл с переменными:
```bash
ansible-vault encrypt group_vars/all.yml
```

### 6. Запуск Playbook
Для выполнения настроек я запустила команду:
```bash
ansible-playbook -i hosts playbooks/site.yml --ask-vault-pass
```

## Проверка результата

1. **Утилиты**: Убедилась, что пакеты установлены:
   ```bash
   dpkg -l | grep -E 'net-tools|curl|nginx'
   ```
2. **Страницы**: Проверила содержимое файлов на каждой ВМ:
   ```bash
   curl http://localhost
   ```
3. **Пользователи**: Проверила наличие пользователей:
   ```bash
   id User1  # На VM1
   id User2  # На VM2
   ```
4. **Брандмауэр**: Проверила настройки UFW:
   ```bash
   sudo ufw status
   ```

## Итоги
Эта работа помогла мне:
- Изучить управление конфигурацией с помощью Ansible.
- Освоить создание playbook для различных задач.
- Попрактиковаться в написании handler-ов для управления сервисами.
- Понять, как использовать Ansible Vault для шифрования данных.
