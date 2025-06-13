# 第7章: 応用編 - インジケーターとエラーハンドリング

## 章の概要

この章では、HTMXアプリケーションのユーザビリティを大幅に向上させるローディングインジケーター、プログレス表示、エラーハンドリング、リトライ機能などの高度なUIパターンについて学びます。

## 7.1 ローディングインジケーターの基本

### htmx-indicatorクラスの使用

```html
<!-- 基本的なインジケーター -->
<button hx-get="/api/slow-data" 
        hx-target="#result"
        hx-indicator="#loading">
    データ取得
</button>
<span id="loading" class="htmx-indicator">
    読み込み中...
</span>

<!-- CSSでインジケーターをスタイリング -->
<style>
.htmx-indicator {
    display: none;
}
.htmx-request .htmx-indicator {
    display: inline;
}
.htmx-request.htmx-indicator {
    display: inline;
}

/* スピナーアニメーション */
.spinner {
    display: inline-block;
    width: 20px;
    height: 20px;
    border: 3px solid #f3f3f3;
    border-top: 3px solid #3498db;
    border-radius: 50%;
    animation: spin 1s linear infinite;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}
</style>

<button hx-get="/api/data" 
        hx-indicator="#spinner">
    取得
</button>
<div id="spinner" class="htmx-indicator">
    <div class="spinner"></div>
</div>
```

### 要素ごとのインジケーター

```html
<!-- ボタン内のインジケーター -->
<button hx-post="/api/save" 
        hx-indicator="find .spinner"
        class="btn-save">
    <span class="btn-text">保存</span>
    <span class="spinner htmx-indicator">
        <i class="fas fa-spinner fa-spin"></i>
    </span>
</button>

<!-- カード全体のローディング状態 -->
<div class="card" 
     hx-get="/api/card-data"
     hx-trigger="revealed"
     hx-indicator="this">
    <div class="card-content">
        <!-- コンテンツ -->
    </div>
    <div class="card-overlay htmx-indicator">
        <div class="loading-message">
            更新中...
        </div>
    </div>
</div>

<style>
.card {
    position: relative;
}
.card-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(255, 255, 255, 0.8);
    display: none;
    align-items: center;
    justify-content: center;
}
.htmx-request .card-overlay {
    display: flex;
}
</style>
```

## 7.2 プログレスバーの実装

### 基本的なプログレスバー

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>プログレスバー</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .progress-container {
            width: 100%;
            height: 20px;
            background: #f0f0f0;
            border-radius: 10px;
            overflow: hidden;
            margin: 20px 0;
        }
        .progress-bar {
            height: 100%;
            background: #4CAF50;
            width: 0%;
            transition: width 0.3s ease;
            text-align: center;
            line-height: 20px;
            color: white;
        }
        .upload-zone {
            border: 2px dashed #ccc;
            padding: 40px;
            text-align: center;
            margin: 20px 0;
        }
    </style>
</head>
<body>
    <h1>ファイルアップロード with プログレスバー</h1>
    
    <form hx-post="/api/upload" 
          hx-encoding="multipart/form-data"
          hx-indicator="#upload-indicator">
        <div class="upload-zone">
            <input type="file" name="file" id="file-input" accept=".pdf,.doc,.docx">
            <p>またはここにファイルをドロップ</p>
        </div>
        
        <div id="upload-indicator" class="htmx-indicator">
            <div class="progress-container">
                <div class="progress-bar" id="progress-bar">0%</div>
            </div>
            <p id="upload-status">アップロード中...</p>
        </div>
        
        <button type="submit">アップロード</button>
    </form>
    
    <div id="result"></div>
    
    <script>
    // プログレスイベントの監視
    htmx.on('htmx:xhr:progress', function(evt) {
        if (evt.detail.lengthComputable) {
            const percentComplete = Math.round((evt.detail.loaded / evt.detail.total) * 100);
            const progressBar = document.getElementById('progress-bar');
            progressBar.style.width = percentComplete + '%';
            progressBar.textContent = percentComplete + '%';
            
            // ステータステキストの更新
            const status = document.getElementById('upload-status');
            if (percentComplete < 100) {
                status.textContent = `アップロード中... ${percentComplete}%`;
            } else {
                status.textContent = '処理中...';
            }
        }
    });
    
    // ドラッグ&ドロップ対応
    const uploadZone = document.querySelector('.upload-zone');
    
    uploadZone.addEventListener('dragover', function(e) {
        e.preventDefault();
        this.style.background = '#e1e1e1';
    });
    
    uploadZone.addEventListener('dragleave', function(e) {
        this.style.background = '';
    });
    
    uploadZone.addEventListener('drop', function(e) {
        e.preventDefault();
        this.style.background = '';
        
        const files = e.dataTransfer.files;
        if (files.length > 0) {
            document.getElementById('file-input').files = files;
            // フォームを自動送信
            this.closest('form').requestSubmit();
        }
    });
    </script>
</body>
</html>
```

### 複数ステップのプログレス

```html
<div class="multi-step-progress">
    <div class="step-indicator">
        <div class="step completed">
            <span class="step-number">1</span>
            <span class="step-label">準備</span>
        </div>
        <div class="step active">
            <span class="step-number">2</span>
            <span class="step-label">処理中</span>
        </div>
        <div class="step">
            <span class="step-number">3</span>
            <span class="step-label">完了</span>
        </div>
    </div>
    
    <div hx-get="/api/process" 
         hx-trigger="load"
         hx-target="#process-result"
         hx-vals='{"step": 2}'>
        <div class="processing-message">
            データを処理しています...
        </div>
    </div>
</div>

<style>
.step-indicator {
    display: flex;
    justify-content: space-between;
    margin-bottom: 30px;
}
.step {
    flex: 1;
    text-align: center;
    position: relative;
}
.step:not(:last-child)::after {
    content: '';
    position: absolute;
    top: 15px;
    right: -50%;
    width: 100%;
    height: 2px;
    background: #ddd;
}
.step.completed::after {
    background: #4CAF50;
}
.step-number {
    display: inline-block;
    width: 30px;
    height: 30px;
    border-radius: 50%;
    background: #ddd;
    line-height: 30px;
}
.step.active .step-number,
.step.completed .step-number {
    background: #4CAF50;
    color: white;
}
</style>
```

## 7.3 高度なエラーハンドリング

### エラー状態の表示

```html
<!-- エラー専用のターゲット -->
<div hx-get="/api/risky-operation"
     hx-target="#result"
     hx-error-target="#error-display">
    <button>実行</button>
</div>

<div id="result"></div>
<div id="error-display"></div>

<!-- グローバルエラーハンドラー -->
<script>
document.body.addEventListener('htmx:responseError', function(evt) {
    const xhr = evt.detail.xhr;
    const errorDisplay = document.getElementById('error-display');
    
    let errorMessage = 'エラーが発生しました';
    let errorDetails = '';
    
    // ステータスコードによる分岐
    switch(xhr.status) {
        case 400:
            errorMessage = '入力内容に誤りがあります';
            try {
                const response = JSON.parse(xhr.responseText);
                errorDetails = response.message || '';
            } catch (e) {
                errorDetails = xhr.responseText;
            }
            break;
        case 401:
            errorMessage = '認証が必要です';
            errorDetails = 'ログインしてください';
            // 3秒後にログインページへリダイレクト
            setTimeout(() => {
                window.location.href = '/login';
            }, 3000);
            break;
        case 403:
            errorMessage = 'アクセス権限がありません';
            break;
        case 404:
            errorMessage = 'リソースが見つかりません';
            break;
        case 429:
            errorMessage = 'リクエストが多すぎます';
            errorDetails = 'しばらく待ってから再試行してください';
            break;
        case 500:
            errorMessage = 'サーバーエラー';
            errorDetails = 'システム管理者に連絡してください';
            break;
        case 503:
            errorMessage = 'サービス利用不可';
            errorDetails = 'メンテナンス中の可能性があります';
            break;
        default:
            if (xhr.status === 0) {
                errorMessage = 'ネットワークエラー';
                errorDetails = 'インターネット接続を確認してください';
            }
    }
    
    // エラー表示
    errorDisplay.innerHTML = `
        <div class="error-box">
            <div class="error-icon">⚠️</div>
            <div class="error-content">
                <h3>${errorMessage}</h3>
                ${errorDetails ? `<p>${errorDetails}</p>` : ''}
                <button onclick="retryLastRequest()" class="retry-button">
                    再試行
                </button>
            </div>
        </div>
    `;
});

// 最後のリクエストを保存
let lastRequest = null;

document.body.addEventListener('htmx:beforeRequest', function(evt) {
    lastRequest = {
        element: evt.detail.elt,
        verb: evt.detail.verb,
        path: evt.detail.path
    };
});

// リトライ機能
function retryLastRequest() {
    if (lastRequest) {
        htmx.ajax(
            lastRequest.verb,
            lastRequest.path,
            lastRequest.element
        );
    }
}
</script>

<style>
.error-box {
    background: #ffebee;
    border: 1px solid #ffcdd2;
    border-radius: 4px;
    padding: 20px;
    margin: 20px 0;
    display: flex;
    align-items: center;
}
.error-icon {
    font-size: 2em;
    margin-right: 15px;
}
.error-content h3 {
    margin: 0 0 10px 0;
    color: #c62828;
}
.retry-button {
    background: #f44336;
    color: white;
    border: none;
    padding: 8px 16px;
    border-radius: 4px;
    cursor: pointer;
    margin-top: 10px;
}
.retry-button:hover {
    background: #d32f2f;
}
</style>
```

### 自動リトライ機能

```html
<script>
// リトライ設定
const retryConfig = {
    maxRetries: 3,
    retryDelay: 1000, // ミリ秒
    backoffMultiplier: 2
};

// リトライカウンターを要素に保存
const retryCounters = new WeakMap();

document.body.addEventListener('htmx:responseError', function(evt) {
    const xhr = evt.detail.xhr;
    const element = evt.detail.elt;
    
    // ネットワークエラーまたは5xxエラーの場合のみリトライ
    if (xhr.status === 0 || (xhr.status >= 500 && xhr.status < 600)) {
        handleRetry(element, evt.detail);
    }
});

function handleRetry(element, requestDetails) {
    // 現在のリトライ回数を取得
    let retryCount = retryCounters.get(element) || 0;
    
    if (retryCount < retryConfig.maxRetries) {
        retryCount++;
        retryCounters.set(element, retryCount);
        
        // 指数バックオフ
        const delay = retryConfig.retryDelay * Math.pow(retryConfig.backoffMultiplier, retryCount - 1);
        
        // リトライ表示
        showRetryNotification(element, retryCount, delay);
        
        // 遅延後にリトライ
        setTimeout(() => {
            htmx.trigger(element, 'retry');
        }, delay);
    } else {
        // リトライ上限に達した
        showFinalError(element);
        retryCounters.delete(element);
    }
}

function showRetryNotification(element, attempt, delay) {
    const notification = document.createElement('div');
    notification.className = 'retry-notification';
    notification.innerHTML = `
        <span>接続エラー: ${attempt}回目の再試行を${delay/1000}秒後に実行します...</span>
        <button onclick="cancelRetry(this)">キャンセル</button>
    `;
    element.parentNode.insertBefore(notification, element.nextSibling);
}
</script>
```

## 7.4 状態管理とフィードバック

### 複雑な状態遷移

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>状態管理デモ</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .status-container {
            padding: 20px;
            border-radius: 8px;
            margin: 20px 0;
            transition: all 0.3s ease;
        }
        .status-idle {
            background: #e3f2fd;
            border: 1px solid #90caf9;
        }
        .status-loading {
            background: #fff3e0;
            border: 1px solid #ffcc80;
        }
        .status-success {
            background: #e8f5e9;
            border: 1px solid #81c784;
        }
        .status-error {
            background: #ffebee;
            border: 1px solid #ef5350;
        }
        .status-icon {
            font-size: 2em;
            margin-right: 10px;
        }
    </style>
</head>
<body>
    <h1>非同期処理の状態管理</h1>
    
    <div id="operation-container" class="status-container status-idle">
        <div class="status-content">
            <span class="status-icon">⏸️</span>
            <span class="status-text">待機中</span>
        </div>
        
        <button hx-post="/api/long-operation"
                hx-target="#operation-result"
                hx-indicator="#operation-indicator"
                hx-on="htmx:beforeRequest: updateStatus('loading')"
                hx-on="htmx:afterRequest: updateStatus(event.detail.successful ? 'success' : 'error')">
            処理開始
        </button>
        
        <div id="operation-indicator" class="htmx-indicator">
            <div class="processing-steps">
                <div class="step" data-step="1">
                    <span class="step-icon">⏳</span>
                    <span>データ取得中...</span>
                </div>
                <div class="step" data-step="2">
                    <span class="step-icon">🔄</span>
                    <span>処理中...</span>
                </div>
                <div class="step" data-step="3">
                    <span class="step-icon">💾</span>
                    <span>保存中...</span>
                </div>
            </div>
        </div>
        
        <div id="operation-result"></div>
    </div>
    
    <script>
    function updateStatus(status) {
        const container = document.getElementById('operation-container');
        const statusText = container.querySelector('.status-text');
        const statusIcon = container.querySelector('.status-icon');
        
        // すべてのステータスクラスを削除
        container.className = 'status-container';
        
        switch(status) {
            case 'loading':
                container.classList.add('status-loading');
                statusIcon.textContent = '⏳';
                statusText.textContent = '処理中...';
                break;
            case 'success':
                container.classList.add('status-success');
                statusIcon.textContent = '✅';
                statusText.textContent = '完了しました';
                break;
            case 'error':
                container.classList.add('status-error');
                statusIcon.textContent = '❌';
                statusText.textContent = 'エラーが発生しました';
                break;
            default:
                container.classList.add('status-idle');
                statusIcon.textContent = '⏸️';
                statusText.textContent = '待機中';
        }
    }
    
    // SSEまたはWebSocketで進捗更新を受信
    htmx.on('htmx:sseMessage', function(evt) {
        if (evt.detail.type === 'progress') {
            const data = JSON.parse(evt.detail.data);
            updateProgress(data.step, data.message);
        }
    });
    </script>
</body>
</html>
```

### トースト通知システム

```html
<div id="toast-container"></div>

<script>
class ToastManager {
    constructor() {
        this.container = document.getElementById('toast-container');
        this.toasts = [];
    }
    
    show(message, type = 'info', duration = 3000) {
        const toast = document.createElement('div');
        toast.className = `toast toast-${type}`;
        toast.innerHTML = `
            <div class="toast-icon">${this.getIcon(type)}</div>
            <div class="toast-message">${message}</div>
            <button class="toast-close" onclick="toastManager.remove('${toast.id}')">×</button>
        `;
        toast.id = `toast-${Date.now()}`;
        
        this.container.appendChild(toast);
        this.toasts.push(toast);
        
        // アニメーション
        requestAnimationFrame(() => {
            toast.classList.add('show');
        });
        
        // 自動削除
        if (duration > 0) {
            setTimeout(() => this.remove(toast.id), duration);
        }
        
        return toast.id;
    }
    
    getIcon(type) {
        const icons = {
            info: 'ℹ️',
            success: '✅',
            warning: '⚠️',
            error: '❌'
        };
        return icons[type] || icons.info;
    }
    
    remove(toastId) {
        const toast = document.getElementById(toastId);
        if (toast) {
            toast.classList.remove('show');
            setTimeout(() => toast.remove(), 300);
        }
    }
}

const toastManager = new ToastManager();

// HTMXイベントと連携
document.body.addEventListener('htmx:afterRequest', function(evt) {
    if (evt.detail.successful) {
        const response = evt.detail.xhr.response;
        if (response.includes('success')) {
            toastManager.show('操作が完了しました', 'success');
        }
    } else {
        toastManager.show('エラーが発生しました', 'error');
    }
});
</script>

<style>
#toast-container {
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 1000;
}

.toast {
    display: flex;
    align-items: center;
    min-width: 250px;
    margin-bottom: 10px;
    padding: 16px;
    border-radius: 4px;
    box-shadow: 0 2px 5px rgba(0,0,0,0.2);
    opacity: 0;
    transform: translateX(100%);
    transition: all 0.3s ease;
}

.toast.show {
    opacity: 1;
    transform: translateX(0);
}

.toast-info { background: #2196F3; color: white; }
.toast-success { background: #4CAF50; color: white; }
.toast-warning { background: #FF9800; color: white; }
.toast-error { background: #F44336; color: white; }

.toast-icon {
    margin-right: 12px;
    font-size: 1.2em;
}

.toast-message {
    flex: 1;
}

.toast-close {
    background: none;
    border: none;
    color: white;
    font-size: 1.5em;
    cursor: pointer;
    opacity: 0.7;
}

.toast-close:hover {
    opacity: 1;
}
</style>
```

## 7.5 パフォーマンスモニタリング

### リクエストタイミングの計測

```html
<script>
// パフォーマンス計測
const performanceMonitor = {
    requests: new Map(),
    
    startTimer(requestId) {
        this.requests.set(requestId, {
            start: performance.now(),
            url: ''
        });
    },
    
    endTimer(requestId) {
        const request = this.requests.get(requestId);
        if (request) {
            const duration = performance.now() - request.start;
            this.requests.delete(requestId);
            return duration;
        }
        return 0;
    },
    
    getStats() {
        const durations = Array.from(this.requests.values())
            .map(r => performance.now() - r.start);
        
        if (durations.length === 0) return null;
        
        return {
            average: durations.reduce((a, b) => a + b, 0) / durations.length,
            min: Math.min(...durations),
            max: Math.max(...durations),
            count: durations.length
        };
    }
};

// HTMXイベントにフック
document.body.addEventListener('htmx:beforeRequest', function(evt) {
    const requestId = evt.detail.requestConfig.requestId;
    performanceMonitor.startTimer(requestId);
});

document.body.addEventListener('htmx:afterRequest', function(evt) {
    const requestId = evt.detail.requestConfig.requestId;
    const duration = performanceMonitor.endTimer(requestId);
    
    // 遅いリクエストを警告
    if (duration > 3000) {
        console.warn(`Slow request detected: ${duration.toFixed(2)}ms`);
        toastManager.show(
            `リクエストに時間がかかっています (${(duration/1000).toFixed(1)}秒)`,
            'warning'
        );
    }
});
</script>

<!-- パフォーマンスダッシュボード -->
<div id="performance-dashboard" class="dashboard">
    <h3>パフォーマンスメトリクス</h3>
    <div hx-get="/api/performance/stats" 
         hx-trigger="every 5s"
         hx-target="this">
        <div class="metric">
            <span class="metric-label">平均応答時間:</span>
            <span class="metric-value">計測中...</span>
        </div>
    </div>
</div>
```

## 7.6 実践的な統合例

### 完全なエラーハンドリングシステム

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>統合エラーハンドリング</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
</head>
<body>
    <div class="container">
        <h1>データ管理システム</h1>
        
        <!-- オフライン警告 -->
        <div id="offline-warning" class="offline-warning" style="display: none;">
            <i class="fas fa-wifi-slash"></i>
            オフラインです - 一部の機能が制限されています
        </div>
        
        <!-- メイン操作エリア -->
        <div class="operations">
            <button hx-get="/api/data/fetch"
                    hx-target="#data-display"
                    hx-indicator="#global-loader"
                    class="btn btn-primary">
                データ取得
            </button>
            
            <button hx-post="/api/data/process"
                    hx-target="#data-display"
                    hx-indicator="#global-loader"
                    hx-confirm="処理を開始しますか？"
                    class="btn btn-warning">
                データ処理
            </button>
            
            <button hx-delete="/api/data/clear"
                    hx-target="#data-display"
                    hx-indicator="#global-loader"
                    hx-confirm="本当にデータを削除しますか？"
                    class="btn btn-danger">
                データ削除
            </button>
        </div>
        
        <!-- グローバルローダー -->
        <div id="global-loader" class="htmx-indicator global-loader">
            <div class="loader-content">
                <div class="spinner-border"></div>
                <p>処理中...</p>
                <button onclick="htmx.trigger(document.body, 'htmx:abort')">
                    キャンセル
                </button>
            </div>
        </div>
        
        <!-- データ表示エリア -->
        <div id="data-display" class="data-display"></div>
        
        <!-- エラーログ -->
        <div id="error-log" class="error-log">
            <h3>エラーログ</h3>
            <div id="error-entries"></div>
        </div>
    </div>
    
    <script>
    // オフライン検知
    window.addEventListener('online', function() {
        document.getElementById('offline-warning').style.display = 'none';
        toastManager.show('オンラインに復帰しました', 'success');
    });
    
    window.addEventListener('offline', function() {
        document.getElementById('offline-warning').style.display = 'block';
        toastManager.show('オフラインになりました', 'warning');
    });
    
    // エラーログ管理
    class ErrorLogger {
        constructor() {
            this.errors = [];
            this.maxErrors = 10;
        }
        
        log(error) {
            this.errors.unshift({
                timestamp: new Date(),
                message: error.message,
                status: error.status,
                url: error.url
            });
            
            if (this.errors.length > this.maxErrors) {
                this.errors.pop();
            }
            
            this.render();
        }
        
        render() {
            const container = document.getElementById('error-entries');
            container.innerHTML = this.errors.map(error => `
                <div class="error-entry">
                    <span class="error-time">${error.timestamp.toLocaleTimeString()}</span>
                    <span class="error-status">${error.status}</span>
                    <span class="error-message">${error.message}</span>
                </div>
            `).join('');
        }
    }
    
    const errorLogger = new ErrorLogger();
    
    // グローバルエラーハンドラー
    document.body.addEventListener('htmx:responseError', function(evt) {
        const xhr = evt.detail.xhr;
        errorLogger.log({
            message: `${evt.detail.verb} ${evt.detail.path} failed`,
            status: xhr.status,
            url: evt.detail.path
        });
    });
    
    // リクエストタイムアウト
    htmx.config.timeout = 30000; // 30秒
    
    document.body.addEventListener('htmx:timeout', function(evt) {
        toastManager.show('リクエストがタイムアウトしました', 'error');
        errorLogger.log({
            message: 'Request timeout',
            status: 0,
            url: evt.detail.path
        });
    });
    </script>
</body>
</html>
```

## まとめ

この章では、HTMXアプリケーションのUXを向上させる高度な技術について学びました：

1. **ローディングインジケーター**: htmx-indicatorクラスの活用
2. **プログレスバー**: ファイルアップロードや長時間処理の進捗表示
3. **エラーハンドリング**: 包括的なエラー処理とユーザーフィードバック
4. **自動リトライ**: ネットワークエラーへの対応
5. **状態管理**: 複雑なUIの状態遷移
6. **パフォーマンスモニタリング**: リクエストの計測と最適化

次章では、これまでの知識を統合して実践的なTODOアプリケーションを構築します。

## 練習問題

### 問題1: カスタムローディング表示
以下の要件を満たすローディング表示を作成してください：
- 3種類の異なるローディングアニメーション
- リクエストの種類によって使い分け
- キャンセルボタン付き

### 問題2: 高度なエラーハンドリング
エラーハンドリングシステムを実装してください：
- エラーの種類別に異なる対処法を提示
- エラーログの永続化（LocalStorage）
- エラーレポートの送信機能

### 問題3: プログレストラッキング
マルチステップ処理のプログレス表示を実装してください：
- 各ステップの所要時間予測
- 完了済みステップの結果表示
- 失敗時の部分的なロールバック

### 問題4: オフライン対応
オフライン時でも動作するフォームを作成してください：
- オフライン時はLocalStorageに保存
- オンライン復帰時に自動送信
- 同期状態の可視化

### 問題5: パフォーマンスダッシュボード
リアルタイムパフォーマンスダッシュボードを作成してください：
- 応答時間のグラフ表示
- エラー率の計算
- 遅いエンドポイントの特定

---

**解答例とヒント：**

前章（第5章）の練習問題の解答例：

問題1の解答：
```html
<!-- マウスオーバーで詳細表示 -->
<div class="card">
    <h3>商品名</h3>
    <div hx-get="/api/product/details" 
         hx-trigger="mouseenter once"
         hx-target="#details">
        詳細を見る
    </div>
    <div id="details"></div>
</div>

<!-- フォーカスアウトでバリデーション -->
<input type="email" 
       hx-post="/api/validate/email"
       hx-trigger="blur"
       hx-target="next .error">
<span class="error"></span>

<!-- ダブルクリックで編集モード -->
<div hx-get="/api/edit-mode" 
     hx-trigger="dblclick"
     hx-swap="outerHTML">
    ダブルクリックで編集
</div>
```