# 前期課題

画像投稿できる掲示板の実装手順について書いています。

## 実装手順

### 1. プロジェクト構成

このようにファイル・ディレクトリを作成してください

```pgsql
zenkikadai/
├── Dockerfile
├── compose.yml
├── nginx/
│   └── conf.d/
│       └── default.conf
└── public/
    └── index.php
```

### 2. 各ファイルの内容

#### 2-1. Dockerfile
PHP + Nginx + MySQLの環境を作成するための設定です。

```dockerfile
FROM php:8.4-fpm-alpine AS php
RUN docker-php-ext-install pdo_mysql
RUN install -o www-data -g www-data -d /var/www/upload/image/
RUN echo -e "post_max_size = 5M\nupload_max_filesize = 5M" >> ${PHP_INI_DIR}/php.ini
```
#### 2-2. compose.yml
Docker Compose で PHP + Nginx + MySQL を連携。

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - 80:80
    volumes:
      - ./nginx/conf.d/:/etc/nginx/conf.d/
      - ./public/:/var/www/public/
      - image:/var/www/upload/image/
    depends_on:
      - php
  php:
    container_name: php-kadai
    build:
      context: .
      target: php-kadai
    volumes:
      - ./public/:/var/www/public/
      - image:/var/www/upload/image/
  mysql:
    container_name: mysql-kadai
    image: mysql:8.4
    environment:
      MYSQL_DATABASE: example_db
      MYSQL_ALLOW_EMPTY_PASSWORD: 1
      TZ: Asia/Tokyo
    volumes:
      - mysql:/var/lib/mysql
    command: >
      mysqld
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --max_allowed_packet=4MB
volumes:
  mysql:
  image:
```

#### 2-3. nginx/conf.d/default.conf
Nginx のリバースプロキシ設定。

```nginx
server {
    listen       0.0.0.0:80;
    server_name  _;
    charset      utf-8;
    client_max_body_size 6M;

    root /var/www/public;
    index index.php;

    location ~ \.php$ {
        fastcgi_pass  php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include       fastcgi_params;
    }

    location /image/ {
      root /var/www/upload;
     }
}
```

#### 2-4. public/index.php
掲示板用PHPファイル

```php
<?php
$dbh = new PDO('mysql:host=mysql;dbname=kadai', 'root', '');

if (isset($_POST['body'])) {
  // POSTで送られてくるフォームパラメータ body がある場合

  $image_filename = null;
  if (isset($_FILES['image']) && !empty($_FILES['image']['tmp_name'])) {
    // アップロードされた画像がある場合
    if (preg_match('/^image\//', mime_content_type($_FILES['image']['tmp_name'])) !== 1) {
      // アップロードされたものが画像ではなかった場合処理を強制的に終了
      header("HTTP/1.1 302 Found");
      header("Location: ./index.php");
      return;
    }

    // 元のファイル名から拡張子を取得
    $pathinfo = pathinfo($_FILES['image']['name']);
    $extension = $pathinfo['extension'];
    // 新しいファイル名を決める。他の投稿の画像ファイルと重複しないように時間+乱数で決める。
    $image_filename = strval(time()) . bin2hex(random_bytes(25)) . '.' . $extension;
    $filepath =  '/var/www/upload/image/' . $image_filename;
    move_uploaded_file($_FILES['image']['tmp_name'], $filepath);
  }

  // insertする
  $insert_sth = $dbh->prepare("INSERT INTO kadai_post (body, image_filename) VALUES (:body, :image_filename)");
  $insert_sth->execute([
    ':body' => $_POST['body'],
    ':image_filename' => $image_filename,
  ]);

  // 処理が終わったらリダイレクトする
  // リダイレクトしないと，リロード時にまた同じ内容でPOSTすることになる
  header("HTTP/1.1 302 Found");
  header("Location: ./index.php");
  return;
}

// いままで保存してきたものを取得
$select_sth = $dbh->prepare('SELECT * FROM kadai_post ORDER BY created_at DESC');
$select_sth->execute();
?>
<head>
  <title>画像投稿できる掲示板</title>

  <style>
    body {
      font-family: sans-serif;
      margin: 20px;
      background: #f8f9fa;
      color: #333;
    }
    h2 { margin-bottom: 16px; }
    form {
      background: #fff;
      border: 1px solid #ddd;
      padding: 16px;
      border-radius: 8px;
      margin-bottom: 24px;
    }
    textarea {
      width: 100%;
      min-height: 100px;
      padding: 8px;
      border: 1px solid #ccc;
      border-radius: 6px;
    }
    input[type="file"] { margin: 8px 0; }
    button {
      padding: 8px 16px;
      border: none;
      border-radius: 6px;
      background: #2563eb;
      color: #fff;
      cursor: pointer;
    }
    button:hover { background: #1d4ed8; }
    dl {
      background: #fff;
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 12px;
      margin-bottom: 16px;
    }
    dt { font-weight: bold; }
    dd { margin: 0 0 8px 0; }
    img {
      max-width: 100%;
      max-height: 200px;
      border-radius: 6px;
      margin-top: 8px;
    }
  </style>
</head>

<!-- フォームのPOST先はこのファイル自身にする -->
<h2>掲示板だよ</h2>
<form method="POST" action="./index.php" enctype="multipart/form-data">
  <textarea name="body" required></textarea>
  <div style="margin: 1em 0;">
    <input type="file" accept="image/*" name="image" id="imageInput">
  </div>
  <button type="submit">送信</button>
</form>

<hr>

<?php foreach($select_sth as $entry): ?>
  <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
    <dt>ID</dt>
    <dd><?= $entry['id'] ?></dd>
    <dt>日時</dt>
    <dd><?= $entry['created_at'] ?></dd>
    <dt>内容</dt>
    <dd>
      <?= nl2br(htmlspecialchars($entry['body'])) // 必ず htmlspecialchars() すること ?>
      <?php if(!empty($entry['image_filename'])): // 画像がある場合は img 要素を使って表示 ?>
      <div>
        <img src="/image/<?= $entry['image_filename'] ?>" style="max-height: 10em;">
      </div>
      <?php endif; ?>
    </dd>
  </dl>
<?php endforeach ?>

<script>
document.addEventListener("DOMContentLoaded", () => {
  const imageInput = document.getElementById("imageInput");
  imageInput.addEventListener("change", () => {
    if (imageInput.files.length < 1) {
      return;
    }
    if (imageInput.files[0].size > 5 * 1024 * 1024) {
      // ファイルが5MBより多い場合
      alert("5MB以下のファイルを選択してください。");
      imageInput.value = "";
    }
  });
});
</script>

```
### 3. 実行
mysql内のMySQLサーバーに接続するためにコンテナを起動します
1. プロジェクトディレクトリに移動
  ```bash
  cd zenkikadai
  ```
2. Dockerイメージをビルド
  ```bash
  docker compose build

  ```
3. コンテナを起動
  ```bash
  docker compose up
  ```
### 4. データベース・テーブルの作成

#### 4-1. dockerコマンドを使用しMySQLコンテナ内に入ります。
```bash
docker exec -it mysql-kadai mysql
```

#### 4-2. データベースの作成と使用
```bash
CREATE DATABASE kadai;
USE kadai;
```

#### 4-3. テーブルの作成
```bash
CREATE TABLE `kadai_post` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `body` TEXT NOT NULL,
    `image_filename` TEXT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

#### 4-4. コンテナから退出
```bash
exit
```

### アクセス方法
コンテナ起動後、ブラウザで以下にアクセスしてください。
http://{インスタンスのパブリックipアドレス}

完成画面
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/42013454-c913-4bd0-bf25-d092479a95d3" />

### コンテナの停止方法
```bash
docker compose down
```

