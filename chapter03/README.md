# 第3章: 基本的な属性（hx-get, hx-post）

## 章の概要

この章では、HTMXの最も基本的で重要な属性である`hx-get`と`hx-post`について詳しく学びます。これらの属性を使用することで、JavaScriptを書かずにAJAXリクエストを実装できます。

## 3.1 HTTPメソッドの基礎知識

HTMXを理解する前に、HTTPメソッドについて簡単に復習しましょう。

### 主なHTTPメソッド

| メソッド | 用途 | 特徴 |
|---------|------|------|
| GET | データの取得 | 冪等性あり、キャッシュ可能 |
| POST | データの作成・送信 | 冪等性なし、ボディにデータを含む |
| PUT | データの更新（全体） | 冪等性あり |
| DELETE | データの削除 | 冪等性あり |
| PATCH | データの部分更新 | 冪等性なし |

HTMXでは、これらすべてのメソッドに対応する属性が用意されています：
- `hx-get`
- `hx-post`
- `hx-put`
- `hx-delete`
- `hx-patch`

## 3.2 hx-get属性の詳細

`hx-get`は最も頻繁に使用される属性で、サーバーからデータを取得する際に使用します。

### 基本的な使い方

```html
<button hx-get="/api/users">ユーザー一覧を取得</button>
```

### 完全な例

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>hx-getの例</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .user-card {
            border: 1px solid #ddd;
            padding: 10px;
            margin: 5px 0;
            border-radius: 4px;
        }
        .loading {
            color: #666;
            font-style: italic;
        }
    </style>
</head>
<body>
    <h1>ユーザー管理システム</h1>
    
    <!-- 単純なGETリクエスト -->
    <div>
        <h2>ユーザー一覧</h2>
        <button hx-get="/api/users" 
                hx-target="#user-list">
            ユーザーを読み込む
        </button>
        <div id="user-list">
            <!-- ここにユーザー一覧が表示される -->
        </div>
    </div>
    
    <!-- パラメータ付きGETリクエスト -->
    <div>
        <h2>ユーザー検索</h2>
        <input type="text" 
               name="search" 
               placeholder="名前で検索..."
               hx-get="/api/users/search"
               hx-trigger="keyup changed delay:500ms"
               hx-target="#search-results">
        <div id="search-results">
            <!-- 検索結果がここに表示される -->
        </div>
    </div>
</body>
</html>
```

### サーバー側の実装例（Flask）

```python
from flask import Flask, request, render_template_string

app = Flask(__name__)

# ダミーデータ
users = [
    {"id": 1, "name": "田中太郎", "email": "tanaka@example.com"},
    {"id": 2, "name": "鈴木花子", "email": "suzuki@example.com"},
    {"id": 3, "name": "佐藤次郎", "email": "sato@example.com"},
]

@app.route('/api/users')
def get_users():
    html = "<div>"
    for user in users:
        html += f'''
        <div class="user-card">
            <h3>{user['name']}</h3>
            <p>Email: {user['email']}</p>
        </div>
        '''
    html += "</div>"
    return html

@app.route('/api/users/search')
def search_users():
    query = request.args.get('search', '').lower()
    filtered_users = [u for u in users if query in u['name'].lower()]
    
    if not filtered_users:
        return "<p>該当するユーザーが見つかりません</p>"
    
    html = "<div>"
    for user in filtered_users:
        html += f'''
        <div class="user-card">
            <h3>{user['name']}</h3>
            <p>Email: {user['email']}</p>
        </div>
        '''
    html += "</div>"
    return html
```

### URLパラメータの扱い

```html
<!-- 方法1: URLに直接パラメータを含める -->
<button hx-get="/api/users?page=2&limit=10">
    次のページ
</button>

<!-- 方法2: hx-paramsを使用 -->
<button hx-get="/api/users" 
        hx-params="page=2&limit=10">
    次のページ
</button>

<!-- 方法3: フォーム要素から自動的に取得 -->
<form>
    <input type="hidden" name="page" value="2">
    <input type="hidden" name="limit" value="10">
    <button hx-get="/api/users">次のページ</button>
</form>
```

## 3.3 hx-post属性の詳細

`hx-post`は、データをサーバーに送信する際に使用します。主にフォームの送信や新規データの作成に使われます。

### 基本的な使い方

```html
<form hx-post="/api/users/create">
    <input type="text" name="name" required>
    <input type="email" name="email" required>
    <button type="submit">ユーザーを作成</button>
</form>
```

### 完全な例：ユーザー登録フォーム

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>hx-postの例</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        form {
            max-width: 400px;
            margin: 20px 0;
        }
        .form-group {
            margin-bottom: 15px;
        }
        label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        input, textarea {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .success {
            color: green;
            padding: 10px;
            background-color: #e7f5e7;
            border-radius: 4px;
        }
        .error {
            color: red;
            padding: 10px;
            background-color: #f5e7e7;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <h1>ユーザー登録</h1>
    
    <!-- 基本的なPOSTフォーム -->
    <form hx-post="/api/users" 
          hx-target="#result"
          hx-swap="innerHTML">
        <div class="form-group">
            <label for="name">名前:</label>
            <input type="text" id="name" name="name" required>
        </div>
        
        <div class="form-group">
            <label for="email">メールアドレス:</label>
            <input type="email" id="email" name="email" required>
        </div>
        
        <div class="form-group">
            <label for="bio">自己紹介:</label>
            <textarea id="bio" name="bio" rows="4"></textarea>
        </div>
        
        <button type="submit">登録</button>
    </form>
    
    <div id="result">
        <!-- 結果がここに表示される -->
    </div>
    
    <!-- JSONデータの送信 -->
    <h2>JSON形式でのデータ送信</h2>
    <button hx-post="/api/users/json"
            hx-vals='{"name": "テストユーザー", "email": "test@example.com"}'
            hx-target="#json-result">
        JSONデータを送信
    </button>
    <div id="json-result"></div>
</body>
</html>
```

### サーバー側の実装例（Flask）

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/users', methods=['POST'])
def create_user():
    # フォームデータの取得
    name = request.form.get('name')
    email = request.form.get('email')
    bio = request.form.get('bio', '')
    
    # バリデーション
    if not name or not email:
        return '<div class="error">名前とメールアドレスは必須です</div>', 400
    
    # ここで実際にはデータベースに保存する
    # ...
    
    return f'''
    <div class="success">
        <h3>登録完了！</h3>
        <p>名前: {name}</p>
        <p>メール: {email}</p>
        <p>自己紹介: {bio}</p>
    </div>
    '''

@app.route('/api/users/json', methods=['POST'])
def create_user_json():
    # JSONデータの取得
    data = request.get_json()
    
    return f'''
    <div class="success">
        <h3>JSONデータを受信しました</h3>
        <pre>{str(data)}</pre>
    </div>
    '''
```

## 3.4 フォームデータの送信

### 自動的なフォームデータの収集

HTMXは、フォーム内のすべての入力要素を自動的に収集します：

```html
<form hx-post="/api/contact">
    <input type="text" name="name">
    <input type="email" name="email">
    <select name="subject">
        <option value="inquiry">お問い合わせ</option>
        <option value="support">サポート</option>
    </select>
    <textarea name="message"></textarea>
    <button type="submit">送信</button>
</form>
```

### hx-includeを使った追加データの送信

```html
<!-- フォーム外の要素も含める -->
<input type="text" id="extra-field" name="extra" value="追加データ">

<form hx-post="/api/submit" 
      hx-include="#extra-field">
    <input type="text" name="main-data">
    <button type="submit">送信</button>
</form>
```

### ファイルアップロード

```html
<form hx-post="/api/upload" 
      hx-encoding="multipart/form-data"
      hx-target="#upload-result">
    <input type="file" name="file" accept="image/*">
    <button type="submit">アップロード</button>
</form>
<div id="upload-result"></div>
```

## 3.5 リクエストヘッダーとカスタマイズ

### カスタムヘッダーの追加

```html
<div hx-post="/api/data"
     hx-headers='{"X-Custom-Header": "value", "Authorization": "Bearer token123"}'>
    カスタムヘッダー付きリクエスト
</div>
```

### CSRFトークンの送信

```html
<meta name="csrf-token" content="{{ csrf_token }}">

<script>
    document.body.addEventListener('htmx:configRequest', (event) => {
        event.detail.headers['X-CSRF-Token'] = document.querySelector('meta[name="csrf-token"]').content;
    });
</script>
```

## 3.6 エラーハンドリング

### 基本的なエラーハンドリング

```html
<div hx-get="/api/data"
     hx-target="#content"
     hx-on="htmx:responseError: alert('エラーが発生しました')">
    データを取得
</div>
```

### 詳細なエラーハンドリング

```html
<script>
document.body.addEventListener('htmx:responseError', function(event) {
    const xhr = event.detail.xhr;
    const target = event.detail.target;
    
    if (xhr.status === 404) {
        target.innerHTML = '<div class="error">データが見つかりません</div>';
    } else if (xhr.status === 500) {
        target.innerHTML = '<div class="error">サーバーエラーが発生しました</div>';
    } else {
        target.innerHTML = '<div class="error">予期しないエラーが発生しました</div>';
    }
});
</script>
```

## 3.7 実践的な使用例

### 例1: 無限スクロール

```html
<div id="post-list">
    <!-- 初期の投稿 -->
    <div class="post">投稿1</div>
    <div class="post">投稿2</div>
    
    <!-- もっと読み込むトリガー -->
    <div hx-get="/api/posts?page=2"
         hx-trigger="revealed"
         hx-swap="afterend"
         hx-indicator="#loading">
        <span id="loading" class="htmx-indicator">読み込み中...</span>
    </div>
</div>
```

### 例2: 自動保存機能

```html
<form id="auto-save-form">
    <textarea name="content"
              hx-post="/api/draft/save"
              hx-trigger="keyup changed delay:2s"
              hx-target="#save-status">
    </textarea>
    <div id="save-status"></div>
</form>
```

### 例3: 削除確認

```html
<div class="item" id="item-123">
    <h3>アイテム名</h3>
    <button hx-delete="/api/items/123"
            hx-target="#item-123"
            hx-swap="outerHTML"
            hx-confirm="本当に削除しますか？">
        削除
    </button>
</div>
```

## まとめ

この章では、HTMXの基本的なHTTP通信属性について学びました：

1. **hx-get**: データの取得とページの部分更新
2. **hx-post**: フォームデータの送信と新規作成
3. **フォームデータの自動収集**: name属性を持つ要素の自動送信
4. **エラーハンドリング**: HTTPエラーの適切な処理
5. **実践的な使用例**: 無限スクロール、自動保存、削除確認

次章では、`hx-target`と`hx-swap`を使って、レスポンスの表示方法をより細かく制御する方法を学びます。

## 練習問題

### 問題1: 基本的なGET/POST
以下の機能を実装してください：
1. 商品一覧を表示するボタン（hx-get使用）
2. 新商品を追加するフォーム（hx-post使用）
3. 追加後、自動的に商品一覧を更新

### 問題2: 検索機能
リアルタイム検索機能を実装してください：
- 入力フィールドに文字を入力すると自動的に検索
- 500ms のデバウンス処理
- 検索中はローディング表示
- 結果がない場合は「見つかりません」と表示

### 問題3: ページネーション
ページネーション機能を実装してください：
- 前へ/次へボタン
- 現在のページ番号表示
- ページ番号をパラメータとして送信

### 問題4: エラーハンドリング
以下のエラーケースを適切に処理してください：
1. 404エラー: 「データが見つかりません」
2. 500エラー: 「サーバーエラー」
3. ネットワークエラー: 「接続できません」

### 問題5: ファイルアップロード
画像アップロード機能を実装してください：
- ファイル選択と同時にアップロード開始
- アップロード中のプログレス表示
- 成功時はサムネイルを表示
- エラー時は適切なメッセージ

---

**解答例とヒント：**

前章の練習問題の解答例：

問題1（第1章）の解答：
1. HTMXの主な目的: HTMLの属性だけでインタラクティブなWebアプリを作成すること
2. SPAとの違い: ①状態管理がサーバー側、②ビルドプロセス不要、③SEOフレンドリー
3. HATEOAS: Hypermedia As The Engine Of Application State - アプリケーションの状態をハイパーメディア（HTML）で表現する考え方

問題2（第1章）の解答：
このコードは5秒ごとに`/notifications`にGETリクエストを送信し、レスポンスで要素の内容を置き換えます。