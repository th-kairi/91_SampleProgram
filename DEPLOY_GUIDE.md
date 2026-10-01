# Webアプリケーション公開実践マニュアル（AWS EC2 / Amazon Linux 2023編）

このマニュアルでは、手元のパソコンで作ったWebアプリケーション（Django / Flask / Servlet）を、世界中のインターネットから誰でも見られるように「クラウドサーバー（AWS EC2: Amazon Linux 2023）」へ設置して公開する手順を、ゼロから順番に解説します。

---

## 全体の流れ（何をするのか？）

私たちが普段使っているパソコンの電源を切ると、その中で動いているアプリも見られなくなってしまいます。
そこで、**24時間365日ずっとインターネットに繋がり続けている専用コンピューター（AWS EC2サーバー）** を1台借りて、そこにプログラムを設置します。

```
[ あなたのパソコン ]
       │
       ▼ (1) プログラムをGitHubに送る (完了済み)
  [ GitHub (th-kairi/91_SampleProgram) ]
       │
       ▼ (2) サーバーがGitHubからダウンロード（またはTera Termで転送）
[ AWS EC2 サーバー (Amazon Linux 2023 / 無料利用枠) ]
       │
       ├─ 受付係（Nginx）：インターネットからのアクセスを整理する
       ├─ アプリ本体（Django / Flask / Servlet）：画面や機能を動かす
       └─ データベース（PostgreSQL / SQLite）：データを安全に保管する
```

作業は以下の **5つのステップ** で進めます。

1. **AWSでサーバー（Amazon Linux 2023）を作成する**
2. **手元のパソコンからサーバーにログイン（Tera Term または PowerShell）する**
3. **サーバーに必要な道具（Python, Java 17, Tomcat 10, Nginxなど）を準備する**
4. **アプリをサーバー上で起動する（常駐化）**
5. **受付係（Nginx）を設定して世界中に公開する**

---

## ステップ 1: AWSでサーバー（EC2）を作ろう

AWS（Amazon Web Services）の中に、**Amazon Linux 2023** が入った仮想コンピューターを立ち上げます。

### 1-1. AWSマネジメントコンソールにログイン
1. ブラウザでAWSにサインインします。
2. 画面右上の地域（リージョン）が **「東京（ap-northeast-1）」** になっていることを確認してください。

### 1-2. EC2の画面を開く
1. 画面上部の検索バーに `EC2` と入力し、表示された **「EC2」** をクリックします。
2. 左メニューの「インスタンス」をクリックし、オレンジ色の **「インスタンスを起動」** ボタンをクリックします。

### 1-3. サーバーの設定を決める
画面の項目を上から順に設定していきます。

| 項目名 | 設定する値 | 解説 |
| :--- | :--- | :--- |
| **名前とタグ** | `my-web-server` など自由 | サーバーの識別名です。 |
| **OSイメージ (AMI)** | **Amazon Linux** を選択<br>（AMI: **Amazon Linux 2023 AMI**） | AWS公式の高速・安全なLinuxOSです。無料利用枠の対象です。 |
| **アーキテクチャ** | `64 ビット (x86)` | 標準のCPU形式です。 |
| **インスタンスタイプ** | **`t2.micro`** または **`t3.micro`** | 「無料利用枠の対象」と緑色で書かれているものを選びます。 |

### 1-4. キーペア（サーバーの鍵）を作る
サーバーにログインするための「秘密鍵」を発行します。**この鍵を紛失すると二度とサーバーに入れなくなるため大切に保管してください。**

1. **「キーペア (ログイン)」** の項目で、右側の **「新しいキーペアの作成」** をクリックします。
2. **キーペア名**: `my-key` など半角英数で入力します。
3. **キーペアのタイプ**: `RSA` を選択します。
4. **プライベートキーファイル形式**:
   - **`.pem`** を選択します（Tera Term も PowerShell も両方 `.pem` で接続できます）。
5. **「キーペアを作成」** をクリックすると、パソコンに `my-key.pem` がダウンロードされます。
   - ダウンロードしたファイルは、消えない安全なフォルダ（例: `C:\Users\ユーザー名\.ssh\` やデスクトップ）に保存してください。

### 1-5. ネットワーク設定（ファイアウォールの設定）
インターネットからサーバーへの「通信の通り道」を開けます。

1. **「ネットワーク設定」** の右上の **「編集」** をクリックします。
2. **ファイアウォール (セキュリティグループ)** で **「セキュリティグループを作成」** を選択します。
3. 以下の **3つのルール** が揃っていることを確認します。

| 種類 | ポート範囲 | ソース（接続元） | 役割 |
| :--- | :--- | :--- | :--- |
| **SSH** | `22` | **自分のIP**（推奨）または `任意の場所 (0.0.0.0/0)` | 自分のパソコンからサーバーを遠隔操作するための通り道 |
| **HTTP** | `80` | **任意の場所 (0.0.0.0/0)** | 世界中の人がWebブラウザでホームページを見るための通り道 |
| **HTTPS** | `443` | **任意の場所 (0.0.0.0/0)** | 暗号化された安全なWeb通信のための通り道 |

> [!WARNING]
> HTTP（ポート80）が開いていないと、あとでアプリを動かしてもブラウザからサイトを開くことができません。必ず「インターネットからの HTTP トラフィックを許可する」にチェックを入れてください。

### 1-6. 起動する
画面右下のオレンジ色の **「インスタンスを起動」** ボタンをクリックします。約1〜2分でステータスが **「実行中」** に変わります。

---

## ステップ 2: 自分のパソコンからサーバーにログインしよう

お使いのツールに合わせて、**【方法A: Tera Term】** または **【方法B: PowerShell】** のどちらか好きな方を選んでログインします。

まずはEC2の管理画面で、作成したインスタンスの **「パブリック IPv4 アドレス」**（例: `54.238.123.45`）をコピーしておきます。

---

### 【方法 A】Tera Term でログインする場合

1. **Tera Term** を起動します。
2. **「新しい接続」** ウィンドウが開くので、以下を入力して「OK」をクリックします：
   - **ホスト**: コピーしたパブリック IPv4 アドレス（例: `54.238.123.45`）
   - **TCPポート**: `22`
   - **サービス**: `SSH`
3. 初回接続時、「セキュリティ警告」が出たら **「続行」** をクリックします。
4. **「SSH認証」** ウィンドウで、以下を入力します：
   - **ユーザ名**: **`ec2-user`** （※Amazon Linux 2023の標準ユーザー名です）
   - **パスフレーズ**: （空欄のままでOK）
   - **「RSA/DSA/ECDSA/ED25519鍵を使う」** にチェックを入れ、**「秘密鍵」** ボタンをクリックして、ステップ1でダウンロードした **`my-key.pem`** を選択します。
     （※ファイル選択ダイアログの右下で「すべてのファイル (*.*)」を選ぶと `.pem` が見つかります）
5. **「OK」** をクリックします。
6. 画面に `[ec2-user@ip-172-31-xx-xx ~]$` と表示されれば、**ログイン大成功** です！

---

### 【方法 B】PowerShell（またはMacターミナル）でログインする場合

1. パソコンで **PowerShell** を開きます。
2. 鍵（`my-key.pem`）を保存したフォルダに移動します（例: ダウンロードフォルダの場合）：
   ```powershell
   cd C:\Users\admin\Downloads
   ```
3. 以下のSSHコマンドを実行します（※IPと鍵ファイル名はご自身のものに置き換えてください）：
   ```powershell
   ssh -i my-key.pem ec2-user@54.238.123.45
   ```
4. 初回のみ「接続を続けますか？ (yes/no)」と聞かれるので、**`yes`** と入力して Enter を押します。
5. 画面の左側の表示が `[ec2-user@ip-172-31-xx-xx ~]$` に変われば、**ログイン大成功** です！

> これ以降の作業は、すべてこのサーバーの画面の中で行います。

---

## ステップ 3: サーバーに必要な道具を入れよう

生まれたてのサーバーには、PythonもJavaもデータベースも入っていません。アプリを動かすための道具を一括でインストールします。

### 3-1. サーバーの更新とツールのインストール
以下のコマンドをコピーして貼り付け、Enter を押してください。

```bash
sudo dnf update -y
sudo dnf install -y git python3-pip python3-devel openjdk-17-jdk nginx postgresql16-server postgresql16 wget unzip
```
> **解説**:
> Amazon Linux 2023 では `dnf` というパッケージ管理ツールを使います。Python、Java 17、Webサーバー（Nginx）、データベース（PostgreSQL 16）などをまとめて導入しています。
> （※1〜2分かかります。プロンプト `[ec2-user@... ~]$` に戻るまでお待ちください）

### 3-2. PostgreSQL（データベース）の初期化と起動
```bash
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql
```

### 3-3. プログラムをサーバーに配置する（2つの方法）

プログラムをサーバーに配置します。**推奨の「GitHubからクローン」** または **Tera Termの「ドラッグ＆ドロップ（SCP）」** のどちらかを選びます。

#### 【方法 1（推奨）】GitHubから直接クローンする
```bash
cd ~
git clone https://github.com/th-kairi/91_SampleProgram.git exshop
```
これだけで、ホームディレクトリに `exshop` フォルダが作成され、すべてのコードが綺麗に配置されます！

#### 【方法 2】Tera Term の SCP 転送を使う場合
手元のパソコンにあるファイル（またはZIP）を直接送りたい場合：
1. Tera Term の黒い画面に、パソコン上の ZIP ファイル（例: `exShop_project.zip` や `exShop.war`）を **そのままドラッグ＆ドロップ** します。
2. ポップアップが出るので **「SCP」** をクリックすると、サーバーのホームディレクトリ（`~/`）にファイルが転送されます。

---

## ステップ 4: アプリケーションを動かそう（エディション別）

動かしたいエディションの手順を選んで進めてください。（ポートが分かれているため、**3つすべてを同時に動かすことも可能** です）

---

### コース A: Django版を動かす（ポート 8000）

#### 1. 仮想環境の作成とライブラリ導入
```bash
cd ~/exshop/01_Django
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt gunicorn
python manage.py migrate
python manage.py collectstatic --noinput
```

#### 2. 24時間常駐サービスとして登録して起動（systemd）
ターミナルを閉じてもアプリが動き続けるように設定します。以下のコマンドをそのまま貼り付けて Enter を押します。

```bash
sudo bash -c 'cat <<EOF > /etc/systemd/system/django_exshop.service
[Unit]
Description=Gunicorn instance for Django exShop
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/exshop/01_Django
Environment="PATH=/home/ec2-user/exshop/01_Django/venv/bin"
ExecStart=/home/ec2-user/exshop/01_Django/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 exShop.wsgi:application

[Install]
WantedBy=multi-user.target
EOF'

sudo systemctl daemon-reload
sudo systemctl enable --now django_exshop
```

状態を確認します：
```bash
sudo systemctl status django_exshop
```
緑色で `active (running)` と表示されていれば成功です！（キーボードの `q` で元の画面に戻ります）

---

### コース B: Flask版を動かす（ポート 5000）

```bash
cd ~/exshop/02_Flask
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt gunicorn

sudo bash -c 'cat <<EOF > /etc/systemd/system/flask_exshop.service
[Unit]
Description=Gunicorn instance for Flask exShop Nordic
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/exshop/02_Flask
Environment="PATH=/home/ec2-user/exshop/02_Flask/venv/bin"
ExecStart=/home/ec2-user/exshop/02_Flask/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:5000 run:app

[Install]
WantedBy=multi-user.target
EOF'

sudo systemctl daemon-reload
sudo systemctl enable --now flask_exshop
```

状態を確認します：
```bash
sudo systemctl status flask_exshop
```
緑色で `active (running)` となっていれば成功です！（`q` で戻ります）

---

### コース C: Servlet / JSP版を動かす（Tomcat 10 + PostgreSQL）

> [!IMPORTANT]
> 今回作成したServlet版は最新の **Jakarta EE 10 / Servlet 6.0** 仕様のため、古い Tomcat 9 ではなく **Tomcat 10** を使用します。

#### 1. Tomcat 10 のインストール
```bash
cd /opt
sudo wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.30/bin/apache-tomcat-10.1.30.tar.gz
sudo tar xzf apache-tomcat-10.1.30.tar.gz
sudo mv apache-tomcat-10.1.30 tomcat10
sudo chown -R ec2-user:ec2-user /opt/tomcat10
chmod +x /opt/tomcat10/bin/*.sh
```

#### 2. PostgreSQL にデータベースと商品データを準備
```bash
sudo -u postgres psql -c "CREATE USER exshop WITH PASSWORD 'exshop_pass';"
sudo -u postgres psql -c "CREATE DATABASE exshop_db OWNER exshop;"

cd ~/exshop/03_Servlet
PGPASSWORD='exshop_pass' psql -U exshop -d exshop_db -h localhost -f sql/schema.sql
PGPASSWORD='exshop_pass' psql -U exshop -d exshop_db -h localhost -f sql/data.sql
```

#### 3. WARファイルをTomcatに配置して起動
```bash
sudo cp ~/exshop/03_Servlet/target/exShop.war /opt/tomcat10/webapps/ROOT.war
/opt/tomcat10/bin/startup.sh
```
> **ポイント**: `ROOT.war` という名前で配置すると、URLに余計なパスを付けずにトップページとして直接開けるようになります。

---

## ステップ 5: 受付係（Nginx）を設定して世界に公開しよう！

外部からのインターネットアクセス（ポート80）を受け取り、各アプリへ案内する受付係（Nginx）を設定します。

### 5-1. Nginxの設定ファイルを作成
以下のコマンドをそのまま貼り付けて Enter を押します。

```bash
sudo bash -c 'cat <<EOF > /etc/nginx/conf.d/exshop.conf
server {
    listen 80 default_server;
    server_name _;

    # 静的ファイルの高速配信設定
    location /static/ {
        alias /home/ec2-user/exshop/01_Django/static/;
    }
    location /media/ {
        alias /home/ec2-user/exshop/01_Django/media/;
    }

    # 通常アクセス (/) は Django へ案内
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }

    # 「/flask/」で始まるアクセスは Flask へ案内
    location /flask/ {
        proxy_pass http://127.0.0.1:5000/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }

    # 「/servlet/」で始まるアクセスは Tomcat (Servlet) へ案内
    location /servlet/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF'
```

### 5-2. ディレクトリ権限の付与とNginxの起動
Nginxが静的ファイルにアクセスできるように権限を付与し、起動します：

```bash
chmod 711 /home/ec2-user
sudo nginx -t
sudo systemctl enable --now nginx
sudo systemctl restart nginx
```
画面に `syntax is ok` および `test is successful` と出れば完璧です！

---

## ステップ 6: ブラウザで実際に見てみよう！

これでインターネットへの本番公開が完了しました！
PCやスマートフォンのブラウザを開いて、以下のURLにアクセスしてみましょう。

* **Django版（ストリート系デザイン）を見たいとき**:
  `http://<あなたのパブリックIP>/`
  （例: `http://54.238.123.45/`）

* **Flask版（北欧ナチュラル系デザイン）を見たいとき**:
  `http://<あなたのパブリックIP>/flask/`
  （例: `http://54.238.123.45/flask/`）

* **Servlet版（ラグジュアリー・シック系デザイン）を見たいとき**:
  `http://<あなたのパブリックIP>/servlet/`
  （例: `http://54.238.123.45/servlet/`）

スマートフォンからでも、友人のパソコンからでも、URLを入力すれば自分が立ち上げた本格ECサイトが世界中どこからでも閲覧・注文できます！

---

## 困ったときのトラブルシューティング

### Q1. ブラウザで開いても「応答時間が長すぎます」「接続できません」となる
* **原因**: AWSのセキュリティグループで、HTTP（ポート80）の通信がブロックされています。
* **解決策**: AWSマネジメントコンソールの「EC2」→「セキュリティグループ」を開き、「インバウンドルール」に「HTTP / ポート80 / 0.0.0.0/0」が許可されているか確認してください。

### Q2. 「502 Bad Gateway」というエラー画面が出る
* **原因**: 受付係のNginxは動いていますが、その奥にいるアプリ（DjangoやFlaskなど）が止まっています。
* **解決策**: サーバーのターミナルで `sudo systemctl status django_exshop` または `sudo systemctl status flask_exshop` を実行し、赤文字でエラーが出ていないか確認してください。停止している場合は `sudo systemctl restart django_exshop` を試します。

### Q3. サイトは開くが、画像やデザイン（CSS）が崩れて文字だけになる
* **原因**: Nginxがホームディレクトリ内の静的ファイルにアクセスできていません。
* **解決策**: `chmod 711 /home/ec2-user` を実行してください。
