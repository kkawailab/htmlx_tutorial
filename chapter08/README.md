# 第8章: 実践プロジェクト - TODOアプリの作成

## 章の概要

この最終章では、これまでに学んだHTMXの知識をすべて活用して、完全に機能するTODOアプリケーションを構築します。リアルタイムの更新、ドラッグ&ドロップによる並び替え、フィルタリング、検索機能など、実践的な機能を実装します。

## 8.1 プロジェクトの設計

### アプリケーションの要件

1. **基本的なCRUD操作**
   - TODOの作成、読み取り、更新、削除
   - インライン編集機能

2. **高度な機能**
   - ドラッグ&ドロップで順序変更
   - カテゴリー分け
   - 優先度設定
   - 期限管理

3. **ユーザビリティ**
   - リアルタイム検索
   - フィルタリング（完了/未完了、カテゴリー別）
   - 一括操作
   - アンドゥ機能

4. **パフォーマンスとUX**
   - ローディング表示
   - エラーハンドリング
   - オフライン対応
   - レスポンシブデザイン

### プロジェクト構造

```
todo-app/
├── index.html          # メインページ
├── static/
│   ├── css/
│   │   └── style.css   # スタイルシート
│   ├── js/
│   │   └── app.js      # カスタムJavaScript
│   └── images/
├── server/
│   ├── app.py          # Flaskアプリケーション
│   ├── models.py       # データモデル
│   └── templates/      # HTMLテンプレート（部分）
│       ├── todo-item.html
│       ├── todo-form.html
│       └── filters.html
└── data/
    └── todos.json      # データストレージ（開発用）
```

## 8.2 フロントエンドの実装

### メインHTML構造

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HTMX TODO App</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <script src="https://unpkg.com/sortablejs@1.15.0/Sortable.min.js"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <link rel="stylesheet" href="/static/css/style.css">
</head>
<body>
    <div class="app-container">
        <!-- ヘッダー -->
        <header class="app-header">
            <h1><i class="fas fa-tasks"></i> HTMX TODO App</h1>
            <div class="header-stats" 
                 hx-get="/api/stats" 
                 hx-trigger="load, todoUpdated from:body"
                 hx-target="this">
                <!-- 統計情報 -->
            </div>
        </header>

        <!-- メインコンテンツ -->
        <main class="app-main">
            <!-- サイドバー -->
            <aside class="sidebar">
                <!-- フィルター -->
                <div class="filters" 
                     hx-get="/api/filters" 
                     hx-trigger="load"
                     hx-target="this">
                    読み込み中...
                </div>
                
                <!-- カテゴリー管理 -->
                <div class="categories">
                    <h3>カテゴリー</h3>
                    <div id="category-list" 
                         hx-get="/api/categories" 
                         hx-trigger="load, categoryUpdated from:body">
                        <!-- カテゴリー一覧 -->
                    </div>
                    <button class="btn-add-category" 
                            hx-get="/api/categories/new" 
                            hx-target="#category-form">
                        <i class="fas fa-plus"></i> 新規カテゴリー
                    </button>
                    <div id="category-form"></div>
                </div>
            </aside>

            <!-- TODOリスト -->
            <section class="todo-section">
                <!-- 新規TODO作成フォーム -->
                <div class="todo-create">
                    <form hx-post="/api/todos" 
                          hx-target="#todo-list" 
                          hx-swap="afterbegin"
                          hx-on="htmx:afterRequest: if(event.detail.successful) this.reset()">
                        <div class="input-group">
                            <input type="text" 
                                   name="title" 
                                   placeholder="新しいTODOを入力..." 
                                   required
                                   autocomplete="off">
                            <select name="priority">
                                <option value="low">低</option>
                                <option value="medium" selected>中</option>
                                <option value="high">高</option>
                            </select>
                            <input type="date" 
                                   name="due_date" 
                                   min="${new Date().toISOString().split('T')[0]}">
                            <button type="submit" class="btn btn-primary">
                                <i class="fas fa-plus"></i> 追加
                            </button>
                        </div>
                    </form>
                </div>

                <!-- 検索バー -->
                <div class="search-bar">
                    <i class="fas fa-search"></i>
                    <input type="search" 
                           placeholder="TODOを検索..." 
                           name="q"
                           hx-get="/api/todos/search" 
                           hx-trigger="keyup changed delay:300ms, search"
                           hx-target="#todo-list"
                           hx-indicator="#search-indicator">
                    <span id="search-indicator" class="htmx-indicator">
                        <i class="fas fa-spinner fa-spin"></i>
                    </span>
                </div>

                <!-- 一括操作 -->
                <div class="bulk-actions" style="display: none;">
                    <span class="selected-count">0個選択中</span>
                    <button class="btn btn-sm" onclick="markSelectedComplete()">
                        <i class="fas fa-check"></i> 完了にする
                    </button>
                    <button class="btn btn-sm btn-danger" onclick="deleteSelected()">
                        <i class="fas fa-trash"></i> 削除
                    </button>
                </div>

                <!-- TODOリスト本体 -->
                <div id="todo-list" 
                     class="todo-list"
                     hx-get="/api/todos" 
                     hx-trigger="load, todoUpdated from:body"
                     hx-vals='js:{filter: getCurrentFilter()}'>
                    <div class="loading">
                        <i class="fas fa-spinner fa-spin"></i> 読み込み中...
                    </div>
                </div>
            </section>
        </main>

        <!-- フッター -->
        <footer class="app-footer">
            <div class="footer-actions">
                <button hx-post="/api/todos/clear-completed" 
                        hx-confirm="完了したTODOをすべて削除しますか？"
                        hx-target="#todo-list">
                    完了項目をクリア
                </button>
                <button onclick="exportTodos()">
                    <i class="fas fa-download"></i> エクスポート
                </button>
                <button onclick="document.getElementById('import-input').click()">
                    <i class="fas fa-upload"></i> インポート
                </button>
                <input type="file" 
                       id="import-input" 
                       accept=".json" 
                       style="display: none;"
                       onchange="importTodos(this)">
            </div>
        </footer>
    </div>

    <!-- トースト通知コンテナ -->
    <div id="toast-container"></div>

    <!-- モーダル -->
    <div id="modal-container"></div>

    <script src="/static/js/app.js"></script>
</body>
</html>
```

### TODOアイテムのテンプレート

```html
<!-- todo-item.html -->
<div class="todo-item ${todo.completed ? 'completed' : ''} priority-${todo.priority}" 
     id="todo-${todo.id}"
     data-id="${todo.id}"
     draggable="true">
    
    <div class="todo-checkbox">
        <input type="checkbox" 
               id="check-${todo.id}"
               ${todo.completed ? 'checked' : ''}
               hx-patch="/api/todos/${todo.id}/toggle"
               hx-target="#todo-${todo.id}"
               hx-swap="outerHTML">
        <label for="check-${todo.id}"></label>
    </div>
    
    <div class="todo-content">
        <div class="todo-title" 
             hx-get="/api/todos/${todo.id}/edit"
             hx-trigger="dblclick"
             hx-target="#todo-${todo.id}"
             hx-swap="outerHTML">
            ${todo.title}
        </div>
        
        <div class="todo-meta">
            <span class="category">
                <i class="fas fa-tag"></i> ${todo.category || '未分類'}
            </span>
            <span class="due-date ${isPastDue(todo.due_date) ? 'overdue' : ''}">
                <i class="fas fa-calendar"></i> ${formatDate(todo.due_date)}
            </span>
        </div>
    </div>
    
    <div class="todo-actions">
        <button class="btn-icon" 
                hx-get="/api/todos/${todo.id}/detail"
                hx-target="#modal-container"
                title="詳細">
            <i class="fas fa-info-circle"></i>
        </button>
        <button class="btn-icon" 
                hx-delete="/api/todos/${todo.id}"
                hx-target="#todo-${todo.id}"
                hx-swap="outerHTML swap:0.3s"
                hx-confirm="このTODOを削除しますか？"
                title="削除">
            <i class="fas fa-trash"></i>
        </button>
    </div>
</div>
```

## 8.3 スタイルシート

```css
/* style.css */
:root {
    --primary-color: #4CAF50;
    --secondary-color: #2196F3;
    --danger-color: #f44336;
    --warning-color: #ff9800;
    --background: #f5f5f5;
    --surface: #ffffff;
    --text-primary: #212529;
    --text-secondary: #6c757d;
    --border-color: #dee2e6;
    --shadow: 0 2px 4px rgba(0,0,0,0.1);
}

* {
    box-sizing: border-box;
}

body {
    margin: 0;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: var(--background);
    color: var(--text-primary);
}

.app-container {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
}

/* ヘッダー */
.app-header {
    background: var(--surface);
    box-shadow: var(--shadow);
    padding: 1rem 2rem;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.app-header h1 {
    margin: 0;
    color: var(--primary-color);
}

.header-stats {
    display: flex;
    gap: 1rem;
    font-size: 0.9rem;
}

.stat-item {
    display: flex;
    align-items: center;
    gap: 0.5rem;
}

/* メインレイアウト */
.app-main {
    flex: 1;
    display: flex;
    padding: 2rem;
    gap: 2rem;
}

/* サイドバー */
.sidebar {
    width: 250px;
    background: var(--surface);
    border-radius: 8px;
    padding: 1.5rem;
    box-shadow: var(--shadow);
}

.filters h3,
.categories h3 {
    margin-top: 0;
    color: var(--text-secondary);
    font-size: 0.875rem;
    text-transform: uppercase;
    letter-spacing: 0.05em;
}

/* フィルター */
.filter-group {
    margin-bottom: 1rem;
}

.filter-item {
    display: flex;
    align-items: center;
    padding: 0.5rem;
    border-radius: 4px;
    cursor: pointer;
    transition: background 0.2s;
}

.filter-item:hover {
    background: var(--background);
}

.filter-item.active {
    background: var(--primary-color);
    color: white;
}

/* TODOセクション */
.todo-section {
    flex: 1;
}

.todo-create {
    background: var(--surface);
    border-radius: 8px;
    padding: 1.5rem;
    margin-bottom: 1rem;
    box-shadow: var(--shadow);
}

.input-group {
    display: flex;
    gap: 0.5rem;
}

.input-group input[type="text"] {
    flex: 1;
}

input, select, button {
    padding: 0.5rem 1rem;
    border: 1px solid var(--border-color);
    border-radius: 4px;
    font-size: 1rem;
}

/* ボタン */
.btn {
    background: var(--primary-color);
    color: white;
    border: none;
    cursor: pointer;
    transition: background 0.2s;
}

.btn:hover {
    background: #45a049;
}

.btn-danger {
    background: var(--danger-color);
}

.btn-danger:hover {
    background: #da190b;
}

/* TODOリスト */
.todo-list {
    background: var(--surface);
    border-radius: 8px;
    box-shadow: var(--shadow);
    min-height: 400px;
}

.todo-item {
    display: flex;
    align-items: center;
    padding: 1rem;
    border-bottom: 1px solid var(--border-color);
    transition: all 0.3s;
}

.todo-item:hover {
    background: var(--background);
}

.todo-item.completed {
    opacity: 0.6;
}

.todo-item.completed .todo-title {
    text-decoration: line-through;
}

.todo-item.dragging {
    opacity: 0.5;
    cursor: move;
}

/* 優先度による色分け */
.priority-high {
    border-left: 4px solid var(--danger-color);
}

.priority-medium {
    border-left: 4px solid var(--warning-color);
}

.priority-low {
    border-left: 4px solid var(--primary-color);
}

/* チェックボックス */
.todo-checkbox {
    margin-right: 1rem;
}

.todo-checkbox input[type="checkbox"] {
    width: 20px;
    height: 20px;
    cursor: pointer;
}

/* TODOコンテンツ */
.todo-content {
    flex: 1;
}

.todo-title {
    font-size: 1.1rem;
    margin-bottom: 0.25rem;
    cursor: pointer;
}

.todo-meta {
    display: flex;
    gap: 1rem;
    font-size: 0.875rem;
    color: var(--text-secondary);
}

.due-date.overdue {
    color: var(--danger-color);
    font-weight: bold;
}

/* アクションボタン */
.todo-actions {
    display: flex;
    gap: 0.5rem;
}

.btn-icon {
    background: none;
    border: none;
    color: var(--text-secondary);
    cursor: pointer;
    padding: 0.5rem;
    border-radius: 4px;
    transition: all 0.2s;
}

.btn-icon:hover {
    background: var(--background);
    color: var(--text-primary);
}

/* ローディング */
.loading {
    text-align: center;
    padding: 3rem;
    color: var(--text-secondary);
}

.htmx-indicator {
    display: none;
}

.htmx-request .htmx-indicator {
    display: inline-block;
}

/* トースト通知 */
#toast-container {
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 1000;
}

.toast {
    background: #333;
    color: white;
    padding: 1rem 1.5rem;
    border-radius: 4px;
    margin-bottom: 0.5rem;
    box-shadow: 0 4px 6px rgba(0,0,0,0.1);
    animation: slideIn 0.3s ease;
}

@keyframes slideIn {
    from {
        transform: translateX(100%);
        opacity: 0;
    }
    to {
        transform: translateX(0);
        opacity: 1;
    }
}

/* レスポンシブ */
@media (max-width: 768px) {
    .app-main {
        flex-direction: column;
    }
    
    .sidebar {
        width: 100%;
    }
    
    .input-group {
        flex-wrap: wrap;
    }
    
    .input-group input[type="text"] {
        width: 100%;
    }
}
```

## 8.4 JavaScript機能

```javascript
// app.js

// グローバル設定
htmx.config.defaultSwapStyle = 'innerHTML';

// 現在のフィルター状態
let currentFilter = {
    status: 'all',
    category: null,
    search: ''
};

// フィルター取得
function getCurrentFilter() {
    return currentFilter;
}

// ドラッグ&ドロップ
document.addEventListener('htmx:afterSwap', function(evt) {
    if (evt.detail.target.id === 'todo-list') {
        initializeSortable();
    }
});

function initializeSortable() {
    const todoList = document.getElementById('todo-list');
    if (!todoList) return;
    
    new Sortable(todoList, {
        animation: 150,
        ghostClass: 'dragging',
        onEnd: function(evt) {
            const todoId = evt.item.dataset.id;
            const newIndex = evt.newIndex;
            
            // サーバーに新しい順序を送信
            htmx.ajax('PATCH', `/api/todos/${todoId}/reorder`, {
                target: '#todo-list',
                values: { position: newIndex }
            });
        }
    });
}

// 一括選択
let selectedTodos = new Set();

function toggleSelection(todoId) {
    if (selectedTodos.has(todoId)) {
        selectedTodos.delete(todoId);
    } else {
        selectedTodos.add(todoId);
    }
    updateBulkActions();
}

function updateBulkActions() {
    const bulkActions = document.querySelector('.bulk-actions');
    const count = selectedTodos.size;
    
    if (count > 0) {
        bulkActions.style.display = 'flex';
        bulkActions.querySelector('.selected-count').textContent = `${count}個選択中`;
    } else {
        bulkActions.style.display = 'none';
    }
}

// 一括操作
function markSelectedComplete() {
    const ids = Array.from(selectedTodos);
    htmx.ajax('PATCH', '/api/todos/bulk-complete', {
        target: '#todo-list',
        values: { ids: ids }
    });
    selectedTodos.clear();
    updateBulkActions();
}

function deleteSelected() {
    if (!confirm('選択したTODOを削除しますか？')) return;
    
    const ids = Array.from(selectedTodos);
    htmx.ajax('DELETE', '/api/todos/bulk-delete', {
        target: '#todo-list',
        values: { ids: ids }
    });
    selectedTodos.clear();
    updateBulkActions();
}

// トースト通知
function showToast(message, type = 'info') {
    const container = document.getElementById('toast-container');
    const toast = document.createElement('div');
    toast.className = `toast toast-${type}`;
    toast.textContent = message;
    
    container.appendChild(toast);
    
    setTimeout(() => {
        toast.style.animation = 'slideOut 0.3s ease';
        setTimeout(() => toast.remove(), 300);
    }, 3000);
}

// HTMXイベントリスナー
document.body.addEventListener('htmx:afterRequest', function(evt) {
    if (evt.detail.successful) {
        if (evt.detail.verb === 'POST') {
            showToast('TODOを作成しました', 'success');
        } else if (evt.detail.verb === 'DELETE') {
            showToast('TODOを削除しました', 'info');
        }
        // カスタムイベントを発火
        htmx.trigger('body', 'todoUpdated');
    }
});

// エクスポート/インポート
function exportTodos() {
    fetch('/api/todos/export')
        .then(response => response.blob())
        .then(blob => {
            const url = window.URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = `todos-${new Date().toISOString().split('T')[0]}.json`;
            a.click();
            window.URL.revokeObjectURL(url);
            showToast('TODOをエクスポートしました', 'success');
        });
}

function importTodos(input) {
    const file = input.files[0];
    if (!file) return;
    
    const formData = new FormData();
    formData.append('file', file);
    
    fetch('/api/todos/import', {
        method: 'POST',
        body: formData
    })
    .then(response => response.json())
    .then(data => {
        showToast(`${data.count}個のTODOをインポートしました`, 'success');
        htmx.trigger('body', 'todoUpdated');
    })
    .catch(error => {
        showToast('インポートに失敗しました', 'error');
    });
    
    input.value = '';
}

// ユーティリティ関数
function formatDate(dateString) {
    if (!dateString) return '期限なし';
    const date = new Date(dateString);
    const today = new Date();
    const diffTime = date - today;
    const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
    
    if (diffDays < 0) return `${Math.abs(diffDays)}日遅れ`;
    if (diffDays === 0) return '今日';
    if (diffDays === 1) return '明日';
    return `${diffDays}日後`;
}

function isPastDue(dateString) {
    if (!dateString) return false;
    return new Date(dateString) < new Date();
}

// オフライン対応
let isOnline = navigator.onLine;

window.addEventListener('online', () => {
    isOnline = true;
    showToast('オンラインに復帰しました', 'success');
    // 保留中の操作を実行
    syncPendingOperations();
});

window.addEventListener('offline', () => {
    isOnline = false;
    showToast('オフラインです', 'warning');
});

// Service Worker登録（PWA対応）
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js');
}
```

## 8.5 バックエンドの実装（Flask）

```python
# app.py
from flask import Flask, request, jsonify, render_template_string
from datetime import datetime
import json
import os
from uuid import uuid4

app = Flask(__name__)

# データストレージ（本番環境ではデータベースを使用）
TODOS_FILE = 'data/todos.json'
CATEGORIES_FILE = 'data/categories.json'

def load_todos():
    if os.path.exists(TODOS_FILE):
        with open(TODOS_FILE, 'r') as f:
            return json.load(f)
    return []

def save_todos(todos):
    os.makedirs('data', exist_ok=True)
    with open(TODOS_FILE, 'w') as f:
        json.dump(todos, f, indent=2)

def load_categories():
    if os.path.exists(CATEGORIES_FILE):
        with open(CATEGORIES_FILE, 'r') as f:
            return json.load(f)
    return ['仕事', 'プライベート', '買い物', '勉強']

# ルートページ
@app.route('/')
def index():
    return open('index.html', 'r').read()

# 静的ファイル
@app.route('/static/<path:path>')
def static_files(path):
    return send_from_directory('static', path)

# API: TODO一覧取得
@app.route('/api/todos')
def get_todos():
    todos = load_todos()
    filter_params = request.args.get('filter', '{}')
    
    try:
        filters = json.loads(filter_params)
    except:
        filters = {}
    
    # フィルタリング
    if filters.get('status') == 'active':
        todos = [t for t in todos if not t.get('completed')]
    elif filters.get('status') == 'completed':
        todos = [t for t in todos if t.get('completed')]
    
    if filters.get('category'):
        todos = [t for t in todos if t.get('category') == filters['category']]
    
    # ソート（位置順）
    todos.sort(key=lambda x: x.get('position', 0))
    
    # HTMLレンダリング
    html = ''
    for todo in todos:
        html += render_todo_item(todo)
    
    if not html:
        html = '<div class="empty-state">TODOがありません</div>'
    
    return html

# API: TODO作成
@app.route('/api/todos', methods=['POST'])
def create_todo():
    todos = load_todos()
    
    new_todo = {
        'id': str(uuid4()),
        'title': request.form.get('title'),
        'completed': False,
        'priority': request.form.get('priority', 'medium'),
        'due_date': request.form.get('due_date'),
        'category': request.form.get('category'),
        'created_at': datetime.now().isoformat(),
        'position': 0  # 最上部に追加
    }
    
    # 既存のTODOの位置を調整
    for todo in todos:
        todo['position'] = todo.get('position', 0) + 1
    
    todos.insert(0, new_todo)
    save_todos(todos)
    
    return render_todo_item(new_todo)

# API: TODO更新（完了/未完了の切り替え）
@app.route('/api/todos/<todo_id>/toggle', methods=['PATCH'])
def toggle_todo(todo_id):
    todos = load_todos()
    
    for todo in todos:
        if todo['id'] == todo_id:
            todo['completed'] = not todo.get('completed', False)
            todo['updated_at'] = datetime.now().isoformat()
            save_todos(todos)
            return render_todo_item(todo)
    
    return 'TODO not found', 404

# API: TODO削除
@app.route('/api/todos/<todo_id>', methods=['DELETE'])
def delete_todo(todo_id):
    todos = load_todos()
    todos = [t for t in todos if t['id'] != todo_id]
    save_todos(todos)
    return ''

# API: TODO編集フォーム
@app.route('/api/todos/<todo_id>/edit')
def edit_todo_form(todo_id):
    todos = load_todos()
    todo = next((t for t in todos if t['id'] == todo_id), None)
    
    if not todo:
        return 'TODO not found', 404
    
    return f'''
    <form class="todo-item editing" 
          hx-put="/api/todos/{todo_id}"
          hx-target="#todo-{todo_id}"
          hx-swap="outerHTML">
        <input type="text" 
               name="title" 
               value="{todo['title']}"
               class="edit-input"
               autofocus>
        <button type="submit" class="btn btn-sm">保存</button>
        <button type="button" 
                class="btn btn-sm"
                hx-get="/api/todos/{todo_id}"
                hx-target="#todo-{todo_id}"
                hx-swap="outerHTML">
            キャンセル
        </button>
    </form>
    '''

# API: TODO更新
@app.route('/api/todos/<todo_id>', methods=['PUT'])
def update_todo(todo_id):
    todos = load_todos()
    
    for todo in todos:
        if todo['id'] == todo_id:
            todo['title'] = request.form.get('title', todo['title'])
            todo['updated_at'] = datetime.now().isoformat()
            save_todos(todos)
            return render_todo_item(todo)
    
    return 'TODO not found', 404

# API: 統計情報
@app.route('/api/stats')
def get_stats():
    todos = load_todos()
    total = len(todos)
    completed = len([t for t in todos if t.get('completed')])
    active = total - completed
    
    return f'''
    <div class="stat-item">
        <i class="fas fa-list"></i>
        <span>全体: {total}</span>
    </div>
    <div class="stat-item">
        <i class="fas fa-clock"></i>
        <span>未完了: {active}</span>
    </div>
    <div class="stat-item">
        <i class="fas fa-check-circle"></i>
        <span>完了: {completed}</span>
    </div>
    '''

# ヘルパー関数
def render_todo_item(todo):
    completed_class = 'completed' if todo.get('completed') else ''
    checked = 'checked' if todo.get('completed') else ''
    priority = todo.get('priority', 'medium')
    category = todo.get('category', '未分類')
    due_date = todo.get('due_date', '')
    
    # 期限チェック
    overdue_class = ''
    if due_date and not todo.get('completed'):
        if datetime.fromisoformat(due_date) < datetime.now():
            overdue_class = 'overdue'
    
    return f'''
    <div class="todo-item {completed_class} priority-{priority}" 
         id="todo-{todo['id']}"
         data-id="{todo['id']}"
         draggable="true">
        
        <div class="todo-checkbox">
            <input type="checkbox" 
                   id="check-{todo['id']}"
                   {checked}
                   hx-patch="/api/todos/{todo['id']}/toggle"
                   hx-target="#todo-{todo['id']}"
                   hx-swap="outerHTML">
        </div>
        
        <div class="todo-content">
            <div class="todo-title" 
                 hx-get="/api/todos/{todo['id']}/edit"
                 hx-trigger="dblclick"
                 hx-target="#todo-{todo['id']}"
                 hx-swap="outerHTML">
                {todo['title']}
            </div>
            
            <div class="todo-meta">
                <span class="category">
                    <i class="fas fa-tag"></i> {category}
                </span>
                {f'<span class="due-date {overdue_class}"><i class="fas fa-calendar"></i> {format_date(due_date)}</span>' if due_date else ''}
            </div>
        </div>
        
        <div class="todo-actions">
            <button class="btn-icon" 
                    hx-delete="/api/todos/{todo['id']}"
                    hx-target="#todo-{todo['id']}"
                    hx-swap="outerHTML swap:0.3s"
                    hx-confirm="このTODOを削除しますか？"
                    title="削除">
                <i class="fas fa-trash"></i>
            </button>
        </div>
    </div>
    '''

def format_date(date_string):
    if not date_string:
        return ''
    
    date = datetime.fromisoformat(date_string)
    today = datetime.now().date()
    diff = (date.date() - today).days
    
    if diff < 0:
        return f'{abs(diff)}日遅れ'
    elif diff == 0:
        return '今日'
    elif diff == 1:
        return '明日'
    else:
        return f'{diff}日後'

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

## 8.6 拡張機能とベストプラクティス

### Progressive Web App（PWA）対応

```javascript
// sw.js - Service Worker
const CACHE_NAME = 'todo-app-v1';
const urlsToCache = [
    '/',
    '/static/css/style.css',
    '/static/js/app.js',
    'https://unpkg.com/htmx.org@1.9.10',
    'https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css'
];

self.addEventListener('install', event => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then(cache => cache.addAll(urlsToCache))
    );
});

self.addEventListener('fetch', event => {
    event.respondWith(
        caches.match(event.request)
            .then(response => {
                // キャッシュがあればそれを返す
                if (response) {
                    return response;
                }
                
                // なければネットワークから取得
                return fetch(event.request)
                    .then(response => {
                        // 成功したレスポンスをキャッシュに追加
                        if (!response || response.status !== 200) {
                            return response;
                        }
                        
                        const responseToCache = response.clone();
                        caches.open(CACHE_NAME)
                            .then(cache => {
                                cache.put(event.request, responseToCache);
                            });
                        
                        return response;
                    });
            })
    );
});
```

### マニフェストファイル

```json
{
    "name": "HTMX TODO App",
    "short_name": "TODO",
    "description": "HTMXで作成したTODOアプリケーション",
    "start_url": "/",
    "display": "standalone",
    "background_color": "#ffffff",
    "theme_color": "#4CAF50",
    "icons": [
        {
            "src": "/static/images/icon-192.png",
            "sizes": "192x192",
            "type": "image/png"
        },
        {
            "src": "/static/images/icon-512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ]
}
```

## まとめ

この章では、HTMXを使用して完全に機能するTODOアプリケーションを構築しました。実装した主な機能：

1. **基本的なCRUD操作**: 作成、読み取り、更新、削除
2. **高度なUI機能**: ドラッグ&ドロップ、インライン編集、リアルタイム検索
3. **データ管理**: フィルタリング、ソート、カテゴリー分け
4. **ユーザビリティ**: ローディング表示、エラーハンドリング、トースト通知
5. **拡張機能**: PWA対応、オフライン機能、エクスポート/インポート

HTMXの強力な機能により、最小限のJavaScriptでリッチなユーザー体験を実現できました。

## 練習問題と発展課題

### 問題1: 機能追加
以下の機能を追加してください：
1. TODOにタグを複数設定できる機能
2. 繰り返しTODO（毎日、毎週など）
3. TODOにファイル添付機能

### 問題2: パフォーマンス最適化
アプリケーションのパフォーマンスを改善してください：
1. 仮想スクロール実装（大量のTODO対応）
2. 画像の遅延読み込み
3. キャッシュ戦略の最適化

### 問題3: 共同作業機能
複数ユーザーでの利用を想定した機能を追加：
1. ユーザー認証
2. TODO の共有機能
3. リアルタイム同期（WebSocket使用）

### 問題4: 分析機能
TODOの完了状況を分析する機能を追加：
1. 完了率のグラフ表示
2. カテゴリー別の統計
3. 生産性レポート

### 問題5: モバイル最適化
モバイルデバイスでの使い勝手を向上：
1. スワイプジェスチャー対応
2. プッシュ通知
3. オフライン同期の改善

---

**最終課題：**

このTODOアプリケーションをベースに、あなた独自のアイデアを追加して、より便利で使いやすいアプリケーションに発展させてください。HTMXの可能性は無限大です！

**お疲れさまでした！** これでHTMXチュートリアルは完了です。HTMXを使って素晴らしいWebアプリケーションを作成してください！