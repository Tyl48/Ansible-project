# 🚀 Triển khai WordPress với Ansible

Dự án này tự động cài đặt **WordPress** với **MariaDB** thông qua Ansible, chạy trên mô hình **3 máy ảo**:



---
## 1. Tạo 3 VM để chạy server ( region Singapore )

- **Ansible Manager**: Máy điều khiển Ansible. ( cấu hình E2 medium )
- **Database Server**: Chạy MariaDB/MySQL để lưu dữ liệu. ( small )
- **WordPress Server**: Chạy Apache + PHP + WordPress. ( micro )
- Chọn ssh ở VM manager, cài đặt mọi thứ trên này và connect tới 2 vm kia 

<img width="957" height="311" alt="image" src="https://github.com/user-attachments/assets/12e6a604-a08e-43ed-a4a9-5e83cf1220f1" />

---
## 1.1. Setup khởi đầu
- Cập nhật
```text
sudo apt update -y
```
  
- Tải ansible
 ```text
 sudo apt install -y ansible
```

- Tải tree
``` text
sudo apt install tree -y
```
- Tạo key
``` text
ssh-keygen -t rsa
```

## 📂 Cấu trúc thư mục

```text
└── ansible-project
    ├── ansible.cfg
    ├── group_vars
    │   ├── database.yml
    │   └── wordpress.yml
    ├── inventory
    │   └── hosts
    ├── playbooks
    │   └── site.yml
    └── roles
        ├── database
        │   ├── handlers
        │   │   └── main.yml
        │   └── tasks
        │       └── main.yml
        └── wordpress
            ├── tasks
            │   └── main.yml
            └── templates
                └── wp-config.php.j2

```
## 2. Tạo cây thư mục của ansible 
```text
mkdir -p ansible-project/{group_vars,inventory,playbooks,roles/{database/{handlers,tasks},wordpress/{tasks,templates}}}
```

## 3. Tạo các file bên trong 
```text
touch ansible-project/ansible.cfg \
      ansible-project/group_vars/database.yml \
      ansible-project/group_vars/wordpress.yml \
      ansible-project/inventory/hosts \
      ansible-project/playbooks/site.yml \
      ansible-project/roles/database/handlers/main.yml \
      ansible-project/roles/database/tasks/main.yml \
      ansible-project/roles/wordpress/tasks/main.yml \
      ansible-project/roles/wordpress/templates/wp-config.php.j2
```
## 4. Các file con  
## 4.1. ansible.cfg 
```text
[defaults]
become_ask_pass=True
inventory = inventory/hosts
remote_user = nguyentruongthinh_9876
roles_path = roles
host_key_checking = False
retry_files_enabled = False
```

## 4.2. database.yml 
```text
db_name: wordpress
db_user: wp_user
db_password: wp_password
```

## 4.3. wordpress.yml
```text
db_host: 10.148.0.10
db_name: wordpress
db_user: wp_user
db_password: supersecret123 
```

## 4.4. hosts
```text
[database]
db-server ansible_host=10.148.0.10 ansible_user=nguyentruongthinh_9876 ansible_ssh_private_key_file=~/.ssh/id_rsa ansible_become=yes ansible_become_method=sudo ansible_become_user=root

[wordpress]
wp-server ansible_host=10.148.0.9 ansible_user=nguyentruongthinh_9876 ansible_ssh_private_key_file=~/.ssh/id_rsa ansible_become=yes ansible_become_method=sudo ansible_become_user=root
```

## 4.5. site.yml
```text
- hosts: database
  become: yes
  roles:
    - database

- hosts: wordpress
  become: yes
  vars_files:
    - ../group_vars/wordpress.yml
  roles:
    - wordpress
```

## 4.6. database / handlers / main.yml
```text
- name: restart mysql
  service:
    name: mariadb
    state: restarted
```

## 4.7. datadabase / tasks / main.yml
```text
- name: Cài MariaDB
  apt:
    name: mariadb-server
    state: present
    update_cache: yes

- name: Cài PyMySQL để Ansible quản lý MySQL
  apt:
    name: python3-pymysql
    state: present

- name: Tạo database cho WordPress
  mysql_db:
    name: "{{ db_name }}"
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock

- name: Tạo user cho WordPress
  mysql_user:
    name: "{{ db_user }}"
    password: "{{ db_password }}"
    priv: "{{ db_name }}.*:ALL"
    host: '%'
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock

- name: Cài đặt UFW
  apt:
    name: ufw
    state: present
    update_cache: yes

- name: Cho phép MySQL listen trên mọi IP
  lineinfile:
    path: /etc/mysql/mariadb.conf.d/50-server.cnf
    regexp: '^bind-address'
    line: 'bind-address = 0.0.0.0'
  notify: restart mysql

- name: Mở port 3306 firewall (UFW)
  ufw:
    rule: allow
    port: 3306
    proto: tcp
```

## 4.8. wordpress / tasks / main.yml
```text
- name: Cài Apache, PHP và các package
  apt:
    name:
      - apache2
      - php
      - php-mysql
      - libapache2-mod-php
      - wget
      - unzip
    state: present
    update_cache: yes

- name: Tải WordPress
  get_url:
    url: https://wordpress.org/latest.zip
    dest: /tmp/wordpress.zip

- name: Giải nén WordPress
  unarchive:
    src: /tmp/wordpress.zip
    dest: /var/www/html/
    remote_src: yes

- name: Tạo file cấu hình wp-config.php
  template:
    src: wp-config.php.j2
    dest: /var/www/html/wordpress/wp-config.php

- name: Restart Apache
  service:
    name: apache2
    state: restarted
```

## 4.9. wordpress / templates / wp-config.php.j2
```text
<?php
define('DB_NAME', '{{ db_name }}');
define('DB_USER', '{{ db_user }}');
define('DB_PASSWORD', '{{ db_password }}');
define('DB_HOST', '{{ db_host }}');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');

$table_prefix  = 'wp_';

define('WP_DEBUG', false);

if ( !defined('ABSPATH') )
    define('ABSPATH', dirname(__FILE__) . '/');

require_once(ABSPATH . 'wp-settings.php');
```
## 5. SSH đến 2 vm 
1. cd vào .ssh trong manager, cat file id_rsa.pub, sau đó copy id
2. Vào vm db_server, di chuyển vào thư mục .ssh, tạo 1 file " authorized_keys " rồi dán id đã copy từ manager vào ( wp làm tương tự )
3. check thử xem đã thành công hay chưa :
``` 
ssh -i ~/.ssh/id_rsa user_name@internal_ip_vm
```
## 5.1. Fix lỗi missing passwd
- vào 2 vm, nhập "sudo visudo" rổi thêm dòng này vào cuối

```
username ALL=(ALL) NOPASSWD:ALL
```

```text
cd /etc/sshe
nano sshd_config
```
- tìm dòng password rồi thay no = yes

## 6. Chạy 
```text
ansible-playbook playbooks/site.yml \
  -e "db_name=wordpress db_user=wp_user db_password=supersecret123 db_host=10.148.0.10"
```
hoặc 
```text
ansible-playbook -i inventory playbooks/site.yml
```
