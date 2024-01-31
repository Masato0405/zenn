---
title: "【Laravel】syncで中間テーブルへの挿入・削除を楽にする"
emoji: "🀄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Laravel", "PHP"]
published: false
---

# この記事について

Eloquentの`sync`メソッドを使ってみた&ちょっと調べてみた記事です。
何か改善点などあればご指摘ください 🙏

# syncについて

> syncメソッドを使用して、多対多の関連付けを構築することもできます。syncメソッドは、中間テーブルに配置するIDの配列を引数に取ります
指定した配列にないIDは、中間テーブルから削除されます。したがってこの操作が完了すると、指定した配列のIDのみが中間テーブルに残ります。
https://readouble.com/laravel/10.x/ja/eloquent-relationships.html

要は、↓という感じですかね。（説明が難しい😿）
- `sync`に渡したIDは中間テーブルに登録される（+既に同じIDの組み合わせが存在する場合はそのまま）
- `sync`に渡されなかったIDは中間テーブルに登録されない（+テーブルに存在する場合は削除）

# 実践

## 環境構築

Laravelの環境はLaravel Sailで構築します。  
`curl -s "https://laravel.build/middleware-test?with=mysql" | bash`

`./vendor/bin/sail up`でコンテナを起動します。


## モデル・テーブルの準備

今回は中間テーブルの例として、ブログ記事テーブルとタグテーブルを想定します。

### posts（記事)テーブル

`php artisan make:model Post --migration`

```PHP:database/migrations/2024_01_30_103649_create_posts_table.php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->string('body');
    $table->timestamps();
});
```

記事テーブルには、一応タイトルとボディカラムを追加しました😺

### tags(タグ)テーブル
`php artisan make:model Tag --migration`

```PHP:database/migrations/2024_01_30_103656_create_tags_table.php
Schema::create('tags', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->timestamps();
});
```

### 中間テーブル

`php artisan make:migration create_post_tag_table`

```PHP:database/migrations/2024_01_30_104404_create_post_tag_table.php
Schema::create('post_tag', function (Blueprint $table) {
    $table->foreignId('post_id')->constrained();
    $table->foreignId('tag_id')->constrained();
    $table->timestamps();
});
```

## タグデータの用意
シードを利用してデータを用意します😺

```PHP:database/seeders/DatabaseSeeder.php
public function run(): void {
    DB::table('tags')->insert(
        [
            [
                'name' => 'hoge',
                'created_at' => now(),
                'updated_at' => now()
            ],
            [
                'name' => 'fuga',
                'created_at' => now(),
                'updated_at' => now()
            ],
            [
                'name' => 'piyo',
                'created_at' => now(),
                'updated_at' => now()
            ],
            [
                'name' => 'foo',
                'created_at' => now(),
                'updated_at' => now()
            ],
            [
                'name' => 'bar',
                'created_at' => now(),
                'updated_at' => now()
            ]
        ],
    );
}
```

`php artisan db:seed`を実行します。

データを作成できました😺

![tag](https://storage.googleapis.com/zenn-user-upload/6347145c2a6c-20240131.png)

## リレーションを定義する

リレーションを定義します。  
今回は多対多の関係になるので、`BelongsToMany`を使います。

```PHP:app/Models/Post.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Post extends Model
{
    use HasFactory;

    public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class);
    }
}
```

```PHP:app/Models/Tag.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Tag extends Model
{
    use HasFactory;

    public function posts(): BelongsToMany
    {
        return $this->belongsToMany(Post::class);
    }
}

```

## コントローラー作成

リソースコントローラーというものがあったので、それを使います😺

`php artisan make:controller PostController --model=Post --resource`

CRUDのアクションを定義してくれます。
（もちろん具体的な処理は自分で書かないといけませんが）

## ルーティング作成

## ビュー作成


# おわりに
