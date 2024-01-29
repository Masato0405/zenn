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

`php artisan make:model Post --migration`
`php artisan make:model Tag --migration`

## データの用意

seed

## リレーションを定義する

`belongsToMany`を両者につける

## フォームの作成



# おわりに
