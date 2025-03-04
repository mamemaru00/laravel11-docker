# laraverl11 + doker環境構築

MySQL + phpMyAdmin 環境構築

---

## **フォルダ構成**

プロジェクトのフォルダ構成は以下のようになります：

```
Laravel11 (プロジェクト名は任意)
├── infra/
│   ├── mysql/
│   │   ├── Dockerfile
│   │   └── my.cnf
│   ├── nginx/
│   │   └── default.conf
│   └── php/
│       ├── Dockerfile
│       └── php.ini
├── docker-compose.yml
├── src/
│   └── Laravel本体のディレクトリ
└── README.md
```

---

## **環境構築の手順**

### **1. 基本設定**

#### docker-compose.ymlの作成

プロジェクトのルートディレクトリで以下のファイルを作成します：  
`touch docker-compose.yml`

以下の内容を記述します：

```yaml
version: "3.9"
services:
  app:
    build: ./infra/php
    volumes:
      - ./src:/data

  web:
    image: nginx:1.20-alpine
    ports:
      - 8080:80
    volumes:
      - ./src:/data
      - ./infra/nginx/default.conf:/etc/nginx/conf.d/default.conf
    working_dir: /data
```

#### 主な設定項目の説明

- **version**: Dockerコンポーズのバージョンを指定  
- **services**: 各コンテナの設定を定義  
- **build**: Dockerfileの場所を指定（`./infra/php`）  
- **volumes**: ホストとコンテナ間のファイル共有設定  

---

### **2. PHPの設定**

#### Dockerfileの作成

必要なディレクトリを作成：  
`mkdir -p infra/php`  

Dockerfileを作成：  
`touch infra/php/Dockerfile`  

以下の内容を記述：

```dockerfile
FROM php:8.2-fpm-buster

ENV COMPOSER_ALLOW_SUPERUSER=1 \
  COMPOSER_HOME=/composer

COPY --from=composer:2.2 /usr/bin/composer /usr/bin/composer

RUN apt-get update && \
  apt-get -y install --no-install-recommends git unzip libzip-dev libicu-dev libonig-dev && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/* && \
  docker-php-ext-install intl pdo_mysql zip bcmath

COPY ./php.ini /usr/local/etc/php/php.ini

WORKDIR /data
```

#### php.iniの設定

php.iniファイルを作成：  
`touch infra/php/php.ini`

基本設定を記述：

```ini
default_charset = UTF-8

[Date]
date.timezone = Asia/Tokyo

[mysqlnd]
mysqlnd.collect_memory_statistics = on

[Assertion]
zend.assertions = 1

[mbstring]
mbstring.language = Japanese
```

---

### **3. Nginxの設定**

必要なディレクトリとファイルを作成：

```bash
mkdir infra/nginx
touch infra/nginx/default.conf
```

Nginxの設定を記述：

```nginx
server {
    listen 80;
    server_name example.com;
    root /data/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

---

### **4. コンテナの起動と動作確認**

以下のコマンドでコンテナをビルドして起動：

```bash
docker compose build
docker compose up -d
```

コンテナの起動状態を確認：  
`docker ps`

コンテナに入る：  
`docker container exec -it vercel-test-app-1 bash`

---

### **5. Laravelのインストール**

コンテナ内でLaravelをインストール：

```bash
docker container exec -it laravel11-app-1 bash
composer create-project --prefer-dist "laravel/laravel=11.*" .
```

権限を設定し、バージョンを確認：

```bash
chmod -R 777 storage bootstrap/cache
php artisan -V
```

---

### **6. データベースの設定**

docker-compose.ymlにデータベース設定を追加：

```yaml
version: "3.9"
services:
  app:
    build: ./infra/php
    volumes:
      - ./src:/data

  web:
    image: nginx:1.20-alpine
    ports:
      - 8080:80
    volumes:
      - ./src:/data
      - ./infra/nginx/default.conf:/etc/nginx/conf.d/default.conf
    working_dir: /data

  db:
    build: ./infra/mysql
    volumes:
      - db-store:/var/lib/mysql

volumes:
  db-store:
```

#### MySQLの設定

必要なファイルを作成：  
`mkdir infra/mysql`  
`touch infra/mysql/Dockerfile`

```dockerfile
# ベースイメージの指定
FROM mysql/mysql-server:8.0

# 環境変数の設定
ENV MYSQL_DATABASE=laravel \
    MYSQL_USER=phper \
    MYSQL_PASSWORD=secret \
    MYSQL_ROOT_PASSWORD=secret \
    TZ=Asia/Tokyo

# 設定ファイルのコピーと権限設定
COPY ./my.cnf /etc/my.cnf
RUN chmod 644 /etc/my.cnf
```

`touch infra/mysql/my.cnf`

my.cnfの設定：

```ini
[mysqld]
# default
skip-host-cache
skip-name-resolve
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
secure-file-priv = /var/lib/mysql-files
user = mysql

pid-file = /var/run/mysqld/mysqld.pid

# character set / collation
character_set_server = utf8mb4
collation_server = utf8mb4_ja_0900_as_cs_ks

# timezone
default-time-zone = SYSTEM
log_timestamps = SYSTEM

# Error Log
log-error = mysql-error.log

# Slow Query Log
slow_query_log = 1
slow_query_log_file = mysql-slow.log
long_query_time = 1.0
log_queries_not_using_indexes = 0

# General Log
general_log = 1
general_log_file = mysql-general.log

[mysql]
default-character-set = utf8mb4

[client]
default-character-set = utf8mb4
```

---

### **7. データベース接続の設定**

`.env` ファイルのデータベース設定：

```
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=phper
DB_PASSWORD=secret
```

---

### **8. マイグレーションの実行**

コンテナ内でマイグレーションを実行：

```bash
docker container exec -it laravel11-app-1 bash
php artisan migrate
composer install
```

- これで環境構築は完了です。全ての機能が正常に動作することを確認してください。
