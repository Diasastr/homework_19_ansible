Налаштування Ansible для Веб-сервера та Сервера Бази Даних
==========================================================

Цей репозиторій містить Ansible playbooks та ролі для налаштування веб-сервера з Nginx та сервера бази даних з MariaDB. Конфігурація включає створення бази даних `myapp` та користувача `myappuser` для зовнішніх підключень до бази даних.

Огляд
-----
Для початку встановили ансібл на control-node, створили файл ansible/hosts і сконфігурували доступ до target-node використовуючи приватну адресу і path to pom.file. Додатково створили файл vault_db.yml (вас спитаюсь вигадати пароль до файлу, запам'ятайте його!):
```bash
ansible-vault create vault_db.yml
```
Потім додали потрібні паролі, нам треба як мінімум ці змінні:
````yml
vault_mariadb_root_password: mySecureRootPassword123!
vault_myapp_db_password: anotherSecurePassword456!
````
Якщо треба змінити щось:
```bash
ansible-vault edit vault_db.yml
```
Тепер, коли ранимо playbook database_setup.yml, нам треба вказати додатковий флаг і ввести пароль від файлу з паролями :)
```bash
ansible-playbook your_playbook.yml --ask-vault-pass
```


Налаштування включає наступні компоненти:

-   Веб-Сервер: Налаштований з Nginx.
   
   ```yml
       ---
    # Install the Nginx web server
    - name: Install Nginx
      ansible.builtin.yum:
        name: nginx
        state: present
    
    # Copy the Nginx configuration template to the server
    # Note: we need to create the 'nginx.conf.j2' Jinja2 template file in the 'templates' directory of this role and the headers, the structure shown later in the file
    - name: Copy Nginx configuration template
      ansible.builtin.template:
        src: "templates/nginx.conf.j2"
        dest: "/etc/nginx/nginx.conf"
      notify: restart nginx
    
    # Ensure the Nginx service is running and enabled to start on boot
    - name: Ensure Nginx is running and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes
   ```
-   Сервер Бази Даних: Налаштований з MariaDB. Включає базу даних `myapp` та користувача `myappuser` з повними привілеями на цій базі даних. Виеористовую баш команди бо лінилася завантажити модулі, а ще неможливо завантажити саме mysql, бо кидає 403 forbidden. 
  ```yml
   ---
  # Install MariaDB server
  - name: Install MariaDB Server
    ansible.builtin.yum:
      name: mariadb105-server
      state: present
    become: yes
  
  # Ensure MariaDB is running and enabled on boot
  - name: Start and enable MariaDB
    ansible.builtin.service:
      name: mariadb
      state: started
      enabled: yes
    become: yes
  
  # Secure the MariaDB installation (manually)
  - name: Set root password
    ansible.builtin.shell:
      cmd: mysqladmin -u root password 'your_strong_root_password'
    ignore_errors: yes
    become: yes
  
  - name: Remove anonymous users
    ansible.builtin.command:
      cmd: "mysql -u root -p'your_strong_root_password' -e \"DELETE FROM mysql.user WHERE User=''; FLUSH PRIVILEGES;\""
    become: yes
  
  # Flush privileges
  - name: Flush privileges
    ansible.builtin.shell:
      cmd: mysql -u root -p'your_strong_root_password' -e 'FLUSH PRIVILEGES;'
    become: yes
  
  # Create a new database named 'myapp'
  - name: Create a database named 'myapp'
    ansible.builtin.shell:
      cmd: "mysql -u root -p'your_strong_root_password' -e \"CREATE DATABASE IF NOT EXISTS myapp;\""
    become: yes
  
  # Add a user for external connections
  - name: Add a MariaDB user for external connections
    ansible.builtin.shell:
      cmd: "mysql -u root -p'your_strong_root_password' -e \"CREATE USER 'myappuser'@'%' IDENTIFIED BY 'user_strong_password'; GRANT ALL PRIVILEGES ON myapp.* TO 'myappuser'@'%'; FLUSH PRIVILEGES;\""
    become: yes
 ```
-   Конфігурація юзера:
  ```yml
    ---
    - name: Ensure group 'itdep' exists
    ansible.builtin.group:
      name: itdep
      state: present
    
    - name: Create user 'devops'
    ansible.builtin.user:
      name: devops
      group: itdep
      create_home: yes
    
    - name: Set up .bashrc for user 'devops'
    ansible.builtin.copy:
      dest: "/home/devops/.bashrc"
      content: |
        # Custom .bashrc content
        export PS1="\[\e[0;33m\][\u@\h \W]\$\[\e[0m\] "  # Change command prompt color to yellow
      owner: devops
      mode: '0644'
  ```
-   Апдейти софта:
  ```yml
  ---
  - name: Install utility packages
    ansible.builtin.yum:
      name:
        - mc
        - screen
        - tree
      state: present
  ```


Передумови
----------

-   EC2 інстанс Amazon Linux 2023 (або схоже середовище).
-   Ansible встановлено на керуючому вузлі (може бути вашим локальним комп'ютером або окремим EC2 інстансом).
-   Python встановлено на керуючих та цільових вузлах.

Ролі та Завдання
----------------

Налаштування Ansible поділене на ролі, кожна з яких відповідає за певну частину конфігурації.

### Роль Веб-Сервера (`web_server`)

-   Встановлює Nginx.
-   Забезпечує запуск та включення Nginx при завантаженні системи.

### Роль Сервера Бази Даних (`mysql_server`)

-   Встановлює сервер MariaDB (`mariadb105-server`).
-   Запускає та включає MariaDB при завантаженні системи.
-   Встановлює пароль root для MariaDB.
-   Видаляє анонімних користувачів 
-   Створює базу даних `myapp`.
-   Додає користувача `myappuser` з повними привілеями на базу даних `myapp`. Цей користувач може підключатися з будь-якого хосту.

Виконання Playbooks
-------------------
Наприклад, один з файлів виглядає так:

```yml
---
- name: Setup Everything
  hosts: target
  become: yes
  roles:
    - user_config
    - install_soft
    - web_server
    - mysql_server
```

Для запуску playbooks перейдіть до кореневої директорії цього репозиторію та виконайте:

bashCopy code

`ansible-playbook -i hosts playbooks/setup_everything.yml`

Ця команда запускає playbook `setup_everything.yml`, який включає всі необхідні ролі та завдання для налаштування.

Питання Безпеки
---------------

-   Паролі жорстко закодовані в playbooks для демонстрації. У виробничому середовищі використовуйте Ansible Vault для безпечного управління конфіденційною інформацією.
-   Користувач бази даних `myappuser` налаштований на підключення з будь-якого хосту (`'%'`). Для більшої безпеки обмежте це специфічними хостами або мережами.

Налаштування
------------

Ми можемо налаштувати playbooks та ролі відповідно до  потреб. Playbooks розроблені таким чином, щоб бути модульними, що дозволяє легко вносити зміни та розширення.

Структура Репозиторію
---------------------

-   `playbooks/`: Містить Ansible playbooks.
-   `roles/`: Містить ролі для налаштування веб- та серверів баз даних.
-   `hosts`: Файл інвентаризації для Ansible.

```bash
/home/ec2-user/ansible/
├── hosts
├── playbooks/
│   ├── setup_everything.yml
│   ├── setup_webserver.yml
│   └── setup_databse.yml
└── roles/
    ├── install_soft/
    │   └── tasks/
    │       └── main.yml
    ├── mysql_server/
    │   └── tasks/
    │       └── main.yml
    ├── user_config/
    │   └── tasks/
    │       └── main.yml
    └── web_server/
        ├── handlers/
        ├── tasks/
        │   └── main.yml
        └── templates/ 

```

Замітка про Шаблони та Файли Конфігурації
-----------------------------------------

-   У репозиторії використовуються шаблони для конфігурації, наприклад, для Nginx. Це дає можливість гнучко налаштовувати конфігурацію серверів.
-   Файли конфігурації та шаблони знаходяться в відповідних директоріях у межах ролей.
