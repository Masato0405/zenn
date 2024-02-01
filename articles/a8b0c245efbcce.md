---
title: "【Laravel】syncで中間テーブルへの挿入・削除を楽にする"
emoji: "🀄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Laravel", "PHP"]
published: true
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

## 準備
### 環境構築

Laravelの環境はLaravel Sailで構築します。  
`curl -s "https://laravel.build/middleware-test?with=mysql" | bash`

`./vendor/bin/sail up`でコンテナを起動します。


### モデル・テーブルの準備

今回は中間テーブルの例として、ブログ記事テーブルとタグテーブルを想定します。

#### posts(記事)テーブル

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

### タグデータの用意
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

タグのデータを作成できました😺

![tag](https://storage.googleapis.com/zenn-user-upload/6347145c2a6c-20240131.png)

### リレーションを定義する

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

### コントローラー作成

リソースコントローラーというものがあったので、それを使います😺

`php artisan make:controller PostController --model=Post --resource`

CRUDのアクションを定義してくれます。
（もちろん具体的な処理は自分で書かないといけませんが）

↓のように改造しました。

```PHP:app/Http/Controllers/PostController.php
class PostController extends Controller
{
    // 新規投稿ページ
    public function create()
    {
        return view('create', ['tags' => Tag::all()]);
    }

    // 登録処理
    public function store(Request $request)
    {
        $request->validate(
            [
                'title' => 'required|string|unique:posts|max:255',
                'body' => 'required',
            ]
        );

        DB::beginTransaction();
        $post = new Post;
        $post->title = $request->title;
        $post->body = $request->body;
        $post->save();

        // 中間テーブルへの登録
        $post->tags()->sync($request->tags);
        DB::commit();

        return redirect('/posts/' . $post->id);
    }

    // 詳細ページ
    public function show(Post $post)
    {
        return view('show', ['post' => $post]);
    }

    // 編集ページ
    public function edit(Post $post)
    {
        return view('edit', ['post' => $post, 'tags' => Tag::all()]);
    }

    // 更新処理
    public function update(Request $request, Post $post)
    {
        $request->validate(
            [
                'title' => ['required', 'string', 'max:255', Rule::unique('posts')->ignore($post->id)],
                'body' => 'required',
            ]
        );

        DB::beginTransaction();
        $post->title = $request->title;
        $post->body = $request->body;
        $post->save();

        // 中間テーブルの更新
        $post->tags()->sync($request->tags);
        DB::commit();

        return redirect('/posts/' . $post->id);
    }
}
```

### ルーティング作成

`routes/web.php`に`Route::resource('posts', PostController::class);`を追記します。
これで、各CRUDアクションへのルートを定義してくれます。

### ビュー作成

`php artisan make:view create`（新規投稿ページ）

```PHP:/resources/views/create.blade.php
<form method="post" action="/posts" style="display:flex; flex-direction:column; width:25%; margin:auto;">
    @csrf
    <label for="title">タイトル</label>
    <input type="text" name="title">
    <label for="body">本文</label>
    <textarea name="body" id="" cols="30" rows="10"></textarea>

    <select name="tags[]" id="" multiple>
        @foreach ($tags as $tag)
            <option value="{{ $tag->id }}">{{ $tag->name }}</option>
        @endforeach
    </select>
    <button>投稿する</button>
</form>
```

`php artisan make:view show`（詳細ページ）

```PHP:/resources/views/show.blade.php
<div style="display:flex; flex-direction:column; width:25%; margin:auto;">
    <h3>タイトル:{{ $post->title }}</h3>
    <h5>本文:{{ $post->body }}</h5>

    <div style="display:flex; flex-direction:row; gap: 10px;">
        <p>タグ:</p>
        @foreach ($post->tags as $tag)
            <p style="border: solid 1px;">{{ $tag->name }}</p>
        @endforeach
    </div>

    <a href="{{$post->id}}/edit">編集する</a>
</div>
```

`php artisan make:view edit`（編集ページ）

```PHP:/resources/views/edit.blade.php
<form method="post" action="/posts/{{$post->id}}" style="display:flex; flex-direction:column; width:25%; margin:auto;">
    @method('PUT')
    @csrf
    <label for="title">タイトル</label>
    <input type="text" name="title" value="{{ $post->title }}">
    <label for="body">本文</label>
    <textarea name="body" id="" cols="30" rows="10">{{ $post->body }}</textarea>

    <select name="tags[]" id="" multiple>
        @foreach ($tags as $tag)
            <option value="{{ $tag->id }}" {{ $post->tags->contains($tag->id) ? 'selected' : '' }}>{{ $tag->name }}</option>
        @endforeach
    </select>
    <button>投稿する</button>
</form>
```

## 実際に試してみる

### 記事のタグを登録してみる
記事を作成します。
タグは`hoge`, `fuga`, `piyo`の3つを選択します。

![create](https://storage.googleapis.com/zenn-user-upload/d5f07bfa043e-20240201.png)

ちゃんと3つのタグが登録されていますね！

![show](https://storage.googleapis.com/zenn-user-upload/1fabe99ec024-20240201.png)

### 記事のタグを更新してみる

今度は`foo`, `bar`を選択します。

![edit](https://storage.googleapis.com/zenn-user-upload/1aca90f9edd4-20240201.png)

`hoge`, `fuga`, `piyo`の3つが消えて、新たに`foo`, `bar`の2つが表示されています😺
![](https://storage.googleapis.com/zenn-user-upload/9632fa650ead-20240201.png)

## `sync`のコード

```PHP:vendor/laravel/framework/src/Illuminate/Database/Eloquent/Relations/Concerns/InteractsWithPivotTable.php
/**
     * Sync the intermediate tables with a list of IDs or collection of models.
     *
     * @param  \Illuminate\Support\Collection|\Illuminate\Database\Eloquent\Model|array  $ids
     * @param  bool  $detaching
     * @return array
     */
    public function sync($ids, $detaching = true)
    {
        $changes = [
            'attached' => [], 'detached' => [], 'updated' => [],
        ];

        // First we need to attach any of the associated models that are not currently
        // in this joining table. We'll spin through the given IDs, checking to see
        // if they exist in the array of current ones, and if not we will insert.
        $current = $this->getCurrentlyAttachedPivots()
                        ->pluck($this->relatedPivotKey)->all();

        $records = $this->formatRecordsList($this->parseIds($ids));

        // Next, we will take the differences of the currents and given IDs and detach
        // all of the entities that exist in the "current" array but are not in the
        // array of the new IDs given to the method which will complete the sync.
        if ($detaching) {
            $detach = array_diff($current, array_keys($records));

            if (count($detach) > 0) {
                $this->detach($detach);

                $changes['detached'] = $this->castKeys($detach);
            }
        }

        // Now we are finally ready to attach the new records. Note that we'll disable
        // touching until after the entire operation is complete so we don't fire a
        // ton of touch operations until we are totally done syncing the records.
        $changes = array_merge(
            $changes, $this->attachNew($records, $current, false)
        );

        // Once we have finished attaching or detaching the records, we will see if we
        // have done any attaching or detaching, and if we have we will touch these
        // relationships if they are configured to touch on any database updates.
        if (count($changes['attached']) ||
            count($changes['updated']) ||
            count($changes['detached'])) {
            $this->touchIfTouching();
        }

        return $changes;
    }
```

当たり前ですがリファレンスにある通り、↓ですね…。
- 指定されたIDが現在の中間テーブルにない場合は、新たに追加する
- 現在の中間テーブルにはあるが、指定されたIDに含まれない場合は削除する

最後にある`$this->touchIfTouching();`は中間テーブルが更新された場合に、関連するテーブル（今回だとPostとTag）のタイムスタンプを更新できるもののようです🧐（Modelで設定が必要そうです）

# おわりに

ドキュメントを読んでると色々便利なメソッドがありますね！
他にも`toggle`も便利そうですね…！（「いいね」などの切り替えに使える感じですかね🤔）