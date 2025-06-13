# 第9章: データ永続化の方法

## 章の概要

この章では、HTMXアプリケーションにおけるデータ永続化の様々な方法について学びます。ローカルストレージ、セッション管理、データベース連携、キャッシュ戦略など、実践的なデータ管理手法を習得します。

## 9.1 データ永続化の概要

### なぜデータ永続化が重要か

HTMXアプリケーションでは、サーバーサイドレンダリングが中心となりますが、以下の場面でクライアント側のデータ永続化も重要になります：

1. **オフライン対応**: ネットワーク接続がない状態でのデータ保持
2. **パフォーマンス向上**: 頻繁にアクセスするデータのキャッシュ
3. **ユーザー体験の向上**: フォームの一時保存、設定の記憶
4. **セッション管理**: ログイン状態の維持

### 利用可能な永続化手法

| 手法 | 容量 | 永続性 | 用途 |
|------|------|---------|------|
| LocalStorage | 5-10MB | 永続的 | 設定、小規模データ |
| SessionStorage | 5-10MB | セッション中 | 一時的なデータ |
| IndexedDB | 制限なし* | 永続的 | 大規模データ、オフライン |
| Cookies | 4KB | 設定可能 | 認証、小さな設定 |
| Cache API | 制限なし* | 永続的 | リソースキャッシュ |

*ブラウザとデバイスの空き容量に依存

## 9.2 LocalStorageとSessionStorage

### 基本的な使い方

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>LocalStorage with HTMX</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
</head>
<body>
    <h1>ユーザー設定</h1>
    
    <!-- テーマ設定 -->
    <div class="theme-selector">
        <h2>テーマ選択</h2>
        <select id="theme-select" 
                hx-post="/api/settings/theme"
                hx-trigger="change"
                hx-on="htmx:afterRequest: saveThemeLocally(this.value)">
            <option value="light">ライト</option>
            <option value="dark">ダーク</option>
            <option value="auto">自動</option>
        </select>
    </div>
    
    <!-- フォームの自動保存 -->
    <form id="draft-form" 
          hx-post="/api/posts/create"
          hx-on="htmx:beforeRequest: clearDraft()">
        <h2>新規投稿</h2>
        <input type="text" 
               name="title" 
               placeholder="タイトル"
               oninput="saveDraft()">
        <textarea name="content" 
                  placeholder="内容"
                  oninput="saveDraft()"></textarea>
        <button type="submit">投稿</button>
        <button type="button" onclick="loadDraft()">下書きを復元</button>
    </form>
    
    <script>
    // ページ読み込み時に設定を復元
    document.addEventListener('DOMContentLoaded', function() {
        // テーマ設定の復元
        const savedTheme = localStorage.getItem('theme');
        if (savedTheme) {
            document.getElementById('theme-select').value = savedTheme;
            applyTheme(savedTheme);
        }
        
        // 下書きの存在確認
        if (localStorage.getItem('draftPost')) {
            if (confirm('保存された下書きがあります。復元しますか？')) {
                loadDraft();
            }
        }
    });
    
    // テーマをローカルに保存
    function saveThemeLocally(theme) {
        localStorage.setItem('theme', theme);
        applyTheme(theme);
    }
    
    // テーマを適用
    function applyTheme(theme) {
        document.body.className = `theme-${theme}`;
    }
    
    // 下書き保存（デバウンス付き）
    let draftTimer;
    function saveDraft() {
        clearTimeout(draftTimer);
        draftTimer = setTimeout(() => {
            const form = document.getElementById('draft-form');
            const formData = new FormData(form);
            const draft = {
                title: formData.get('title'),
                content: formData.get('content'),
                savedAt: new Date().toISOString()
            };
            localStorage.setItem('draftPost', JSON.stringify(draft));
            showNotification('下書きを保存しました');
        }, 1000);
    }
    
    // 下書き読み込み
    function loadDraft() {
        const draftString = localStorage.getItem('draftPost');
        if (draftString) {
            const draft = JSON.parse(draftString);
            document.querySelector('[name="title"]').value = draft.title || '';
            document.querySelector('[name="content"]').value = draft.content || '';
            showNotification('下書きを復元しました');
        }
    }
    
    // 下書きクリア
    function clearDraft() {
        localStorage.removeItem('draftPost');
    }
    
    // 通知表示
    function showNotification(message) {
        const notification = document.createElement('div');
        notification.className = 'notification';
        notification.textContent = message;
        document.body.appendChild(notification);
        setTimeout(() => notification.remove(), 3000);
    }
    </script>
    
    <style>
    .notification {
        position: fixed;
        bottom: 20px;
        right: 20px;
        background: #4CAF50;
        color: white;
        padding: 10px 20px;
        border-radius: 4px;
        animation: slideIn 0.3s ease;
    }
    
    @keyframes slideIn {
        from { transform: translateX(100%); }
        to { transform: translateX(0); }
    }
    
    /* テーマ */
    .theme-dark {
        background: #1a1a1a;
        color: #ffffff;
    }
    
    .theme-dark input,
    .theme-dark textarea,
    .theme-dark select {
        background: #2a2a2a;
        color: #ffffff;
        border-color: #444;
    }
    </style>
</body>
</html>
```

### セッションストレージの活用

```html
<!-- タブ間で共有されない一時データ -->
<div class="wizard">
    <h2>登録ウィザード</h2>
    
    <!-- ステップ1 -->
    <div id="step1" class="step">
        <input type="text" 
               name="name" 
               placeholder="名前"
               oninput="saveWizardData('step1')">
        <button hx-get="/api/wizard/step2"
                hx-target="#wizard-content"
                onclick="saveWizardData('step1')">
            次へ
        </button>
    </div>
</div>

<script>
// ウィザードデータの保存
function saveWizardData(step) {
    const wizardData = JSON.parse(sessionStorage.getItem('wizardData') || '{}');
    const stepElement = document.getElementById(step);
    const inputs = stepElement.querySelectorAll('input, select, textarea');
    
    wizardData[step] = {};
    inputs.forEach(input => {
        wizardData[step][input.name] = input.value;
    });
    
    sessionStorage.setItem('wizardData', JSON.stringify(wizardData));
}

// ウィザードデータの復元
function loadWizardData(step) {
    const wizardData = JSON.parse(sessionStorage.getItem('wizardData') || '{}');
    const stepData = wizardData[step] || {};
    
    Object.keys(stepData).forEach(name => {
        const input = document.querySelector(`[name="${name}"]`);
        if (input) {
            input.value = stepData[name];
        }
    });
}

// ページ遷移時にデータクリア
window.addEventListener('beforeunload', function(e) {
    // 完了していない場合は警告
    const wizardData = sessionStorage.getItem('wizardData');
    if (wizardData && !sessionStorage.getItem('wizardCompleted')) {
        e.preventDefault();
        e.returnValue = '入力中のデータが失われます。';
    }
});
</script>
```

## 9.3 IndexedDBによる大規模データ管理

### IndexedDBの基本操作

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>IndexedDB with HTMX</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
</head>
<body>
    <h1>オフライン対応アプリケーション</h1>
    
    <!-- 商品リスト -->
    <div id="product-list" 
         hx-get="/api/products"
         hx-trigger="load"
         hx-on="htmx:afterRequest: cacheProducts(event)">
        読み込み中...
    </div>
    
    <!-- オフライン表示 -->
    <div id="offline-indicator" style="display: none;">
        <p>オフラインモード - キャッシュされたデータを表示しています</p>
    </div>
    
    <script>
    // IndexedDBの初期化
    let db;
    const DB_NAME = 'HTMXAppDB';
    const DB_VERSION = 1;
    
    function initDB() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(DB_NAME, DB_VERSION);
            
            request.onerror = () => reject(request.error);
            request.onsuccess = () => {
                db = request.result;
                resolve(db);
            };
            
            request.onupgradeneeded = (event) => {
                db = event.target.result;
                
                // オブジェクトストアの作成
                if (!db.objectStoreNames.contains('products')) {
                    const productStore = db.createObjectStore('products', { keyPath: 'id' });
                    productStore.createIndex('category', 'category', { unique: false });
                    productStore.createIndex('updatedAt', 'updatedAt', { unique: false });
                }
                
                if (!db.objectStoreNames.contains('offlineQueue')) {
                    db.createObjectStore('offlineQueue', { keyPath: 'id', autoIncrement: true });
                }
                
                if (!db.objectStoreNames.contains('cache')) {
                    const cacheStore = db.createObjectStore('cache', { keyPath: 'url' });
                    cacheStore.createIndex('timestamp', 'timestamp', { unique: false });
                }
            };
        });
    }
    
    // 商品データのキャッシュ
    async function cacheProducts(event) {
        if (!event.detail.successful) return;
        
        try {
            await initDB();
            
            // レスポンスからデータを抽出（実際の実装では適切にパース）
            const products = parseProductsFromHTML(event.detail.xhr.response);
            
            const transaction = db.transaction(['products'], 'readwrite');
            const store = transaction.objectStore('products');
            
            // 既存データをクリア
            await store.clear();
            
            // 新しいデータを保存
            for (const product of products) {
                product.updatedAt = new Date().toISOString();
                await store.add(product);
            }
            
            console.log('商品データをキャッシュしました');
        } catch (error) {
            console.error('キャッシュエラー:', error);
        }
    }
    
    // オフライン時のデータ取得
    async function loadOfflineProducts() {
        try {
            await initDB();
            
            const transaction = db.transaction(['products'], 'readonly');
            const store = transaction.objectStore('products');
            const products = await store.getAll();
            
            // HTMLとして表示
            displayProducts(products);
            document.getElementById('offline-indicator').style.display = 'block';
        } catch (error) {
            console.error('オフラインデータ取得エラー:', error);
        }
    }
    
    // オフラインキューの管理
    async function queueOfflineRequest(method, url, data) {
        try {
            await initDB();
            
            const transaction = db.transaction(['offlineQueue'], 'readwrite');
            const store = transaction.objectStore('offlineQueue');
            
            await store.add({
                method: method,
                url: url,
                data: data,
                timestamp: new Date().toISOString()
            });
            
            showNotification('オフライン: 操作はオンライン復帰時に実行されます');
        } catch (error) {
            console.error('オフラインキュー追加エラー:', error);
        }
    }
    
    // オンライン復帰時の処理
    async function processOfflineQueue() {
        try {
            await initDB();
            
            const transaction = db.transaction(['offlineQueue'], 'readwrite');
            const store = transaction.objectStore('offlineQueue');
            const requests = await store.getAll();
            
            for (const request of requests) {
                try {
                    // キューに入れられたリクエストを実行
                    await fetch(request.url, {
                        method: request.method,
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(request.data)
                    });
                    
                    // 成功したら削除
                    await store.delete(request.id);
                } catch (error) {
                    console.error('オフラインリクエスト実行エラー:', error);
                }
            }
            
            showNotification('オフライン中の操作を同期しました');
        } catch (error) {
            console.error('オフラインキュー処理エラー:', error);
        }
    }
    
    // オンライン/オフライン検知
    window.addEventListener('online', () => {
        processOfflineQueue();
        htmx.trigger('#product-list', 'load');
    });
    
    window.addEventListener('offline', () => {
        loadOfflineProducts();
    });
    
    // HTMXリクエストのインターセプト
    document.body.addEventListener('htmx:beforeRequest', function(evt) {
        if (!navigator.onLine) {
            evt.preventDefault();
            
            // オフラインキューに追加
            const config = evt.detail.requestConfig;
            queueOfflineRequest(config.verb, config.path, config.parameters);
        }
    });
    </script>
</body>
</html>
```

### 高度なIndexedDB活用

```javascript
// データベースヘルパークラス
class HTMXDatabase {
    constructor() {
        this.db = null;
        this.dbName = 'HTMXAppDB';
        this.version = 1;
    }
    
    async init() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(this.dbName, this.version);
            
            request.onsuccess = (event) => {
                this.db = event.target.result;
                resolve(this);
            };
            
            request.onerror = () => reject(request.error);
            
            request.onupgradeneeded = (event) => {
                const db = event.target.result;
                this.createStores(db);
            };
        });
    }
    
    createStores(db) {
        // メインデータストア
        if (!db.objectStoreNames.contains('entities')) {
            const entityStore = db.createObjectStore('entities', { keyPath: 'id' });
            entityStore.createIndex('type', 'type', { unique: false });
            entityStore.createIndex('timestamp', 'timestamp', { unique: false });
            entityStore.createIndex('type_timestamp', ['type', 'timestamp'], { unique: false });
        }
        
        // メタデータストア
        if (!db.objectStoreNames.contains('metadata')) {
            db.createObjectStore('metadata', { keyPath: 'key' });
        }
        
        // 同期状態ストア
        if (!db.objectStoreNames.contains('syncStatus')) {
            const syncStore = db.createObjectStore('syncStatus', { keyPath: 'id' });
            syncStore.createIndex('status', 'status', { unique: false });
            syncStore.createIndex('retryCount', 'retryCount', { unique: false });
        }
    }
    
    // データの保存（型を指定）
    async save(type, data) {
        const entity = {
            ...data,
            type: type,
            timestamp: new Date().toISOString(),
            syncStatus: 'pending'
        };
        
        const transaction = this.db.transaction(['entities'], 'readwrite');
        const store = transaction.objectStore('entities');
        
        return new Promise((resolve, reject) => {
            const request = store.put(entity);
            request.onsuccess = () => resolve(entity);
            request.onerror = () => reject(request.error);
        });
    }
    
    // データの取得（型とフィルタ条件）
    async find(type, filter = {}) {
        const transaction = this.db.transaction(['entities'], 'readonly');
        const store = transaction.objectStore('entities');
        const index = store.index('type');
        
        return new Promise((resolve, reject) => {
            const request = index.getAll(type);
            request.onsuccess = () => {
                let results = request.result;
                
                // フィルタ適用
                if (filter.where) {
                    results = results.filter(item => {
                        return Object.entries(filter.where).every(([key, value]) => {
                            return item[key] === value;
                        });
                    });
                }
                
                // ソート
                if (filter.orderBy) {
                    results.sort((a, b) => {
                        const aVal = a[filter.orderBy];
                        const bVal = b[filter.orderBy];
                        return filter.order === 'desc' ? bVal - aVal : aVal - bVal;
                    });
                }
                
                // リミット
                if (filter.limit) {
                    results = results.slice(0, filter.limit);
                }
                
                resolve(results);
            };
            request.onerror = () => reject(request.error);
        });
    }
    
    // バッチ操作
    async batchSave(type, items) {
        const transaction = this.db.transaction(['entities'], 'readwrite');
        const store = transaction.objectStore('entities');
        
        const promises = items.map(item => {
            return new Promise((resolve, reject) => {
                const entity = {
                    ...item,
                    type: type,
                    timestamp: new Date().toISOString()
                };
                
                const request = store.put(entity);
                request.onsuccess = () => resolve(entity);
                request.onerror = () => reject(request.error);
            });
        });
        
        return Promise.all(promises);
    }
    
    // データの削除
    async delete(id) {
        const transaction = this.db.transaction(['entities'], 'readwrite');
        const store = transaction.objectStore('entities');
        
        return new Promise((resolve, reject) => {
            const request = store.delete(id);
            request.onsuccess = () => resolve();
            request.onerror = () => reject(request.error);
        });
    }
    
    // データのクリア
    async clear(type = null) {
        if (!type) {
            // 全データクリア
            const transaction = this.db.transaction(['entities'], 'readwrite');
            const store = transaction.objectStore('entities');
            return store.clear();
        }
        
        // 特定タイプのデータをクリア
        const items = await this.find(type);
        const transaction = this.db.transaction(['entities'], 'readwrite');
        const store = transaction.objectStore('entities');
        
        const promises = items.map(item => {
            return new Promise((resolve, reject) => {
                const request = store.delete(item.id);
                request.onsuccess = () => resolve();
                request.onerror = () => reject(request.error);
            });
        });
        
        return Promise.all(promises);
    }
}

// 使用例
const db = new HTMXDatabase();

// 初期化とデータ操作
async function initializeApp() {
    await db.init();
    
    // データの保存
    await db.save('product', {
        id: 'prod-1',
        name: '商品A',
        price: 1000,
        category: '電化製品'
    });
    
    // データの取得
    const products = await db.find('product', {
        where: { category: '電化製品' },
        orderBy: 'price',
        order: 'asc',
        limit: 10
    });
    
    console.log('取得した商品:', products);
}
```

## 9.4 サーバー側のセッション管理

### Cookieベースのセッション

```python
# Flask でのセッション管理例
from flask import Flask, session, request, make_response
from datetime import datetime, timedelta
import secrets

app = Flask(__name__)
app.secret_key = secrets.token_hex(16)

# セッション設定
app.config['SESSION_COOKIE_SECURE'] = True  # HTTPS only
app.config['SESSION_COOKIE_HTTPONLY'] = True  # XSS対策
app.config['SESSION_COOKIE_SAMESITE'] = 'Lax'  # CSRF対策
app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(hours=24)

@app.route('/api/login', methods=['POST'])
def login():
    username = request.form.get('username')
    password = request.form.get('password')
    
    # 認証処理（実際の実装では適切な認証を行う）
    if authenticate_user(username, password):
        # セッションにユーザー情報を保存
        session['user_id'] = get_user_id(username)
        session['username'] = username
        session['login_time'] = datetime.now().isoformat()
        session.permanent = True
        
        # カスタムクッキーの設定
        response = make_response(render_user_dashboard())
        response.set_cookie(
            'user_preferences',
            value=json.dumps(get_user_preferences(username)),
            max_age=30*24*60*60,  # 30日
            secure=True,
            httponly=False  # JavaScriptからアクセス可能
        )
        
        return response
    
    return 'ログイン失敗', 401

@app.route('/api/session/extend', methods=['POST'])
def extend_session():
    if 'user_id' in session:
        session.permanent = True
        session['last_activity'] = datetime.now().isoformat()
        return 'セッションを延長しました'
    return 'セッションが見つかりません', 401

@app.route('/api/logout', methods=['POST'])
def logout():
    # セッションクリア
    session.clear()
    
    response = make_response('ログアウトしました')
    # クッキーも削除
    response.set_cookie('user_preferences', '', expires=0)
    
    return response

# セッション情報の取得
@app.route('/api/session/info')
def session_info():
    if 'user_id' not in session:
        return 'ログインしていません', 401
    
    return f'''
    <div class="session-info">
        <p>ユーザー: {session.get('username')}</p>
        <p>ログイン時刻: {session.get('login_time')}</p>
        <p>最終アクティビティ: {session.get('last_activity', 'N/A')}</p>
    </div>
    '''
```

### JWTトークンによる認証

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>JWT認証 with HTMX</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
</head>
<body>
    <h1>セキュアなアプリケーション</h1>
    
    <!-- ログインフォーム -->
    <form id="login-form" 
          hx-post="/api/auth/login"
          hx-target="#content"
          hx-on="htmx:afterRequest: handleLoginResponse(event)">
        <input type="text" name="username" placeholder="ユーザー名" required>
        <input type="password" name="password" placeholder="パスワード" required>
        <button type="submit">ログイン</button>
    </form>
    
    <!-- 保護されたコンテンツ -->
    <div id="content"></div>
    
    <script>
    // トークン管理
    class TokenManager {
        constructor() {
            this.accessToken = null;
            this.refreshToken = null;
            this.tokenExpiry = null;
        }
        
        // トークンの保存
        saveTokens(accessToken, refreshToken, expiresIn) {
            this.accessToken = accessToken;
            this.refreshToken = refreshToken;
            this.tokenExpiry = new Date(Date.now() + expiresIn * 1000);
            
            // リフレッシュトークンのみlocalStorageに保存
            localStorage.setItem('refreshToken', refreshToken);
            
            // 自動更新のスケジュール
            this.scheduleTokenRefresh();
        }
        
        // トークンの取得
        getAccessToken() {
            // 期限切れチェック
            if (!this.accessToken || new Date() >= this.tokenExpiry) {
                return null;
            }
            return this.accessToken;
        }
        
        // トークンのリフレッシュ
        async refreshAccessToken() {
            const refreshToken = this.refreshToken || localStorage.getItem('refreshToken');
            if (!refreshToken) {
                throw new Error('No refresh token available');
            }
            
            try {
                const response = await fetch('/api/auth/refresh', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ refreshToken })
                });
                
                if (!response.ok) {
                    throw new Error('Token refresh failed');
                }
                
                const data = await response.json();
                this.saveTokens(data.accessToken, data.refreshToken, data.expiresIn);
                
                return data.accessToken;
            } catch (error) {
                // リフレッシュ失敗時はログアウト
                this.clearTokens();
                window.location.href = '/login';
                throw error;
            }
        }
        
        // トークンの自動更新
        scheduleTokenRefresh() {
            if (!this.tokenExpiry) return;
            
            // 期限の5分前に更新
            const refreshTime = this.tokenExpiry.getTime() - Date.now() - 5 * 60 * 1000;
            
            if (refreshTime > 0) {
                setTimeout(() => {
                    this.refreshAccessToken();
                }, refreshTime);
            }
        }
        
        // トークンのクリア
        clearTokens() {
            this.accessToken = null;
            this.refreshToken = null;
            this.tokenExpiry = null;
            localStorage.removeItem('refreshToken');
        }
    }
    
    const tokenManager = new TokenManager();
    
    // HTMXリクエストにトークンを追加
    document.body.addEventListener('htmx:configRequest', async (event) => {
        let token = tokenManager.getAccessToken();
        
        // トークンがない場合はリフレッシュを試みる
        if (!token) {
            try {
                token = await tokenManager.refreshAccessToken();
            } catch (error) {
                console.error('Token refresh failed:', error);
                return;
            }
        }
        
        // Authorizationヘッダーを追加
        event.detail.headers['Authorization'] = `Bearer ${token}`;
    });
    
    // ログインレスポンスの処理
    function handleLoginResponse(event) {
        if (!event.detail.successful) return;
        
        try {
            const response = JSON.parse(event.detail.xhr.response);
            tokenManager.saveTokens(
                response.accessToken,
                response.refreshToken,
                response.expiresIn
            );
            
            // ダッシュボードへリダイレクト
            htmx.ajax('GET', '/api/dashboard', '#content');
        } catch (error) {
            console.error('Login response handling error:', error);
        }
    }
    
    // 401エラーの処理
    document.body.addEventListener('htmx:responseError', (event) => {
        if (event.detail.xhr.status === 401) {
            // 認証エラー時はログインページへ
            tokenManager.clearTokens();
            window.location.href = '/login';
        }
    });
    </script>
</body>
</html>
```

## 9.5 キャッシュ戦略

### HTTP キャッシュヘッダーの活用

```python
from flask import Flask, make_response, request
from datetime import datetime, timedelta
import hashlib

app = Flask(__name__)

# ETagの生成
def generate_etag(content):
    return hashlib.md5(content.encode()).hexdigest()

@app.route('/api/static-content')
def static_content():
    content = render_static_content()
    etag = generate_etag(content)
    
    # クライアントのETagと比較
    if request.headers.get('If-None-Match') == etag:
        return '', 304  # Not Modified
    
    response = make_response(content)
    
    # キャッシュヘッダーの設定
    response.headers['ETag'] = etag
    response.headers['Cache-Control'] = 'public, max-age=3600'  # 1時間
    response.headers['Last-Modified'] = datetime.now().strftime('%a, %d %b %Y %H:%M:%S GMT')
    
    return response

@app.route('/api/dynamic-content')
def dynamic_content():
    content = render_dynamic_content()
    
    response = make_response(content)
    
    # 動的コンテンツはキャッシュしない
    response.headers['Cache-Control'] = 'no-cache, no-store, must-revalidate'
    response.headers['Pragma'] = 'no-cache'
    response.headers['Expires'] = '0'
    
    return response

@app.route('/api/user-specific/<user_id>')
def user_specific_content(user_id):
    content = render_user_content(user_id)
    
    response = make_response(content)
    
    # プライベートキャッシュ（ブラウザのみ）
    response.headers['Cache-Control'] = 'private, max-age=600'  # 10分
    response.headers['Vary'] = 'Authorization'  # 認証ヘッダーごとに異なるキャッシュ
    
    return response
```

### Service Workerによる高度なキャッシュ

```javascript
// sw.js - Service Worker でのキャッシュ戦略
const CACHE_VERSION = 'v1';
const CACHE_NAMES = {
    static: `static-${CACHE_VERSION}`,
    dynamic: `dynamic-${CACHE_VERSION}`,
    api: `api-${CACHE_VERSION}`
};

// キャッシュ戦略の定義
const cacheStrategies = {
    // ネットワークファースト（API用）
    networkFirst: async (request, cacheName) => {
        try {
            const networkResponse = await fetch(request);
            
            if (networkResponse.ok) {
                const cache = await caches.open(cacheName);
                cache.put(request, networkResponse.clone());
            }
            
            return networkResponse;
        } catch (error) {
            const cachedResponse = await caches.match(request);
            if (cachedResponse) {
                return cachedResponse;
            }
            throw error;
        }
    },
    
    // キャッシュファースト（静的リソース用）
    cacheFirst: async (request, cacheName) => {
        const cachedResponse = await caches.match(request);
        if (cachedResponse) {
            return cachedResponse;
        }
        
        const networkResponse = await fetch(request);
        
        if (networkResponse.ok) {
            const cache = await caches.open(cacheName);
            cache.put(request, networkResponse.clone());
        }
        
        return networkResponse;
    },
    
    // ステイル・ワイル・リバリデート
    staleWhileRevalidate: async (request, cacheName) => {
        const cache = await caches.open(cacheName);
        const cachedResponse = await cache.match(request);
        
        const fetchPromise = fetch(request).then(networkResponse => {
            if (networkResponse.ok) {
                cache.put(request, networkResponse.clone());
            }
            return networkResponse;
        });
        
        return cachedResponse || fetchPromise;
    }
};

// リクエストのルーティング
self.addEventListener('fetch', (event) => {
    const { request } = event;
    const url = new URL(request.url);
    
    // APIリクエスト
    if (url.pathname.startsWith('/api/')) {
        // 重要なAPIはネットワークファースト
        if (url.pathname.includes('/critical/')) {
            event.respondWith(
                cacheStrategies.networkFirst(request, CACHE_NAMES.api)
            );
        } else {
            // その他のAPIはステイル・ワイル・リバリデート
            event.respondWith(
                cacheStrategies.staleWhileRevalidate(request, CACHE_NAMES.api)
            );
        }
        return;
    }
    
    // 静的リソース
    if (request.destination === 'image' || 
        request.destination === 'style' || 
        request.destination === 'script') {
        event.respondWith(
            cacheStrategies.cacheFirst(request, CACHE_NAMES.static)
        );
        return;
    }
    
    // その他のリクエスト
    event.respondWith(
        cacheStrategies.networkFirst(request, CACHE_NAMES.dynamic)
    );
});

// キャッシュのクリーンアップ
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then(cacheNames => {
            return Promise.all(
                cacheNames
                    .filter(cacheName => !Object.values(CACHE_NAMES).includes(cacheName))
                    .map(cacheName => caches.delete(cacheName))
            );
        })
    );
});

// バックグラウンド同期
self.addEventListener('sync', (event) => {
    if (event.tag === 'sync-data') {
        event.waitUntil(syncOfflineData());
    }
});

async function syncOfflineData() {
    // IndexedDBから保留中のデータを取得
    const pendingData = await getPendingData();
    
    for (const data of pendingData) {
        try {
            await fetch(data.url, {
                method: data.method,
                headers: data.headers,
                body: data.body
            });
            
            // 成功したらIndexedDBから削除
            await removePendingData(data.id);
        } catch (error) {
            console.error('Sync failed:', error);
        }
    }
}
```

## 9.6 実践的な永続化パターン

### ハイブリッドアプローチ

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>ハイブリッド永続化</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
</head>
<body>
    <h1>スマート永続化システム</h1>
    
    <div id="app-container">
        <!-- アプリケーションコンテンツ -->
    </div>
    
    <script>
    // 永続化マネージャー
    class PersistenceManager {
        constructor() {
            this.strategies = {
                settings: 'localStorage',      // 設定は LocalStorage
                session: 'sessionStorage',     // セッションデータ
                largeData: 'indexedDB',       // 大規模データ
                temporary: 'memory'           // 一時データはメモリ
            };
            
            this.memoryCache = new Map();
            this.db = null;
        }
        
        // データの保存（自動的に適切な場所を選択）
        async save(key, value, options = {}) {
            const strategy = options.strategy || this.determineStrategy(key, value);
            
            switch (strategy) {
                case 'localStorage':
                    return this.saveToLocalStorage(key, value);
                
                case 'sessionStorage':
                    return this.saveToSessionStorage(key, value);
                
                case 'indexedDB':
                    return this.saveToIndexedDB(key, value);
                
                case 'memory':
                    return this.saveToMemory(key, value);
                
                default:
                    throw new Error(`Unknown strategy: ${strategy}`);
            }
        }
        
        // データの取得
        async get(key, options = {}) {
            const strategy = options.strategy || this.determineStrategy(key);
            
            switch (strategy) {
                case 'localStorage':
                    return this.getFromLocalStorage(key);
                
                case 'sessionStorage':
                    return this.getFromSessionStorage(key);
                
                case 'indexedDB':
                    return this.getFromIndexedDB(key);
                
                case 'memory':
                    return this.getFromMemory(key);
            }
        }
        
        // 戦略の自動決定
        determineStrategy(key, value = null) {
            // キーのプレフィックスで判断
            if (key.startsWith('setting_')) return 'localStorage';
            if (key.startsWith('session_')) return 'sessionStorage';
            if (key.startsWith('temp_')) return 'memory';
            
            // データサイズで判断
            if (value) {
                const size = JSON.stringify(value).length;
                if (size > 1024 * 100) { // 100KB以上
                    return 'indexedDB';
                }
            }
            
            // デフォルトは localStorage
            return 'localStorage';
        }
        
        // LocalStorage 操作
        saveToLocalStorage(key, value) {
            try {
                localStorage.setItem(key, JSON.stringify({
                    data: value,
                    timestamp: new Date().toISOString()
                }));
                return true;
            } catch (error) {
                // 容量制限エラーの場合は古いデータを削除
                if (error.name === 'QuotaExceededError') {
                    this.cleanupLocalStorage();
                    // リトライ
                    return this.saveToLocalStorage(key, value);
                }
                throw error;
            }
        }
        
        getFromLocalStorage(key) {
            const item = localStorage.getItem(key);
            if (!item) return null;
            
            try {
                const { data, timestamp } = JSON.parse(item);
                
                // 有効期限チェック（オプション）
                if (this.isExpired(timestamp, key)) {
                    localStorage.removeItem(key);
                    return null;
                }
                
                return data;
            } catch (error) {
                return null;
            }
        }
        
        // LocalStorage のクリーンアップ
        cleanupLocalStorage() {
            const items = [];
            
            for (let i = 0; i < localStorage.length; i++) {
                const key = localStorage.key(i);
                const item = localStorage.getItem(key);
                
                try {
                    const { timestamp } = JSON.parse(item);
                    items.push({ key, timestamp });
                } catch (error) {
                    // パースできないアイテムは削除対象
                    items.push({ key, timestamp: '1970-01-01' });
                }
            }
            
            // 古い順にソート
            items.sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));
            
            // 古いアイテムから削除（全体の20%）
            const deleteCount = Math.floor(items.length * 0.2);
            for (let i = 0; i < deleteCount; i++) {
                localStorage.removeItem(items[i].key);
            }
        }
        
        // 有効期限チェック
        isExpired(timestamp, key) {
            const expiry = {
                'cache_': 3600000,      // 1時間
                'temp_': 600000,        // 10分
                'default': 86400000     // 24時間
            };
            
            const prefix = Object.keys(expiry).find(p => key.startsWith(p));
            const maxAge = expiry[prefix] || expiry.default;
            
            return new Date() - new Date(timestamp) > maxAge;
        }
    }
    
    // グローバルインスタンス
    const persistence = new PersistenceManager();
    
    // HTMXとの統合
    document.body.addEventListener('htmx:afterRequest', async (event) => {
        // レスポンスをキャッシュ
        const url = event.detail.pathInfo.requestPath;
        const response = event.detail.xhr.response;
        
        if (event.detail.successful && response) {
            await persistence.save(`cache_${url}`, {
                content: response,
                headers: event.detail.xhr.getAllResponseHeaders()
            });
        }
    });
    
    // オフライン時のフォールバック
    document.body.addEventListener('htmx:sendError', async (event) => {
        const url = event.detail.pathInfo.requestPath;
        
        // キャッシュから取得を試みる
        const cached = await persistence.get(`cache_${url}`);
        
        if (cached) {
            // キャッシュされたコンテンツで更新
            const target = event.detail.target;
            target.innerHTML = cached.content;
            
            // オフライン表示
            showNotification('オフライン: キャッシュデータを表示しています');
        }
    });
    </script>
</body>
</html>
```

## まとめ

この章では、HTMXアプリケーションにおける様々なデータ永続化手法について学びました：

1. **LocalStorage/SessionStorage**: 小規模データの簡単な永続化
2. **IndexedDB**: 大規模データとオフライン対応
3. **Cookie/セッション**: サーバー側での状態管理
4. **JWT認証**: セキュアなトークンベース認証
5. **キャッシュ戦略**: Service Workerを使った高度なキャッシュ
6. **ハイブリッドアプローチ**: 複数の手法を組み合わせた最適化

これらの技術を適切に組み合わせることで、高速で信頼性の高いHTMXアプリケーションを構築できます。

## 練習問題

### 問題1: オフライン対応フォーム
以下の要件を満たすフォームを実装してください：
- オフライン時は LocalStorage に保存
- オンライン復帰時に自動送信
- 送信状態の可視化
- 重複送信の防止

### 問題2: スマートキャッシュ
以下の機能を持つキャッシュシステムを実装してください：
- URLパターンに基づく異なるキャッシュ戦略
- キャッシュサイズの管理
- 有効期限の自動管理
- キャッシュヒット率の計測

### 問題3: セッション管理
以下の機能を実装してください：
- 自動ログアウト（アイドル時間）
- セッション延長機能
- 複数タブでのセッション同期
- セキュアなトークン管理

### 問題4: データ同期
オンライン/オフライン間でのデータ同期システムを作成してください：
- 競合解決メカニズム
- 同期状態の表示
- 部分的な同期のサポート
- エラーリカバリー

### 問題5: パフォーマンス最適化
永続化レイヤーのパフォーマンスを最適化してください：
- 遅延読み込み
- バッチ処理
- 圧縮
- インデックスの最適化

---

**解答例とヒント：**

前章（第6章）の練習問題の解答例：

問題1の解答（基本的なフォームバリデーション）：
```html
<form hx-post="/api/register" hx-target="#result">
    <input type="email" 
           name="email"
           hx-post="/api/validate/email"
           hx-trigger="blur"
           hx-target="next .error"
           required>
    <span class="error"></span>
    
    <input type="password" 
           name="password"
           hx-post="/api/validate/password"
           hx-trigger="input throttle:500ms"
           hx-target="next .strength">
    <div class="strength"></div>
    
    <label>
        <input type="checkbox" name="terms" required>
        利用規約に同意する
    </label>
    
    <button type="submit" disabled>登録</button>
</form>

<script>
// すべての条件を満たした時のみボタンを有効化
document.querySelectorAll('input').forEach(input => {
    input.addEventListener('change', checkFormValidity);
});

function checkFormValidity() {
    const form = document.querySelector('form');
    const button = form.querySelector('button[type="submit"]');
    button.disabled = !form.checkValidity();
}
</script>
```