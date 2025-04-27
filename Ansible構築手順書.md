# Ansibleを使ったWordPress環境構築手順書
## 前提条件
- Ansibleがインストールされたローカルマシンまたは管理サーバーを使用すること

- AWS上にOSの異なる管理対象のサーバー2台が用意されていること
  - dbサーバー（例：Amazon Linux 2023）
  - webサーバー（例：Ubuntu 22.04）

- 各サーバーともに外部IPアドレス、内部IPアドレスが分かっていること

- それぞれのサーバーの必要なセキュリティグループが解放されていること

- SSH秘密鍵（pemファイル等）がローカルマシンに配置されていること

## 手順
### 1. ディレクトリ構成の作成
```bash
mkdir -p ~/ansible-wordpress
cd ~/ansible-wordpress
mkdir -p host_vars group_vars
```

### 2. インベントリファイルの作成
```bash
vi inventory
```
```ini
[db]
db
# host_vars/db.yml

[web]
web
# host_vars/web.yml
```

### 3. ホスト変数ファイルの作成
```bash
vi group_vars/db.yml
```
dbサーバー用の設定値
```yml
ansible_host: dbサーバーの外部IPアドレス  
ansible_user: dbサーバーのホスト名
ansible_ssh_private_key_file: dbサーバーのssh認証鍵のパス
```

webサーバー用の設定値
```bash
vi group_vars/web.yml
```
```yml
ansible_host: webサーバーの外部IPアドレス  
ansible_user: webサーバーのホスト名
ansible_ssh_private_key_file: webサーバーのssh認証鍵のパス
```

### 4. グループ変数ファイルの作成
```bash
vi group_vars/all.yml
```
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
### 5. 接続確認
以下のコマンドでAnsibleからサーバーへの接続確認　　
```shell
ansible -i inventory all -m ping
```
で疎通確認する。下記のように表示されれば問題なし　

```shell
db | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.9"
    },
    "changed": false,
    "ping": "pong"
}
web | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
### 6. playbookに必要なディレクトリの作成
```bash
mkdir -p roles/db/tasks roles/web/tasks templates
```
### 7. テンプレートファイルの作成
WordPress設定用ファイルのテンプレートを作成
```bash
vi templates/wp-config.php.j2
```
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
nginx設定用ファイルのテンプレートを作成
```bash
vi templates/nginx.conf.j2
```

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

### 8. ロールの作成
dbサーバー用ロールの作成
```bash
vi roles/db/tasks/main.yml
```

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

# --- ここから条件分岐 ---

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
webサーバー用ロール
```bash
vi roles/web/tasks/main.yml
```

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
### 9. site.ymlの作成
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
### 10. Playbookの実行
以下のコマンドでPlaybookを実行する。
```bash
ansible-playbook -i inventory site.yml
```

### 11. ブラウザ確認
webサーバーの外部IPアドレスにブラウザからアクセスする  

http://webサーバーのIPアドレス  

WordpressのWelcomeページが表示されれば構築成功

