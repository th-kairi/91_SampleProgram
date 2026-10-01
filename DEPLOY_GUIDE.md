# 【完全手順書】GitHub リリース ＆ AWS EC2 デプロイガイド (Django / Flask / Servlet)

本ドキュメントは、**exShop Multi-Framework Suite**（Django版・Flask版・Servlet版）を **GitHub で管理・リリース** し、クラウドの **AWS EC2（Linux）環境へデプロイして本番公開するまでの全手順** をまとめた演習用完全マニュアルです。

---

## 目次
1. [手順書の形式と推奨運用方法](#1-手順書の形式と推奨運用方法)
2. [PART 1: GitHub リポジトリ作成 ＆ Releases 公開](#2-part-1-github-リポジトリ作成--releases-公開)
3. [PART 2: AWS EC2 インスタンスの準備 ＆ 初期設定](#3-part-2-aws-ec2-インスタンスの準備--初期設定)
4. [PART 3: 各システムのデプロイ手順](#4-part-3-各システムのデプロイ手順)
   - [① Django 版のデプロイ (Gunicorn + Systemd)](#①-django-版のデプロイ-gunicorn--systemd)
   - [② Flask 版のデプロイ (Gunicorn + Systemd)](#②-flask-版のデプロイ-gunicorn--systemd)
   - [③ Servlet 版のデプロイ (Tomcat 10 + WAR)](#③-servlet-版のデプロイ-tomcat-10--war)
5. [PART 4: Nginx による 1台のEC2での同時公開設定](#5-part-4-nginx-による-1台のec2での同時公開設定)

---

## 1. 手順書の形式と推奨運用方法

| 形式 | メリット | デメリット | おすすめの用途 |
|---|---|---|---|
| **Markdown (.md)** ★推奨 | GitHub上で最も綺麗に閲覧可能。IDEでも直接見られ、Gitで差分管理できる。Qiita/Zennへの転載もワンクリック。 | ブラウザ単体では raw 表示になる。 | **原本（マスター）としてリポジトリに配置** |
| **単一HTML (.html)** ★推奨 | **全環境（Win/Mac/スマホ）でブラウザさえあれば100%確実に動作**。目次ナビやワンクリックコピー機能が付けられる。 | テキストエディタで見るとタグが多くなる。 | **学生へのオフライン配布・自習用マニュアル** |
| **Qiita / Zenn** | 一般公開しやすく、外部からの認知やポートフォリオになる。 | 機密情報や演習用限定コードを含む場合に注意が必要。 | 一般公開・ノウハウ共有用 |
| **PDF** | 印刷・提出用に向く。 | **コードをコピーした時に改行や記号が崩れてエラーになる**トラブルが多発する。 | 提出書類用（開発演習には不向き） |

> **【結論】**: リポジトリ内には **GitHub標準の `DEPLOY_GUIDE.md`** を配置し、学生配布用として **コードコピーボタン付きの `deploy_guide.html`** を同梱するのがベストです！

---

## 2. PART 1: GitHub リポジトリ作成 ＆ Releases 公開

### Step 1: ローカル Git リポジトリの初期化
ルートディレクトリ（`91_SampleProgram`）で実行します：

```bash
# Git 初期化
git init

# ファイルをステージング (.gitignore により一時ファイルは自動除外)
git add .

# 初回コミット
git commit -m "feat: initial commit of exShop Multi-Framework EC Suite (Django, Flask, Servlet)"
```

### Step 2: GitHub へのプッシュ
1. GitHub にログインし、**「New repository」** から空のリポジトリ（例: `exShop-suite`）を作成。
2. 以下のコマンドでリモート登録＆プッシュ：

```bash
# main ブランチにリネーム
git branch -M main

# リモートURLの登録
git remote add origin https://github.com/<あなたのユーザー名>/exShop-suite.git

# プッシュ
git push -u origin main
```

### Step 3: GitHub Releases による学生向けバイナリ配布
1. バージョンタグを切ってプッシュします：
   ```bash
   git tag -a v1.0.0 -m "Release v1.0.0: exShop Suite"
   git push origin v1.0.0
   ```
2. GitHub リポジトリ画面右側の **「Releases」→「Create a new release」** を開きます。
3. `v1.0.0` タグを選択し、タイトルを入力（例: `v1.0.0 - 初回リリース`）。
4. **「Attach binaries」** エリアに以下の配布ファイルをドラッグ＆ドロップして添付：
   - `01_Django/exShop_project.zip` (Django演習パッケージ)
   - `02_Flask/exShop_flask.zip` (Flask演習パッケージ)
   - `03_Servlet/target/exShop.war` (Servletビルド成果物)
5. **「Publish release」** をクリック。学生は Release ページから一発でダウンロード可能になります。

---

## 3. PART 2: AWS EC2 インスタンスの準備 ＆ 初期設定

### 1. EC2 インスタンスの作成
- **AMI**: Ubuntu Server 22.04 LTS または 24.04 LTS (x86_64)
- **インスタンスタイプ**: `t3.small` または `t3.medium` 推奨 (メモリ 2GB 以上が快適)
- **ストレージ**: 20GB ~ 30GB (gp3)

### 2. セキュリティグループの設定 (インバウンドルール)
| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|---|---|---|---|---|
| SSH | TCP | 22 | 自分のIP | 管理用接続 |
| HTTP | TCP | 80 | 0.0.0.0/0 | Nginx Webアクセス |
| カスタムTCP | TCP | 8000 | 0.0.0.0/0 | Django 直接確認用 (任意) |
| カスタムTCP | TCP | 5000 | 0.0.0.0/0 | Flask 直接確認用 (任意) |
| カスタムTCP | TCP | 8090 | 0.0.0.0/0 | Servlet 直接確認用 (任意) |

### 3. EC2 に SSH 接続し、共通ミドルウェアを一括インストール
```bash
sudo apt update && sudo apt upgrade -y

# 共通ツールのインストール (Git, Nginx, Python3, JDK 17, Tomcat 10)
sudo apt install -y git nginx curl build-essential     python3-pip python3-venv python3-dev libpq-dev     openjdk-17-jdk tomcat10 postgresql postgresql-contrib

# リポジトリのクローン
cd /var/www
sudo git clone https://github.com/<あなたのユーザー名>/exShop-suite.git
sudo chown -R ubuntu:ubuntu /var/www/exShop-suite
cd /var/www/exShop-suite
```

---

## 4. PART 3: 各システムのデプロイ手順

### ① Django 版のデプロイ (Gunicorn + Systemd)

```bash
cd /var/www/exShop-suite/01_Django/exShop

# 1. 仮想環境の作成とライブラリ導入
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

# 2. 静的ファイルの集約 & DBマイグレーション & サンプルデータ投入
python manage.py collectstatic --noinput
python manage.py migrate
python manage.py load_sample_data

# 3. Gunicorn の systemd サービス定義を作成
sudo bash -c 'cat <<EOF > /etc/systemd/system/exshop-django.service
[Unit]
Description=Gunicorn daemon for exShop Django
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/var/www/exShop-suite/01_Django/exShop
ExecStart=/var/www/exShop-suite/01_Django/exShop/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 exShop.wsgi:application

[Install]
WantedBy=multi-user.target
EOF'

# 4. サービスの起動と自動起動登録
sudo systemctl daemon-reload
sudo systemctl start exshop-django
sudo systemctl enable exshop-django
sudo systemctl status exshop-django
```

---

### ② Flask 版のデプロイ (Gunicorn + Systemd)

```bash
cd /var/www/exShop-suite/02_Flask

# 1. 仮想環境の作成とライブラリ導入
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

# 2. サンプルデータの投入 (SQLite初期化)
python seed.py

# 3. Gunicorn の systemd サービス定義を作成
sudo bash -c 'cat <<EOF > /etc/systemd/system/exshop-flask.service
[Unit]
Description=Gunicorn daemon for exShop Flask Nordic
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/var/www/exShop-suite/02_Flask
ExecStart=/var/www/exShop-suite/02_Flask/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:5000 run:app

[Install]
WantedBy=multi-user.target
EOF'

# 4. サービスの起動と自動起動登録
sudo systemctl daemon-reload
sudo systemctl start exshop-flask
sudo systemctl enable exshop-flask
sudo systemctl status exshop-flask
```

---

### ③ Servlet 版のデプロイ (Tomcat 10 + WAR)

```bash
cd /var/www/exShop-suite/03_Servlet

# 1. PostgreSQL データベースの作成と初期化 (ローカルまたはAWS RDS)
sudo -u postgres psql -c "CREATE USER postgres WITH PASSWORD 'postgres';"
sudo -u postgres psql -c "CREATE DATABASE exshop_db OWNER postgres;"
PGPASSWORD=postgres psql -U postgres -h localhost -d exshop_db -f sql/schema.sql
PGPASSWORD=postgres psql -U postgres -h localhost -d exshop_db -f sql/data.sql

# 2. Maven による WAR パッケージのビルド
mvn clean package -DskipTests
# ※ mvn コマンドがない場合は: ./.tools/maven/apache-maven-3.9.9/bin/mvn clean package -DskipTests

# 3. Tomcat 10 への WAR ファイル配置
# /var/lib/tomcat10/webapps/exShop.war として配置 (http://<IP>/exShop/ でアクセス可能に)
sudo cp target/exShop.war /var/lib/tomcat10/webapps/exShop.war
sudo chown -R tomcat:tomcat /var/lib/tomcat10/webapps/exShop.war

# 4. Tomcat 10 の再起動
sudo systemctl restart tomcat10
sudo systemctl status tomcat10
```

---

## 5. PART 4: Nginx による 1台のEC2での同時公開設定

1台の EC2 インスタンス（ポート80）で、3つのシステムすべてにルーティングする設定例です。

### `/etc/nginx/sites-available/default` を編集:
```bash
sudo bash -c 'cat <<EOF > /etc/nginx/sites-available/default
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    client_max_body_size 50M;

    # 1. Django 版へのルーティング (ポート8000または /django/ 等)
    location /django/ {
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }

    # 2. Flask 版へのルーティング (ポート5000または /flask/ 等)
    location /flask/ {
        proxy_pass http://127.0.0.1:5000/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }

    # 3. Servlet 版へのルーティング (Tomcat /exShop/)
    location /servlet/ {
        proxy_pass http://127.0.0.1:8080/exShop/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }

    # ルートアクセス時のデフォルト（Django版へ転送）
    location / {
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF'

# Nginx 設定テストと再読み込み
sudo nginx -t
sudo systemctl reload nginx
```

ブラウザで `http://<EC2のパブリックIP>/` にアクセスすると、Nginx を経由して各システムへ快適にアクセスできます！
