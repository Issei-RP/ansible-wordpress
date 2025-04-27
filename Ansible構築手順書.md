mkdir -p ~/ansible-wordpress
cd ~/ansible-wordpress

vi inventory
```ini
[db]
db
# host_vars/db.yml

[web]
web
# host_vars/web.yml
```
mkdir -p host_vars
vi group_vars/db.yml
```yml
ansible_host: dbサーバーの外部IPアドレス  
ansible_user: dbサーバーのホスト名
ansible_ssh_private_key_file: dbサーバーのssh認証鍵のパス
```

vi group_vars/web.yml
```yml
ansible_host: webサーバーの外部IPアドレス  
ansible_user: webサーバーのホスト名
ansible_ssh_private_key_file: webサーバーのssh認証鍵のパス
```

mkdir -p group_vars
vi group_vars/all.yml
```yml
wordpress_db_name: wordpress
wordpress_db_user: root
wordpress_db_password: rootpassword
wordpress_site_name: wordpress

db_pip: dbサーバーの内部IPアドレス # 内部接続用のプライベートIP

ubuntu_packages:
  - nginx
  - php-fpm
  - php-mysql
  - php-curl
  - php-gd
  - php-xml
  - php-mbstring
```

```shell
ansible -i inventory all -m ping
```
で疎通確認する。下記のように表示されれば問題なし。  

```shell
amazonlinux | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.9"
    },
    "changed": false,
    "ping": "pong"
}
ubuntu | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

mkdir -p roles/ubuntu/tasks roles/amazonlinux/tasks templates

vi templates/wp-config.php.j2

```php
<?php
define('DB_NAME', '{{ wordpress_db_name }}');
define('DB_USER', '{{ wordpress_db_user }}');
define('DB_PASSWORD', '{{ wordpress_db_password }}');
define('DB_HOST', '{{ db_pip }}');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');
$table_prefix = 'wp_';
define('WP_DEBUG', false);
if ( !defined('ABSPATH') )
    define('ABSPATH', dirname(__FILE__) . '/');
require_once(ABSPATH . 'wp-settings.php');
?>
```
vi templates/nginx.conf.j2

```conf
server {
    listen 80;

    root /var/www/html/{{ wordpress_site_name }};
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php-fpm.sock;
    }
}
```
vi roles/db/tasks/main.yml

```yml
- name: Install MariaDB server and client
  ansible.builtin.dnf:
    name:
      - mariadb105-server
      - mariadb105
    state: present
  become: yes

- name: Start and enable MariaDB service
  ansible.builtin.service:
    name: mariadb
    state: started
    enabled: true
  become: yes

- name: Install pip3
  ansible.builtin.dnf:
    name: python3-pip
    state: present
  become: yes

- name: Install PyMySQL via pip
  ansible.builtin.pip:
    name: PyMySQL
    executable: pip3
  become: yes

# --- ここから条件分岐追加 ---

- name: Check if MySQL root password is already set
  ansible.builtin.stat:
    path: /root/.mysql_password_set
  register: mysql_password_already_set
  become: yes

- name: Allow root user to use password authentication (disable unix_socket plugin)
  community.mysql.mysql_user:
    name: root
    host: localhost
    plugin: mysql_native_password
    password: "{{ wordpress_db_password }}"
    login_unix_socket: /var/lib/mysql/mysql.sock
    check_implicit_admin: true
    state: present
  when: not mysql_password_already_set.stat.exists
  become: yes

- name: Create a marker file after setting root password
  ansible.builtin.file:
    path: /root/.mysql_password_set
    state: touch
  when: not mysql_password_already_set.stat.exists
  become: yes

# --- ここまでが初回だけ実行される ---

- name: Create WordPress database
  community.mysql.mysql_db:
    name: "{{ wordpress_db_name }}"
    state: present
    login_user: root
    login_password: "{{ wordpress_db_password }}"
  become: yes

- name: Create WordPress database user
  community.mysql.mysql_user:
    name: "{{ wordpress_db_user }}"
    password: "{{ wordpress_db_password }}"
    priv: "{{ wordpress_db_name }}.*:ALL"
    host: "%"
    state: present
    login_user: root
    login_password: "{{ wordpress_db_password }}"
  become: yes

```
vi roles/web/tasks/main.yml

```yml
- name: Install packages
  apt:
    name: "{{ ubuntu_packages }}"
    state: present
    update_cache: yes

- name: Download WordPress (tar.gz version)
  get_url:
    url: https://wordpress.org/latest.tar.gz
    dest: /tmp/wordpress.tar.gz

- name: Extract WordPress
  unarchive:
    src: /tmp/wordpress.tar.gz
    dest: /var/www/html/
    remote_src: yes

- name: Apply WordPress config
  template:
    src: ../../templates/wp-config.php.j2
    dest: /var/www/html/wordpress/wp-config.php

- name: Apply nginx config
  template:
    src: ../../templates/nginx.conf.j2
    dest: /etc/nginx/sites-available/default

- name: Reload nginx
  service:
    name: nginx
    state: restarted
```
vi site.yml
```yml
---
- hosts: db
  become: yes
  roles:
    - db

- hosts: web
  become: yes
  roles:
    - web
```

ansible-playbook -i inventory site.yml

ブラウザを開き、http://webサーバーの外部IPアドレスで検索する。  
WordpressのWelcomeページが表示されれば構築完了

