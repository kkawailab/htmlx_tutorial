# 第5章: トリガーとイベント（hx-trigger）

## 章の概要

この章では、HTMXの中でも最も柔軟で強力な機能の一つである`hx-trigger`について学びます。この属性を使うことで、いつHTTPリクエストを発生させるかを細かく制御でき、ユーザーインタラクションやタイミングに基づいた動的な振る舞いを実装できます。

## 5.1 hx-triggerの基本

`hx-trigger`は、HTMXリクエストを発生させるイベントを指定します。

### 基本的な構文

```html
<!-- クリックイベント（デフォルト） -->
<button hx-get="/api/data" hx-trigger="click">
    クリックでデータ取得
</button>

<!-- マウスオーバーイベント -->
<div hx-get="/api/preview" hx-trigger="mouseenter">
    マウスオーバーでプレビュー
</div>

<!-- フォーカスイベント -->
<input hx-get="/api/suggestions" hx-trigger="focus">
```

### デフォルトトリガー

要素によってデフォルトのトリガーが異なります：

| 要素 | デフォルトトリガー |
|------|-------------------|
| `<button>` | click |
| `<form>` | submit |
| `<input type="text">` | change |
| `<input type="checkbox">` | change |
| `<select>` | change |
| その他の要素 | click |

## 5.2 標準的なイベントトリガー

### マウスイベント

```html
<!-- クリック関連 -->
<div hx-get="/api/single-click" hx-trigger="click">シングルクリック</div>
<div hx-get="/api/double-click" hx-trigger="dblclick">ダブルクリック</div>
<div hx-get="/api/right-click" hx-trigger="contextmenu">右クリック</div>

<!-- マウス移動関連 -->
<div hx-get="/api/hover" hx-trigger="mouseenter">マウスエンター</div>
<div hx-get="/api/leave" hx-trigger="mouseleave">マウスリーブ</div>
<div hx-get="/api/move" hx-trigger="mousemove">マウス移動</div>

<!-- 実践例：ホバーで詳細表示 -->
<div class="product-card">
    <h3>商品名</h3>
    <div hx-get="/api/products/123/details" 
         hx-trigger="mouseenter"
         hx-target="#details-123"
         class="hover-trigger">
        詳細を見る
    </div>
    <div id="details-123"></div>
</div>
```

### キーボードイベント

```html
<!-- キー入力イベント -->
<input hx-get="/api/search" 
       hx-trigger="keyup"
       hx-target="#results"
       name="q"
       placeholder="リアルタイム検索">

<!-- 特定のキーに反応 -->
<input hx-post="/api/save" 
       hx-trigger="keyup[ctrlKey&&key=='s']"
       placeholder="Ctrl+Sで保存">

<!-- Enterキーで送信 -->
<input hx-get="/api/quick-search" 
       hx-trigger="keyup[key=='Enter']"
       placeholder="Enterで検索">
```

### フォームイベント

```html
<!-- 入力変更時 -->
<select hx-get="/api/filter" 
        hx-trigger="change"
        hx-target="#product-list"
        name="category">
    <option>すべて</option>
    <option>電化製品</option>
    <option>書籍</option>
</select>

<!-- フォーカス/ブラー -->
<input hx-get="/api/validate/email" 
       hx-trigger="blur"
       hx-target="next .error"
       type="email"
       name="email">
<span class="error"></span>

<!-- 入力中のイベント -->
<input hx-post="/api/autosave" 
       hx-trigger="input"
       hx-target="#save-status"
       name="content">
```

## 5.3 修飾子（Modifiers）

修飾子を使用してトリガーの動作を細かく制御できます。

### changed修飾子

要素の値が変更された場合のみトリガー：

```html
<!-- 値が変更された時のみ検索 -->
<input hx-get="/api/search" 
       hx-trigger="keyup changed"
       name="query">

<!-- セレクトボックスの値が変更された時 -->
<select hx-get="/api/filter" 
        hx-trigger="change changed">
    <option>選択してください</option>
    <option>オプション1</option>
    <option>オプション2</option>
</select>
```

### delay修飾子

一定時間後にトリガー：

```html
<!-- 500ms後に実行 -->
<input hx-get="/api/search" 
       hx-trigger="keyup changed delay:500ms"
       placeholder="入力後500msで検索">

<!-- 2秒後に自動保存 -->
<textarea hx-post="/api/autosave" 
          hx-trigger="keyup delay:2s"
          placeholder="2秒後に自動保存"></textarea>
```

### throttle修飾子

指定時間内に1回のみ実行：

```html
<!-- 1秒間に最大1回 -->
<div hx-get="/api/track" 
     hx-trigger="mousemove throttle:1s">
    マウストラッキング（1秒間隔）
</div>

<!-- スクロールイベントの制限 -->
<div hx-get="/api/more" 
     hx-trigger="scroll throttle:500ms">
    スクロールで追加読み込み
</div>
```

### once修飾子

一度だけ実行：

```html
<!-- 初回クリックのみ -->
<button hx-get="/api/init" 
        hx-trigger="click once">
    初期化（一度だけ実行）
</button>

<!-- 最初の表示時のみ -->
<div hx-get="/api/lazy-load" 
     hx-trigger="revealed once">
    遅延読み込みコンテンツ
</div>
```

## 5.4 特殊なトリガー

HTMXは標準的なDOMイベント以外にも特殊なトリガーを提供しています。

### load トリガー

要素が読み込まれた時に実行：

```html
<!-- ページ読み込み時に実行 -->
<div hx-get="/api/initial-data" 
     hx-trigger="load"
     hx-target="#content">
    読み込み中...
</div>
```

### revealed トリガー

要素が画面に表示された時に実行：

```html
<!-- 遅延読み込み -->
<div hx-get="/api/comments" 
     hx-trigger="revealed"
     hx-swap="outerHTML">
    <div class="placeholder">コメントを読み込む...</div>
</div>

<!-- 無限スクロール -->
<div class="post-list">
    <!-- 既存の投稿 -->
    <div hx-get="/api/posts?page=2" 
         hx-trigger="revealed"
         hx-swap="afterend"
         hx-target="this">
        さらに読み込む...
    </div>
</div>
```

### intersect トリガー

Intersection Observer APIを使用：

```html
<!-- 50%表示されたら実行 -->
<div hx-get="/api/analytics/view" 
     hx-trigger="intersect threshold:0.5">
    ビュー計測対象コンテンツ
</div>

<!-- ルート要素からのマージン指定 -->
<div hx-get="/api/preload" 
     hx-trigger="intersect root:body rootMargin:100px">
    先読みコンテンツ
</div>
```

### every トリガー

定期的に実行：

```html
<!-- 5秒ごとに更新 -->
<div hx-get="/api/notifications" 
     hx-trigger="every 5s"
     hx-target="#notification-area">
    通知エリア
</div>

<!-- 条件付き定期実行 -->
<div hx-get="/api/status" 
     hx-trigger="every 2s [isActive]"
     hx-target="#status">
    ステータス
</div>
<script>
    var isActive = true;
    // 必要に応じてisActiveを変更
</script>
```

## 5.5 複数のトリガーとカスタムイベント

### 複数トリガーの組み合わせ

```html
<!-- カンマで区切って複数指定 -->
<input hx-get="/api/search" 
       hx-trigger="keyup changed delay:500ms, blur"
       placeholder="入力中または フォーカスを外した時に検索">

<!-- 異なるイベントで異なるアクション -->
<div hx-get="/api/preview" 
     hx-trigger="mouseenter, focus"
     hx-delete="/api/clear" 
     hx-trigger="mouseleave, blur">
    ホバーまたはフォーカスでプレビュー
</div>
```

### カスタムイベント

```html
<!-- カスタムイベントをリッスン -->
<div hx-get="/api/refresh" 
     hx-trigger="custom-refresh"
     id="refreshable">
    更新可能なコンテンツ
</div>

<!-- JavaScriptからカスタムイベントを発火 -->
<script>
    function triggerRefresh() {
        htmx.trigger('#refreshable', 'custom-refresh');
    }
    
    // または
    document.getElementById('refreshable').dispatchEvent(
        new CustomEvent('custom-refresh')
    );
</script>
```

### from修飾子（他の要素のイベントをリッスン）

```html
<!-- 別の要素のイベントを監視 -->
<button id="global-refresh">全体更新</button>

<div hx-get="/api/section1" 
     hx-trigger="click from:#global-refresh">
    セクション1
</div>

<div hx-get="/api/section2" 
     hx-trigger="click from:#global-refresh">
    セクション2
</div>

<!-- ドキュメント全体のイベントを監視 -->
<div hx-get="/api/save" 
     hx-trigger="keydown[ctrlKey&&key=='s'] from:body">
    Ctrl+Sで保存
</div>
```

## 5.6 実践的な使用例

### 例1: インテリジェント検索フォーム

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>インテリジェント検索</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .search-container {
            max-width: 600px;
            margin: 50px auto;
        }
        .search-input {
            width: 100%;
            padding: 10px;
            font-size: 16px;
            border: 2px solid #ddd;
            border-radius: 4px;
        }
        .suggestions {
            border: 1px solid #ddd;
            border-top: none;
            max-height: 300px;
            overflow-y: auto;
        }
        .suggestion-item {
            padding: 10px;
            cursor: pointer;
        }
        .suggestion-item:hover {
            background-color: #f5f5f5;
        }
        .searching {
            color: #666;
            padding: 10px;
        }
    </style>
</head>
<body>
    <div class="search-container">
        <h1>スマート検索</h1>
        
        <input type="text" 
               class="search-input"
               name="q"
               placeholder="検索キーワードを入力..."
               hx-get="/api/search/suggestions"
               hx-trigger="keyup changed delay:300ms, focus"
               hx-target="#suggestions"
               hx-indicator="#search-indicator"
               autocomplete="off">
        
        <div id="search-indicator" class="htmx-indicator searching">
            検索中...
        </div>
        
        <div id="suggestions" class="suggestions"></div>
        
        <!-- 検索履歴 -->
        <div hx-get="/api/search/history" 
             hx-trigger="load"
             hx-target="#search-history">
        </div>
        <div id="search-history"></div>
    </div>
</body>
</html>
```

### 例2: リアルタイムダッシュボード

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>リアルタイムダッシュボード</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .dashboard {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 20px;
            padding: 20px;
        }
        .widget {
            border: 1px solid #ddd;
            padding: 15px;
            border-radius: 8px;
            background: white;
        }
        .metric {
            font-size: 2em;
            font-weight: bold;
            color: #333;
        }
        .status-online { color: green; }
        .status-offline { color: red; }
    </style>
</head>
<body>
    <h1>システムダッシュボード</h1>
    
    <div class="dashboard">
        <!-- アクティブユーザー数（5秒ごと更新） -->
        <div class="widget" 
             hx-get="/api/metrics/active-users" 
             hx-trigger="load, every 5s"
             hx-target="this">
            <h3>アクティブユーザー</h3>
            <div class="metric">読み込み中...</div>
        </div>
        
        <!-- サーバーステータス（3秒ごと更新） -->
        <div class="widget" 
             hx-get="/api/metrics/server-status" 
             hx-trigger="load, every 3s"
             hx-target="this">
            <h3>サーバーステータス</h3>
            <div class="status">確認中...</div>
        </div>
        
        <!-- エラーログ（新しいエラーがあれば更新） -->
        <div class="widget" 
             hx-get="/api/metrics/errors" 
             hx-trigger="load, sse:new-error"
             hx-target="#error-list">
            <h3>最新エラー</h3>
            <div id="error-list">読み込み中...</div>
        </div>
        
        <!-- パフォーマンスグラフ（マウスオーバーで詳細） -->
        <div class="widget">
            <h3>パフォーマンス</h3>
            <div hx-get="/api/metrics/performance/summary" 
                 hx-trigger="load"
                 hx-target="this">
                <div class="metric">--</div>
            </div>
            <div hx-get="/api/metrics/performance/details" 
                 hx-trigger="mouseenter once"
                 hx-target="#perf-details">
                詳細を見る
            </div>
            <div id="perf-details"></div>
        </div>
    </div>
    
    <!-- 手動更新ボタン -->
    <button onclick="htmx.trigger('.widget', 'refresh')">
        すべて更新
    </button>
    
    <script>
        // カスタムイベントで全ウィジェットを更新
        document.querySelectorAll('.widget').forEach(widget => {
            widget.setAttribute('hx-trigger', 
                widget.getAttribute('hx-trigger') + ', refresh');
        });
    </script>
</body>
</html>
```

### 例3: フォームバリデーション

```html
<form hx-post="/api/register" hx-target="#result">
    <div class="form-group">
        <label>ユーザー名</label>
        <input type="text" 
               name="username"
               hx-get="/api/validate/username"
               hx-trigger="blur changed"
               hx-target="next .error"
               required>
        <span class="error"></span>
    </div>
    
    <div class="form-group">
        <label>メールアドレス</label>
        <input type="email" 
               name="email"
               hx-get="/api/validate/email"
               hx-trigger="blur changed, keyup changed delay:1s"
               hx-target="next .error"
               required>
        <span class="error"></span>
    </div>
    
    <div class="form-group">
        <label>パスワード</label>
        <input type="password" 
               name="password"
               hx-post="/api/validate/password"
               hx-trigger="input changed throttle:500ms"
               hx-target="next .strength"
               required>
        <div class="strength"></div>
    </div>
    
    <button type="submit">登録</button>
</form>
<div id="result"></div>
```

## 5.7 パフォーマンスとベストプラクティス

### イベントリスナーの最適化

```html
<!-- 悪い例：頻繁に発火するイベント -->
<div hx-get="/api/track" hx-trigger="mousemove">
    マウストラッキング
</div>

<!-- 良い例：throttleで制限 -->
<div hx-get="/api/track" hx-trigger="mousemove throttle:100ms">
    マウストラッキング（最適化済み）
</div>
```

### 条件付きトリガー

```javascript
// グローバル変数で制御
window.shouldUpdate = true;

// 条件付きトリガー
<div hx-get="/api/data" 
     hx-trigger="every 1s [shouldUpdate]">
    条件付き自動更新
</div>

// 更新の停止/再開
function toggleUpdate() {
    window.shouldUpdate = !window.shouldUpdate;
}
```

### メモリリークの防止

```html
<!-- once修飾子を使用して一度だけ実行 -->
<div hx-get="/api/expensive-operation" 
     hx-trigger="revealed once">
    高コストな処理
</div>

<!-- 要素が削除される前にクリーンアップ -->
<script>
document.body.addEventListener('htmx:beforeSwap', function(evt) {
    // タイマーやイベントリスナーのクリーンアップ
    if (evt.detail.target.querySelector('[hx-trigger*="every"]')) {
        // everyトリガーのクリーンアップ処理
    }
});
</script>
```

## まとめ

この章では、HTMXの強力なイベント制御システムについて学びました：

1. **基本的なトリガー**: click、keyup、change などの標準イベント
2. **修飾子**: delay、throttle、once、changed による細かい制御
3. **特殊なトリガー**: load、revealed、intersect、every
4. **カスタムイベント**: 独自のイベントシステムとの統合
5. **実践的なパターン**: 検索、ダッシュボード、バリデーション

次章では、フォーム処理とバリデーションについてさらに詳しく学びます。

## 練習問題

### 問題1: 基本的なトリガー
以下の要件を満たすHTMLを作成してください：
1. マウスオーバーで詳細情報を表示
2. フォーカスを外したときにバリデーション実行
3. ダブルクリックで編集モードに切り替え

### 問題2: 遅延とスロットリング
検索フォームを実装してください：
- 入力後300msで検索開始
- 検索中は追加のリクエストを送らない
- Enterキーで即座に検索

### 問題3: 無限スクロール
商品リストに無限スクロール機能を実装してください：
- 最後の要素が見えたら次のページを読み込む
- 読み込み中の表示
- すべて読み込んだら「もうありません」と表示

### 問題4: リアルタイム更新
チャットアプリのメッセージ表示部分を実装してください：
- 2秒ごとに新着メッセージをチェック
- ユーザーが非アクティブなら更新を停止
- 新着メッセージは既存メッセージの後に追加

### 問題5: カスタムイベント統合
以下の機能を持つコンポーネントを作成してください：
- 他のコンポーネントから`update-request`イベントを受け取る
- 更新完了後に`update-complete`イベントを発火
- エラー時は`update-error`イベントを発火

---

**解答例とヒント：**

前章（第3章）の練習問題の解答例：

問題1の解答：
```html
<!-- 商品一覧表示 -->
<button hx-get="/api/products" hx-target="#product-list">商品一覧</button>
<div id="product-list"></div>

<!-- 新商品追加フォーム -->
<form hx-post="/api/products" 
      hx-target="#product-list" 
      hx-swap="beforeend">
    <input name="name" placeholder="商品名" required>
    <input name="price" type="number" placeholder="価格" required>
    <button type="submit">追加</button>
</form>
```

問題2の解答：
```html
<input type="search" 
       name="q"
       hx-get="/api/search"
       hx-trigger="keyup changed delay:500ms"
       hx-target="#results"
       hx-indicator="#loading"
       placeholder="検索...">
<span id="loading" class="htmx-indicator">検索中...</span>
<div id="results"></div>
```