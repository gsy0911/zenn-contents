---
title: "DockerでPHPのSymfonyを立ててAPI Platformを動かすまで"
emoji: "🎣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php", "symfony", "apiplatform"]
published: false
---

# はじめに

本記事では、`PHP + Symfony + API Platform`を`Docker`上で実行することを目指します。

# 環境・バージョン

- ローカル環境
  - macOS Ventura 13.4.1(c)
- Docker環境
  - PHP: 8.2
  - Symfony: 5.5系
  - mysql: 8.0

# 立ち上げからAPI実行まで

## コード

本記事のコードは[GitHub](https://github.com/gsy0911/zenn-php-symfony/tree/article1)にありますので、適宜確認ください。

## 立ち上げ

ディレクトリ直下で以下のコマンドを実行し、`zenn-php-symfony`コンテナ内に入ります。

```shell
$ docker compose up --build
$ docker compose exec -it php /bin/bash
```

## 初期化とパッケージなどのインストール

ここからは全てコンテナ内での操作です。
インストールされている`Symfony`のバージョンを確認します。

```shell
$ symfony -V
Symfony CLI version 5.5.7 (c) 2021-2023 Fabien Potencier #StandWithUkraine Support Ukraine (2023-07-21T10:15:00Z - stable)
```

Symfonyの5.5系が入っていることを確認し、新しいプロジェクトを作成します。

```shell
$ symfony new . -no-git
(...中略)
 [OK] Your project is now ready in /var/www/zenn_example
```

::::message
`$ composer create-project symfony/skeleton .`
と
`$ symfony new . --no-git`
の2通りの方法がありますが、今回は後者を採用しています。
（大きな差異は、パッケージのインストールがあるかないかだと思っています。）
::::

必要なパッケージのインストールを行います。
途中で聞かれる質問は`n`で大丈夫です。

```shell
$ composer require doctrine twig orm api
(...中略)
    This may create/update docker-compose.yml or update Dockerfile (if it exists).

    Do you want to include Docker configuration from recipes?
    [y] Yes
    [n] No
    [p] Yes permanently, never ask again for this project
    [x] No permanently, never ask again for this project
    (defaults to y): n

$ composer require symfony/maker-bundle maker ormfixtures --dev
```

## 設定の編集

最初に環境変数ファイルのコピーを行います。

```shell
$ cp .env .env.local
```

環境変数ファイルを以下の箇所を変更します。

```diff: .env.local
- DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=15&charset=utf8"
+ DATABASE_URL="mysql://root:secret@zenn-mysql-symfony:3306/zenn_example?serverVersion=8.0"
```

## Entityの作成

以下のコマンドを実行すると、実行可能コマンドの一覧が出てきます。
今回はそのうちの`make:entity`コマンドを使います。

```shell
$ symfony console
(...前略)
  make:entity                                Creates or updates a Doctrine entity class, and optionally an API Platform resource
(...後略)
```

例として、`Book`エンティティを作ってみます。
対話形式で必要なフィールドを入力していくとクラスを作成してくれます。

```shell
$ symfony console make:entity Book
 created: src/Entity/Book.php
 created: src/Repository/BookRepository.php

 Entity generated! Now let's add some fields!
 You can always add more fields later manually or by re-running this command.

 New property name (press <return> to stop adding fields):
 > title

 Field type (enter ? to see all types) [string]:
 > string

 Field length [255]:
 >

 Can this field be null in the database (nullable) (yes/no) [no]:
 >

 updated: src/Entity/Book.php

 Add another property? Enter the property name (or press <return> to stop adding fields):
 > author

 Field type (enter ? to see all types) [string]:
 > string

 Field length [255]:
 >

 Can this field be null in the database (nullable) (yes/no) [no]:
 >

 updated: src/Entity/Book.php

 Add another property? Enter the property name (or press <return> to stop adding fields):
 >



  Success!


 Next: When you're ready, create a migration with symfony console make:migration
```

以下に自動作成されたエンティティを載せておきます。

::::details 作成されたエンティティ


```php: src/Entity/Book
<?php

namespace App\Entity;

use App\Repository\BookRepository;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: BookRepository::class)]
class Book
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private ?string $title = null;

    #[ORM\Column(length: 255)]
    private ?string $author = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getTitle(): ?string
    {
        return $this->title;
    }

    public function setTitle(string $title): static
    {
        $this->title = $title;

        return $this;
    }

    public function getAuthor(): ?string
    {
        return $this->author;
    }

    public function setAuthor(string $author): static
    {
        $this->author = $author;

        return $this;
    }
}
```

::::

## マイグレーション

エンティティ作成時に指示されたコマンドを実行するだけで、versionファイルが作成されます。

```shell
$ symfony console make:migration
 created: migrations/Version20230721135723.php
 
  Success!

 Review the new migration then run it with symfony console doctrine:migrations:migrate
(...後略)
```

生成されたファイルを確認して問題なければ、以下のコマンドでマイグレーションを実行します。

```shell
$ symfony console doctrine:migrations:migrate
 WARNING! You are about to execute a migration in database "zenn_example" that could result in schema changes and data loss. Are you sure you wish to continue? (yes/no) [yes]:
 >

[notice] Migrating up to DoctrineMigrations\Version20230721135723
[notice] finished in 23.3ms, used 16M memory, 1 migrations executed, 1 sql queries

 [OK] Successfully migrated to version : DoctrineMigrations\Version20230721135723
```

mysqlのコンテナに入って確認してみます。

```shell
# ローカルから実行
$ docker compose exec -it mysql /bin/bash
bash-4.4# mysql -usymfony -p -h localhost zenn_example
Enter password: (symfonyと入力)

mysql> show tables;
+-----------------------------+
| Tables_in_zenn_example      |
+-----------------------------+
| book                        |
| doctrine_migration_versions |
+-----------------------------+
2 rows in set (0.00 sec)

mysql> desc `book`;
+--------+--------------+------+-----+---------+----------------+
| Field  | Type         | Null | Key | Default | Extra          |
+--------+--------------+------+-----+---------+----------------+
| id     | int          | NO   | PRI | NULL    | auto_increment |
| title  | varchar(255) | NO   |     | NULL    |                |
| author | varchar(255) | NO   |     | NULL    |                |
+--------+--------------+------+-----+---------+----------------+
3 rows in set (0.01 sec)
```

意図した`book`テーブルが作成されていることが確認できました。

## CRUDのAPI作成

最後に`API Platform`の機能を使って、単純なCURDのAPIを作成します。
最も単純なパターンは以下に2箇所コードを追記するだけです。

```diff php:src/Entity/Book.php
<?php

namespace App\Entity;

use App\Repository\BookRepository;
+ use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: BookRepository::class)]
+ #[ApiResource]
class Book
{
(...後略)
```

追記したら`http://localhost:8080/api` にアクセスします。

![](/images/php_symfony_1/php_symfony_1_1.png =600x)

この画面で適当なAPIを叩いてみます。

![](/images/php_symfony_1/php_symfony_1_2.png =600x)

すると以下の結果が返ってきます。

![](/images/php_symfony_1/php_symfony_1_3.png =600x)

データベースを覗いてみるとデータが入っていることが確認できます。

```shell
# mysqlのコンテナ
mysql> select * from book;
+----+----------------+-------------+
| id | title          | author      |
+----+----------------+-------------+
|  1 | zenn-example-1 | some author |
+----+----------------+-------------+
1 row in set (0.00 sec)
```

`GET /api/books/1`を実行しても結果が返ってくることが確認できました。

![](/images/php_symfony_1/php_symfony_1_4.png =600x)

# おわりに

`Book`エンティティに対してCRUDのAPIを作成しました。
ただ、これだけだとORMと合わせて利用する必要があり、APIとしては少し扱いにくいです。
これから`API Platform`の機能を利用してAPIを作成していきたいです。

# 参考文献

- [The Fast Track - 基礎から最速で学ぶ Symfony 入門](https://symfony.com/doc/5.4/the-fast-track/ja/index.html)
- [symfony を Dockerで利用する](https://qiita.com/idani/items/73fb7041aa84ea60a778)
- [symfony-docker](https://github.com/dunglas/symfony-docker#getting-started)
- [Let's EncryptでHTTPSを終端させたいだけならNginxよりCaddyを使うと楽だった件](https://qiita.com/ssc-ksaitou/items/ee0cda84dcf358a2b5eb)
- [nginx と PHP-FPM の仕組みをちゃんと理解しながら PHP の実行環境を構築する](https://qiita.com/kotarella1110/items/634f6fafeb33ae0f51dc)
- [Docker ComposeとSymfonyを使って開発してみよう](https://www.twilio.com/ja/blog/get-started-docker-symfony-jp)

