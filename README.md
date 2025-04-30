# Ansible WordPress Deployment
## 構築概要
本プロジェクトでは、Ansibleを用いて2台のサーバー（dbサーバーとwebサーバー）上にWordPress環境を構築しました。  

MariaDBサーバー（Amazon Linux 2023）と、nginx＋PHP環境のWebサーバー（Ubuntu 22.04）を構成し、WordPressの初期セットアップ画面までを表示できるようにします。  

今後参画が予想されるシステムインフラのOS更改プロジェクトを見据え、マルチOS対応を意識し、上記の構成で作成しました。
変数管理や条件分岐、jinja2テンプレートを活用した、柔軟な運用を意識した設計としています。  

詳しい構築手順についてはsetup.mdに記載しております。

## 工夫した点

### セキュリティ面の配慮  
公開リポジトリには実サーバ情報を含めないようなインベントリファイルを使用しました。  


### DB構築における冪等性の確保  
MySQL rootパスワード設定やデータベース作成については、初回のみ実行されるよう条件分岐を追加し、２回目以降の実行時発生していたエラーを防止しました。

## ディレクトリ構成
```bash
コードをコピーする
ansible-wordpress/
├── inventory              # インベントリファイル（ホストグループ管理）
├── host_vars/             # 各ホストごとの変数管理(gitignoreを適用)
│   ├── db.yml
│   └── web.yml
├── group_vars/            # 全ホスト共通の変数管理
│   └── all.yml
├── templates/             # WordPress・nginx設定ファイルのテンプレート
│   ├── wp-config.php.j2
│   └── nginx.conf.j2
├── roles/                 # Ansibleのロール管理
│   ├── db/                # dbサーバー用ロール
│   │   └── tasks/
│   │       └── main.yml
│   └── web/               # webサーバー用ロール
│       └── tasks/
│           └── main.yml
└── site.yml               # Playbook本体（全体制御）
``` 
