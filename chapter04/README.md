# 第4章: ターゲットとスワップ（hx-target, hx-swap）

## 章の概要

この章では、HTMXのレスポンス処理を制御する重要な属性である`hx-target`と`hx-swap`について学びます。これらの属性を使うことで、サーバーからのレスポンスをどこに、どのように表示するかを細かく制御できます。

## 4.1 hx-targetの基本概念

`hx-target`は、AJAXレスポンスを挿入する対象要素を指定する属性です。

### 基本的な構文

```html
<button hx-get="/api/data" hx-target="#result">
    データ取得
</button>
<div id="result">
    <!-- レスポンスがここに表示される -->
</div>
```

### CSSセレクタの使用

`hx-target`では、あらゆるCSSセレクタを使用できます：

```html
<!-- IDセレクタ -->
<button hx-get="/api/users" hx-target="#user-list">ユーザー一覧</button>

<!-- クラスセレクタ -->
<button hx-get="/api/messages" hx-target=".message-container">メッセージ</button>

<!-- 要素セレクタ -->
<button hx-get="/api/content" hx-target="main">メインコンテンツ</button>

<!-- 複合セレクタ -->
<button hx-get="/api/data" hx-target="div.content > p:first-child">最初の段落</button>
```

### 特殊なターゲットキーワード

HTMXは便利な特殊キーワードを提供しています：

```html
<!-- this: トリガー要素自身 -->
<div hx-get="/api/self" hx-target="this">
    クリックで自分を更新
</div>

<!-- closest: 最も近い親要素 -->
<button hx-delete="/api/items/1" 
        hx-target="closest .item">
    削除
</button>

<!-- next: 次の兄弟要素 -->
<button hx-get="/api/details" 
        hx-target="next .details">
    詳細を表示
</button>

<!-- previous: 前の兄弟要素 -->
<button hx-get="/api/summary" 
        hx-target="previous .summary">
    概要を更新
</button>
```

### 実践例：動的なカード更新

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>hx-targetの実践例</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .card {
            border: 1px solid #ddd;
            padding: 15px;
            margin: 10px;
            border-radius: 8px;
        }
        .card-actions {
            margin-top: 10px;
        }
        button {
            margin-right: 5px;
            padding: 5px 10px;
        }
    </style>
</head>
<body>
    <h1>商品カード管理</h1>
    
    <div class="card">
        <h3>商品A</h3>
        <div class="details">
            <p>価格: ¥1,000</p>
            <p>在庫: 10個</p>
        </div>
        <div class="card-actions">
            <!-- カード全体を更新 -->
            <button hx-get="/api/products/a/refresh" 
                    hx-target="closest .card">
                更新
            </button>
            
            <!-- 詳細部分だけを更新 -->
            <button hx-get="/api/products/a/details" 
                    hx-target="previous .details">
                詳細更新
            </button>
            
            <!-- カードを削除 -->
            <button hx-delete="/api/products/a" 
                    hx-target="closest .card"
                    hx-swap="outerHTML">
                削除
            </button>
        </div>
    </div>
</body>
</html>
```

## 4.2 hx-swapの詳細

`hx-swap`は、レスポンスをターゲット要素にどのように挿入するかを制御します。

### 利用可能なswapモード

| モード | 説明 | 使用例 |
|--------|------|--------|
| innerHTML | ターゲットの内部HTMLを置換（デフォルト） | コンテンツの更新 |
| outerHTML | ターゲット要素全体を置換 | 要素の完全な置き換え |
| beforebegin | ターゲットの直前に挿入 | リストの先頭追加 |
| afterbegin | ターゲットの最初の子として挿入 | コンテナの先頭追加 |
| beforeend | ターゲットの最後の子として挿入 | コンテナの末尾追加 |
| afterend | ターゲットの直後に挿入 | リストの末尾追加 |
| delete | ターゲット要素を削除 | 要素の削除 |
| none | 何もしない | サイドエフェクトのみ |

### 各swapモードの視覚的説明

```html
<!-- innerHTML (デフォルト) -->
<div id="target">
    古いコンテンツ → 新しいコンテンツ
</div>

<!-- outerHTML -->
<div id="target">要素全体</div> → <div id="new">新しい要素</div>

<!-- beforebegin -->
→ 新しいコンテンツ
<div id="target">既存要素</div>

<!-- afterbegin -->
<div id="target">
    → 新しいコンテンツ
    既存のコンテンツ
</div>

<!-- beforeend -->
<div id="target">
    既存のコンテンツ
    → 新しいコンテンツ
</div>

<!-- afterend -->
<div id="target">既存要素</div>
→ 新しいコンテンツ
```

### 実践例：TODOリストの実装

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>hx-swapの実践例</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .todo-item {
            padding: 10px;
            margin: 5px 0;
            background: #f5f5f5;
            border-radius: 4px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .completed {
            text-decoration: line-through;
            opacity: 0.6;
        }
        #todo-form {
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <h1>TODOリスト</h1>
    
    <!-- 新規TODO追加フォーム -->
    <form id="todo-form" 
          hx-post="/api/todos" 
          hx-target="#todo-list" 
          hx-swap="beforeend">
        <input type="text" name="title" placeholder="新しいTODO" required>
        <button type="submit">追加</button>
    </form>
    
    <!-- TODOリスト -->
    <div id="todo-list">
        <div class="todo-item" id="todo-1">
            <span>買い物に行く</span>
            <div>
                <!-- 完了/未完了の切り替え -->
                <button hx-patch="/api/todos/1/toggle" 
                        hx-target="#todo-1" 
                        hx-swap="outerHTML">
                    完了
                </button>
                
                <!-- 削除 -->
                <button hx-delete="/api/todos/1" 
                        hx-target="#todo-1" 
                        hx-swap="outerHTML swap:1s">
                    削除
                </button>
            </div>
        </div>
    </div>
    
    <!-- 全削除ボタン -->
    <button hx-delete="/api/todos/all" 
            hx-target="#todo-list" 
            hx-swap="innerHTML"
            hx-confirm="すべてのTODOを削除しますか？">
        すべて削除
    </button>
</body>
</html>
```

## 4.3 swap修飾子とタイミング制御

`hx-swap`には、アニメーションやタイミングを制御する修飾子があります。

### swap修飾子の種類

```html
<!-- 遅延付きswap -->
<button hx-get="/api/data" 
        hx-target="#result" 
        hx-swap="innerHTML swap:1s">
    1秒後に更新
</button>

<!-- settleタイミングの制御 -->
<button hx-get="/api/content" 
        hx-target="#content" 
        hx-swap="innerHTML settle:500ms">
    アニメーション付き更新
</button>

<!-- スクロール動作の制御 -->
<button hx-get="/api/messages" 
        hx-target="#message-list" 
        hx-swap="beforeend scroll:bottom">
    メッセージ追加（自動スクロール）
</button>

<!-- 表示/非表示の制御 -->
<button hx-get="/api/details" 
        hx-target="#details" 
        hx-swap="innerHTML show:top">
    詳細表示（トップにスクロール）
</button>
```

### CSSトランジションとの組み合わせ

```html
<style>
    .fade-me-in.htmx-added {
        opacity: 0;
    }
    .fade-me-in {
        opacity: 1;
        transition: opacity 1s ease-out;
    }
</style>

<div hx-get="/api/content" 
     hx-target="#result" 
     hx-swap="innerHTML"
     class="fade-me-in">
    フェードインコンテンツ
</div>
```

## 4.4 高度なターゲティング技術

### 動的ターゲット選択

```html
<!-- データ属性を使った動的ターゲット -->
<button hx-get="/api/update" 
        hx-target="[data-update-target]"
        data-update-target="#dynamic-content">
    動的更新
</button>

<!-- JavaScript連携 -->
<script>
    document.addEventListener('htmx:configRequest', function(evt) {
        // リクエスト時に動的にターゲットを変更
        if (evt.detail.elt.id === 'special-button') {
            evt.detail.target = document.querySelector('#special-target');
        }
    });
</script>
```

### 複数要素の更新（OOB - Out of Band）

```html
<!-- サーバーレスポンスで複数要素を更新 -->
<button hx-get="/api/multi-update">複数更新</button>

<!-- サーバーからのレスポンス例 -->
<div id="main-content">
    メインコンテンツの更新
</div>
<div id="sidebar" hx-swap-oob="true">
    サイドバーも同時に更新
</div>
<div id="notification" hx-swap-oob="innerHTML">
    通知エリアも更新
</div>
```

## 4.5 実践的なパターン集

### パターン1: モーダルダイアログ

```html
<!-- モーダルトリガー -->
<button hx-get="/api/modal/user-edit" 
        hx-target="#modal-container" 
        hx-swap="innerHTML">
    ユーザー編集
</button>

<!-- モーダルコンテナ -->
<div id="modal-container"></div>

<!-- サーバーレスポンス例 -->
<div class="modal" id="user-edit-modal">
    <div class="modal-content">
        <h2>ユーザー編集</h2>
        <form hx-put="/api/users/1" 
              hx-target="#user-info" 
              hx-swap="outerHTML">
            <!-- フォーム内容 -->
        </form>
        <button onclick="this.closest('.modal').remove()">閉じる</button>
    </div>
</div>
```

### パターン2: インラインエディット

```html
<div class="editable-field" id="user-name">
    <span class="display-mode">
        田中太郎
        <button hx-get="/api/users/1/edit-name" 
                hx-target="#user-name" 
                hx-swap="innerHTML">
            編集
        </button>
    </span>
</div>

<!-- 編集モードのレスポンス -->
<form hx-put="/api/users/1/name" 
      hx-target="#user-name" 
      hx-swap="outerHTML">
    <input type="text" name="name" value="田中太郎">
    <button type="submit">保存</button>
    <button hx-get="/api/users/1/display-name" 
            hx-target="#user-name" 
            hx-swap="innerHTML">
        キャンセル
    </button>
</form>
```

### パターン3: プログレッシブローディング

```html
<div class="content-area">
    <!-- 基本コンテンツ -->
    <div id="basic-content">
        <h2>記事タイトル</h2>
        <p>概要...</p>
    </div>
    
    <!-- 詳細コンテンツを遅延読み込み -->
    <div hx-get="/api/article/1/details" 
         hx-trigger="revealed" 
         hx-target="this" 
         hx-swap="outerHTML">
        <div class="loading">詳細を読み込み中...</div>
    </div>
    
    <!-- コメントセクション -->
    <div hx-get="/api/article/1/comments" 
         hx-trigger="intersect once" 
         hx-target="this" 
         hx-swap="outerHTML">
        <div class="loading">コメントを読み込み中...</div>
    </div>
</div>
```

## 4.6 パフォーマンスとベストプラクティス

### 効率的なDOM更新

```html
<!-- 悪い例: 大きな要素全体を更新 -->
<div id="large-container" hx-get="/api/update-all">
    <!-- 大量のコンテンツ -->
</div>

<!-- 良い例: 必要な部分だけを更新 -->
<div id="large-container">
    <div id="update-target" hx-get="/api/update-part">
        <!-- 更新が必要な部分だけ -->
    </div>
    <!-- 変更されない部分 -->
</div>
```

### メモリリークの防止

```javascript
// HTMXイベントリスナーのクリーンアップ
document.body.addEventListener('htmx:beforeSwap', function(evt) {
    // 古い要素のイベントリスナーをクリーンアップ
    const oldElement = evt.detail.target;
    // カスタムクリーンアップロジック
});
```

## まとめ

この章では、HTMXのレスポンス処理を制御する重要な属性について学びました：

1. **hx-target**: レスポンスの挿入先を指定
   - CSSセレクタの活用
   - 特殊キーワード（this, closest, next, previous）
   
2. **hx-swap**: 挿入方法の制御
   - 様々なswapモード
   - タイミングとアニメーション制御
   
3. **高度な技術**:
   - Out of Band更新
   - 動的ターゲティング
   - 実践的なUIパターン

次章では、`hx-trigger`を使ったイベント制御について詳しく学びます。

## 練習問題

### 問題1: 基本的なターゲティング
以下の要件を満たすHTMLを作成してください：
1. ボタンクリックで自身のテキストを更新
2. 親要素の特定のクラスを持つ要素を更新
3. 隣接する要素を更新

### 問題2: 動的リスト管理
TODOリストを実装してください：
- 新規項目は先頭に追加（beforeend）
- 完了項目は見た目を変更（outerHTML）
- 削除時はフェードアウトアニメーション

### 問題3: タブインターフェース
タブ切り替えUIを実装してください：
- アクティブタブの内容だけを更新
- タブボタンのアクティブ状態も更新
- スムーズな切り替えアニメーション

### 問題4: インラインエディット
テーブルの各セルをクリックで編集可能にしてください：
- 表示モードと編集モードの切り替え
- 保存時は表示モードに戻る
- キャンセル機能付き

### 問題5: 無限スクロールギャラリー
画像ギャラリーを実装してください：
- スクロールで次の画像セットを読み込み
- 既存の画像の後に追加
- ローディング表示付き

---

**解答例とヒント：**

前章（第2章）の練習問題の解答例：

問題1の解答：
```bash
project/
├── index.html
├── js/
│   └── htmx.min.js
├── css/
│   └── style.css
└── api/
    └── server.py
```

問題2の解答：
```html
<button hx-get="/api/greeting/morning" hx-target="#message">朝の挨拶</button>
<button hx-get="/api/greeting/afternoon" hx-target="#message">昼の挨拶</button>
<button hx-get="/api/greeting/evening" hx-target="#message">夜の挨拶</button>
<div id="message"></div>
```