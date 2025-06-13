# 第6章: フォームとバリデーション

## 章の概要

この章では、HTMXを使用したフォーム処理とバリデーションについて詳しく学びます。クライアントサイドとサーバーサイドの両方でのバリデーション実装、エラーハンドリング、そしてユーザーフレンドリーなフィードバックの提供方法を習得します。

## 6.1 HTMXでのフォーム基礎

### 基本的なフォーム送信

```html
<!-- 基本的なフォーム -->
<form hx-post="/api/contact" 
      hx-target="#result">
    <input type="text" name="name" placeholder="名前" required>
    <input type="email" name="email" placeholder="メール" required>
    <textarea name="message" placeholder="メッセージ" required></textarea>
    <button type="submit">送信</button>
</form>
<div id="result"></div>

<!-- フォーム送信後のリセット -->
<form hx-post="/api/submit" 
      hx-target="#result"
      hx-on="htmx:afterRequest: if(event.detail.successful) this.reset()">
    <!-- フォーム要素 -->
</form>
```

### フォームデータの拡張

```html
<!-- hx-includeで追加データを含める -->
<input type="hidden" id="session-token" value="abc123">

<form hx-post="/api/secure-submit" 
      hx-include="#session-token">
    <input type="text" name="data">
    <button type="submit">送信</button>
</form>

<!-- hx-valsで固定値を追加 -->
<form hx-post="/api/submit" 
      hx-vals='{"source": "web", "version": "1.0"}'>
    <input type="text" name="user_input">
    <button type="submit">送信</button>
</form>
```

## 6.2 リアルタイムバリデーション

### フィールドレベルのバリデーション

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>リアルタイムバリデーション</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .form-group {
            margin-bottom: 20px;
        }
        .form-control {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .form-control.error {
            border-color: #dc3545;
        }
        .form-control.success {
            border-color: #28a745;
        }
        .feedback {
            margin-top: 5px;
            font-size: 14px;
        }
        .error-message {
            color: #dc3545;
        }
        .success-message {
            color: #28a745;
        }
        .hint {
            color: #6c757d;
            font-size: 12px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>ユーザー登録フォーム</h1>
        
        <form hx-post="/api/register" hx-target="#form-result">
            <!-- ユーザー名 -->
            <div class="form-group">
                <label>ユーザー名</label>
                <input type="text" 
                       name="username" 
                       class="form-control"
                       hx-post="/api/validate/username"
                       hx-trigger="blur changed, keyup changed delay:500ms"
                       hx-target="next .feedback"
                       hx-indicator="next .checking"
                       required>
                <div class="feedback">
                    <span class="checking htmx-indicator">確認中...</span>
                </div>
                <div class="hint">3文字以上、英数字のみ</div>
            </div>
            
            <!-- メールアドレス -->
            <div class="form-group">
                <label>メールアドレス</label>
                <input type="email" 
                       name="email" 
                       class="form-control"
                       hx-post="/api/validate/email"
                       hx-trigger="blur changed"
                       hx-target="next .feedback"
                       required>
                <div class="feedback"></div>
            </div>
            
            <!-- パスワード -->
            <div class="form-group">
                <label>パスワード</label>
                <input type="password" 
                       name="password" 
                       id="password"
                       class="form-control"
                       hx-post="/api/validate/password"
                       hx-trigger="input changed throttle:500ms"
                       hx-target="next .feedback"
                       required>
                <div class="feedback"></div>
                <div class="hint">8文字以上、大文字・小文字・数字を含む</div>
            </div>
            
            <!-- パスワード確認 -->
            <div class="form-group">
                <label>パスワード（確認）</label>
                <input type="password" 
                       name="password_confirm" 
                       class="form-control"
                       hx-post="/api/validate/password-confirm"
                       hx-trigger="blur changed, keyup changed delay:500ms"
                       hx-target="next .feedback"
                       hx-vals='js:{password: document.getElementById("password").value}'
                       required>
                <div class="feedback"></div>
            </div>
            
            <button type="submit" class="btn-submit">登録</button>
        </form>
        
        <div id="form-result"></div>
    </div>
</body>
</html>
```

### サーバーサイドバリデーション（Flask例）

```python
from flask import Flask, request, jsonify
import re

app = Flask(__name__)

@app.route('/api/validate/username', methods=['POST'])
def validate_username():
    username = request.form.get('username', '')
    
    # バリデーションルール
    if len(username) < 3:
        return '''
        <div class="error-message">
            ユーザー名は3文字以上必要です
        </div>
        ''', 400
    
    if not re.match('^[a-zA-Z0-9]+$', username):
        return '''
        <div class="error-message">
            英数字のみ使用可能です
        </div>
        ''', 400
    
    # データベースで重複チェック（仮想）
    if username.lower() in ['admin', 'user', 'test']:
        return '''
        <div class="error-message">
            このユーザー名は既に使用されています
        </div>
        ''', 400
    
    return '''
    <div class="success-message">
        ✓ 利用可能なユーザー名です
    </div>
    '''

@app.route('/api/validate/password', methods=['POST'])
def validate_password():
    password = request.form.get('password', '')
    strength = 0
    feedback = []
    
    if len(password) >= 8:
        strength += 25
    else:
        feedback.append("8文字以上必要")
    
    if re.search(r'[A-Z]', password):
        strength += 25
    else:
        feedback.append("大文字を含める")
    
    if re.search(r'[a-z]', password):
        strength += 25
    else:
        feedback.append("小文字を含める")
    
    if re.search(r'[0-9]', password):
        strength += 25
    else:
        feedback.append("数字を含める")
    
    # 強度メーター
    if strength == 100:
        color = "green"
        message = "強い"
    elif strength >= 75:
        color = "yellow"
        message = "普通"
    else:
        color = "red"
        message = "弱い"
    
    return f'''
    <div class="password-strength">
        <div class="strength-meter" style="width: {strength}%; background: {color}"></div>
        <span>パスワード強度: {message}</span>
        {f'<ul class="feedback-list">{"".join(f"<li>{f}</li>" for f in feedback)}</ul>' if feedback else ''}
    </div>
    '''
```

## 6.3 フォームの状態管理

### 送信中の状態表示

```html
<!-- hx-indicatorを使った表示 -->
<form hx-post="/api/submit" 
      hx-indicator="#submit-indicator">
    <input type="text" name="data">
    <button type="submit">
        送信
        <span id="submit-indicator" class="htmx-indicator">
            処理中...
        </span>
    </button>
</form>

<!-- ボタンの無効化 -->
<form hx-post="/api/submit"
      hx-on="htmx:beforeRequest: this.querySelector('button').disabled = true"
      hx-on="htmx:afterRequest: this.querySelector('button').disabled = false">
    <button type="submit">送信</button>
</form>
```

### プログレス表示付きファイルアップロード

```html
<form hx-post="/api/upload" 
      hx-encoding="multipart/form-data"
      hx-indicator="#upload-progress">
    <input type="file" name="file" accept="image/*" required>
    <button type="submit">アップロード</button>
    
    <div id="upload-progress" class="htmx-indicator">
        <div class="progress-bar">
            <div class="progress-fill"></div>
        </div>
        <span>アップロード中...</span>
    </div>
</form>

<script>
htmx.on('htmx:xhr:progress', function(evt) {
    if (evt.detail.lengthComputable) {
        const percentComplete = (evt.detail.loaded / evt.detail.total) * 100;
        document.querySelector('.progress-fill').style.width = percentComplete + '%';
    }
});
</script>
```

## 6.4 複雑なフォームパターン

### 動的フォームフィールド

```html
<!-- 動的にフィールドを追加 -->
<form id="dynamic-form" hx-post="/api/submit">
    <div id="field-container">
        <div class="field-group">
            <input type="text" name="items[]" placeholder="アイテム1">
            <button type="button" onclick="removeField(this)">削除</button>
        </div>
    </div>
    
    <button type="button" 
            hx-get="/api/form/new-field"
            hx-target="#field-container"
            hx-swap="beforeend">
        フィールド追加
    </button>
    
    <button type="submit">送信</button>
</form>

<!-- サーバーレスポンス（新規フィールド） -->
<div class="field-group">
    <input type="text" name="items[]" placeholder="新しいアイテム">
    <button type="button" onclick="removeField(this)">削除</button>
</div>
```

### 条件付きフィールド表示

```html
<form hx-post="/api/order">
    <div class="form-group">
        <label>配送方法</label>
        <select name="shipping_method" 
                hx-get="/api/form/shipping-details"
                hx-trigger="change"
                hx-target="#shipping-details">
            <option value="">選択してください</option>
            <option value="standard">通常配送</option>
            <option value="express">速達</option>
            <option value="pickup">店舗受取</option>
        </select>
    </div>
    
    <div id="shipping-details">
        <!-- 配送方法に応じた追加フィールドが表示される -->
    </div>
    
    <button type="submit">注文確定</button>
</form>
```

### マルチステップフォーム

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>マルチステップフォーム</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .wizard-container {
            max-width: 600px;
            margin: 0 auto;
        }
        .step-indicator {
            display: flex;
            justify-content: space-between;
            margin-bottom: 30px;
        }
        .step {
            flex: 1;
            text-align: center;
            padding: 10px;
            background: #f0f0f0;
            position: relative;
        }
        .step.active {
            background: #007bff;
            color: white;
        }
        .step.completed {
            background: #28a745;
            color: white;
        }
        .step-content {
            min-height: 200px;
        }
        .nav-buttons {
            display: flex;
            justify-content: space-between;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <div class="wizard-container">
        <h1>会員登録ウィザード</h1>
        
        <!-- ステップインジケーター -->
        <div class="step-indicator">
            <div class="step active" id="step-1-indicator">基本情報</div>
            <div class="step" id="step-2-indicator">詳細情報</div>
            <div class="step" id="step-3-indicator">確認</div>
        </div>
        
        <!-- フォームコンテンツ -->
        <form id="wizard-form" hx-post="/api/register/complete">
            <div class="step-content" id="step-content">
                <!-- 初期ステップ -->
                <div hx-get="/api/register/step/1" 
                     hx-trigger="load"
                     hx-target="#step-content">
                    読み込み中...
                </div>
            </div>
        </form>
    </div>
</body>
</html>

<!-- ステップ1のコンテンツ例 -->
<div class="step-1">
    <h2>基本情報</h2>
    <div class="form-group">
        <label>名前</label>
        <input type="text" name="name" required>
    </div>
    <div class="form-group">
        <label>メールアドレス</label>
        <input type="email" name="email" required>
    </div>
    
    <div class="nav-buttons">
        <button type="button" disabled>戻る</button>
        <button type="button" 
                hx-post="/api/register/validate/1"
                hx-target="#step-content"
                hx-vals='js:{data: collectFormData()}'>
            次へ
        </button>
    </div>
</div>
```

## 6.5 エラーハンドリングとフィードバック

### グローバルエラーハンドリング

```html
<script>
// HTMXのグローバルエラーハンドラー
document.body.addEventListener('htmx:responseError', function(evt) {
    const xhr = evt.detail.xhr;
    const target = evt.detail.target;
    
    // ステータスコードに応じた処理
    switch(xhr.status) {
        case 400:
            // バリデーションエラー
            const errors = JSON.parse(xhr.responseText);
            displayValidationErrors(errors);
            break;
        case 401:
            // 認証エラー
            showNotification('ログインが必要です', 'error');
            window.location.href = '/login';
            break;
        case 500:
            // サーバーエラー
            showNotification('サーバーエラーが発生しました', 'error');
            break;
        default:
            showNotification('エラーが発生しました', 'error');
    }
});

function displayValidationErrors(errors) {
    // フィールドごとのエラー表示
    Object.keys(errors).forEach(field => {
        const input = document.querySelector(`[name="${field}"]`);
        if (input) {
            input.classList.add('error');
            const errorDiv = input.parentElement.querySelector('.error-message');
            if (errorDiv) {
                errorDiv.textContent = errors[field];
            }
        }
    });
}
</script>
```

### ユーザーフレンドリーなフィードバック

```html
<!-- トースト通知 -->
<div id="toast-container"></div>

<script>
function showToast(message, type = 'info') {
    const toast = document.createElement('div');
    toast.className = `toast toast-${type}`;
    toast.textContent = message;
    
    document.getElementById('toast-container').appendChild(toast);
    
    // アニメーション
    setTimeout(() => toast.classList.add('show'), 10);
    
    // 自動的に削除
    setTimeout(() => {
        toast.classList.remove('show');
        setTimeout(() => toast.remove(), 300);
    }, 3000);
}

// HTMXイベントでトースト表示
document.body.addEventListener('htmx:afterRequest', function(evt) {
    if (evt.detail.successful) {
        const response = evt.detail.xhr.response;
        if (response.includes('success')) {
            showToast('操作が完了しました', 'success');
        }
    }
});
</script>

<style>
.toast {
    position: fixed;
    top: 20px;
    right: 20px;
    padding: 15px 20px;
    background: #333;
    color: white;
    border-radius: 4px;
    opacity: 0;
    transition: opacity 0.3s;
    z-index: 1000;
}
.toast.show {
    opacity: 1;
}
.toast-success {
    background: #28a745;
}
.toast-error {
    background: #dc3545;
}
</style>
```

## 6.6 フォームのベストプラクティス

### アクセシビリティ

```html
<!-- アクセシブルなフォーム -->
<form hx-post="/api/submit">
    <fieldset>
        <legend>個人情報</legend>
        
        <div class="form-group">
            <label for="name">
                名前 <span aria-label="必須">*</span>
            </label>
            <input type="text" 
                   id="name" 
                   name="name" 
                   required
                   aria-required="true"
                   aria-describedby="name-error">
            <div id="name-error" class="error-message" role="alert"></div>
        </div>
    </fieldset>
    
    <button type="submit">送信</button>
</form>
```

### セキュリティ対策

```html
<!-- CSRF保護 -->
<meta name="csrf-token" content="{{ csrf_token }}">

<script>
document.body.addEventListener('htmx:configRequest', (event) => {
    event.detail.headers['X-CSRF-Token'] = 
        document.querySelector('meta[name="csrf-token"]').content;
});
</script>

<!-- XSS対策 -->
<form hx-post="/api/comment" 
      hx-target="#comments"
      hx-swap="beforeend">
    <textarea name="comment" 
              maxlength="500"
              placeholder="コメントを入力..."></textarea>
    <button type="submit">投稿</button>
</form>
```

### パフォーマンス最適化

```html
<!-- デバウンスとスロットリング -->
<input type="text" 
       hx-post="/api/search"
       hx-trigger="keyup changed delay:300ms"
       hx-target="#results">

<!-- 条件付きバリデーション -->
<input type="email" 
       hx-post="/api/validate/email"
       hx-trigger="blur changed[this.value.length > 3]"
       hx-target="next .error">
```

## まとめ

この章では、HTMXを使用したフォーム処理とバリデーションについて学びました：

1. **基本的なフォーム処理**: 送信、データ拡張、状態管理
2. **リアルタイムバリデーション**: クライアントサイドとサーバーサイドの連携
3. **複雑なフォームパターン**: 動的フィールド、条件付き表示、マルチステップ
4. **エラーハンドリング**: グローバルハンドラー、ユーザーフィードバック
5. **ベストプラクティス**: アクセシビリティ、セキュリティ、パフォーマンス

次章では、ローディング表示やエラーハンドリングなど、より高度なUIパターンについて学びます。

## 練習問題

### 問題1: 基本的なフォームバリデーション
以下の要件を満たす会員登録フォームを作成してください：
- メールアドレスの形式チェック
- パスワードの強度表示
- 利用規約への同意チェックボックス
- すべての条件を満たした時のみ送信ボタンを有効化

### 問題2: 動的フォーム
商品注文フォームを作成してください：
- 商品を動的に追加/削除できる
- 各商品の数量を変更すると合計金額が自動更新
- 在庫チェック機能付き

### 問題3: ファイルアップロード
画像アップロードフォームを作成してください：
- ドラッグ&ドロップ対応
- プレビュー表示
- アップロード進捗表示
- ファイルサイズとタイプの検証

### 問題4: マルチステップフォーム
3ステップの申し込みフォームを作成してください：
- 各ステップでのバリデーション
- 前のステップに戻れる
- 進捗インジケーター
- 最終確認画面

### 問題5: エラーリカバリー
ネットワークエラーに対応したフォームを作成してください：
- オフライン時のエラー表示
- 自動リトライ機能
- ローカルストレージへの一時保存
- 接続復帰時の自動送信

---

**解答例とヒント：**

前章（第4章）の練習問題の解答例：

問題1の解答：
```html
<!-- 自身のテキストを更新 -->
<button hx-get="/api/self-update" hx-target="this">
    クリックで更新
</button>

<!-- 親要素の更新 -->
<div class="container">
    <button hx-get="/api/parent-update" 
            hx-target="closest .container">
        親を更新
    </button>
</div>

<!-- 隣接要素の更新 -->
<button hx-get="/api/next-update" 
        hx-target="next .content">
    次を更新
</button>
<div class="content">更新される内容</div>
```