# 第2章: HTMXの導入とセットアップ

## 章の概要

この章では、HTMXを実際にプロジェクトに導入する方法を学びます。様々なインストール方法、開発環境の構築、そして最初のHTMXアプリケーションの作成まで、ステップバイステップで進めていきます。

## 2.1 HTMXのインストール方法

HTMXは複数の方法でプロジェクトに追加できます。プロジェクトの要件に応じて最適な方法を選択してください。

### 方法1: CDN経由での読み込み（最も簡単）

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HTMX入門</title>
    <!-- HTMXをCDNから読み込む -->
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
</head>
<body>
    <!-- ここにHTMXを使ったコンテンツを記述 -->
</body>
</html>
```

**メリット:**
- セットアップが不要
- すぐに始められる
- プロトタイピングに最適

**デメリット:**
- インターネット接続が必要
- CDNの可用性に依存

### 方法2: ローカルファイルとして配置

1. HTMXの公式サイトからファイルをダウンロード
2. プロジェクトの適切なディレクトリに配置

```bash
# プロジェクト構造の例
project/
├── index.html
├── js/
│   └── htmx.min.js
└── css/
    └── styles.css
```

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>HTMX入門</title>
    <!-- ローカルのHTMXファイルを読み込む -->
    <script src="js/htmx.min.js"></script>
</head>
<body>
    <!-- コンテンツ -->
</body>
</html>
```

### 方法3: npmを使用したインストール

Node.jsプロジェクトの場合：

```bash
# npmを使用
npm install htmx.org

# またはyarnを使用
yarn add htmx.org
```

使用例（webpack等のバンドラーを使用する場合）：

```javascript
// JavaScriptファイル内でインポート
import 'htmx.org';

// またはrequire
require('htmx.org');
```

### 方法4: ES6モジュールとして使用

```html
<script type="module">
    import 'https://unpkg.com/htmx.org@1.9.10/dist/htmx.min.js';
</script>
```

## 2.2 開発環境の準備

HTMXアプリケーションを開発するために、ローカルWebサーバーを設定しましょう。

### Pythonを使用する場合

```bash
# Python 3
python -m http.server 8000

# Python 2
python -m SimpleHTTPServer 8000
```

### Node.jsを使用する場合

1. http-serverをインストール：
```bash
npm install -g http-server
```

2. サーバーを起動：
```bash
http-server -p 8000
```

### VS Code Live Server拡張機能

1. VS Codeで「Live Server」拡張機能をインストール
2. HTMLファイルを右クリック → 「Open with Live Server」

## 2.3 プロジェクト構造の設計

典型的なHTMXプロジェクトの構造：

```
htmx-project/
├── index.html          # メインのHTMLファイル
├── server/             # サーバーサイドのコード
│   ├── app.py         # Pythonの例
│   └── routes/        # APIエンドポイント
├── static/            # 静的ファイル
│   ├── css/
│   │   └── style.css
│   ├── js/
│   │   └── htmx.min.js
│   └── images/
├── templates/         # HTMLテンプレート（部分的なHTML）
│   ├── header.html
│   └── footer.html
└── README.md
```

## 2.4 最初のHTMXアプリケーション

実際に動作する簡単なHTMXアプリケーションを作成してみましょう。

### Step 1: HTMLファイルの作成

`index.html`:
```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>初めてのHTMXアプリ</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        }
        .container {
            background-color: #f5f5f5;
            padding: 20px;
            border-radius: 8px;
            margin-top: 20px;
        }
        button {
            background-color: #007bff;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 4px;
            cursor: pointer;
        }
        button:hover {
            background-color: #0056b3;
        }
        #content {
            margin-top: 20px;
            padding: 15px;
            background-color: white;
            border-radius: 4px;
            min-height: 50px;
        }
    </style>
</head>
<body>
    <h1>初めてのHTMXアプリケーション</h1>
    
    <div class="container">
        <h2>クリックでコンテンツを更新</h2>
        <button hx-get="/api/greeting" 
                hx-target="#content"
                hx-swap="innerHTML">
            挨拶を取得
        </button>
        
        <button hx-get="/api/time" 
                hx-target="#content"
                hx-swap="innerHTML">
            現在時刻を取得
        </button>
        
        <div id="content">
            ここにコンテンツが表示されます
        </div>
    </div>
</body>
</html>
```

### Step 2: サーバーサイドの実装（Python Flask版）

`server.py`:
```python
from flask import Flask, render_template_string
from datetime import datetime
import random

app = Flask(__name__)

# メインページ
@app.route('/')
def index():
    return open('index.html', 'r', encoding='utf-8').read()

# 挨拶を返すAPI
@app.route('/api/greeting')
def greeting():
    greetings = [
        "こんにちは！HTMXへようこそ！",
        "やあ！HTMXは楽しいですね！",
        "HTMXで開発を始めましょう！",
        "サーバーサイドレンダリングは素晴らしい！"
    ]
    selected = random.choice(greetings)
    return f'<div class="greeting">{selected}</div>'

# 現在時刻を返すAPI
@app.route('/api/time')
def current_time():
    now = datetime.now()
    formatted_time = now.strftime('%Y年%m月%d日 %H:%M:%S')
    return f'''
    <div class="time-display">
        <h3>現在時刻</h3>
        <p>{formatted_time}</p>
    </div>
    '''

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

### Step 3: Node.js Express版（代替案）

`server.js`:
```javascript
const express = require('express');
const app = express();
const path = require('path');

// 静的ファイルの提供
app.use(express.static('.'));

// 挨拶を返すAPI
app.get('/api/greeting', (req, res) => {
    const greetings = [
        "こんにちは！HTMXへようこそ！",
        "やあ！HTMXは楽しいですね！",
        "HTMXで開発を始めましょう！",
        "サーバーサイドレンダリングは素晴らしい！"
    ];
    const selected = greetings[Math.floor(Math.random() * greetings.length)];
    res.send(`<div class="greeting">${selected}</div>`);
});

// 現在時刻を返すAPI
app.get('/api/time', (req, res) => {
    const now = new Date();
    const formatted = now.toLocaleString('ja-JP');
    res.send(`
        <div class="time-display">
            <h3>現在時刻</h3>
            <p>${formatted}</p>
        </div>
    `);
});

app.listen(5000, () => {
    console.log('Server is running on http://localhost:5000');
});
```

## 2.5 HTMXの動作確認

### ブラウザでの確認手順

1. サーバーを起動
2. ブラウザで `http://localhost:5000` を開く
3. 開発者ツールのNetworkタブを開く
4. ボタンをクリックして、リクエストとレスポンスを確認

### 開発者ツールでの確認ポイント

- **Networkタブ**: XHRリクエストの確認
- **Elementsタブ**: DOM更新の確認
- **Consoleタブ**: エラーメッセージの確認

## 2.6 トラブルシューティング

### よくある問題と解決方法

#### 1. HTMXが動作しない

```html
<!-- 確認事項 -->
<!-- 1. HTMXが正しく読み込まれているか -->
<script>
    console.log(typeof htmx !== 'undefined' ? 'HTMX loaded' : 'HTMX not loaded');
</script>

<!-- 2. 属性名が正しいか（hx-で始まる） -->
<button hx-get="/api/data">正しい</button>
<button htmx-get="/api/data">間違い</button>
```

#### 2. CORSエラー

```python
# Flask でCORSを有効にする
from flask_cors import CORS
CORS(app)
```

#### 3. 404エラー

- URLパスが正しいか確認
- サーバーが起動しているか確認
- エンドポイントが定義されているか確認

## 2.7 開発のベストプラクティス

### 1. プログレッシブエンハンスメント

```html
<!-- JavaScriptが無効でも動作する基本機能 -->
<form action="/search" method="get">
    <input type="search" name="q">
    <button type="submit">検索</button>
</form>

<!-- HTMXで強化 -->
<form hx-get="/search" hx-target="#results">
    <input type="search" name="q">
    <button type="submit">検索</button>
</form>
<div id="results"></div>
```

### 2. エラーハンドリング

```html
<!-- エラー時の表示先を指定 -->
<div hx-get="/api/data" 
     hx-target="#content"
     hx-error-target="#error">
    データを読み込む
</div>
<div id="error"></div>
```

### 3. ローディング表示

```html
<button hx-get="/api/slow-endpoint" 
        hx-target="#result"
        hx-indicator="#spinner">
    データ取得
</button>
<span id="spinner" class="htmx-indicator">読み込み中...</span>
```

## まとめ

この章では、HTMXの導入方法と基本的なセットアップを学びました：

1. 様々なインストール方法（CDN、npm、ローカルファイル）
2. 開発環境の構築
3. 基本的なプロジェクト構造
4. 最初のHTMXアプリケーションの作成
5. トラブルシューティングとベストプラクティス

次章では、HTMXの中核となる属性である`hx-get`と`hx-post`について詳しく学びます。

## 練習問題

### 問題1: 環境構築
以下の要件を満たすHTMXプロジェクトをセットアップしてください：
1. ローカルにHTMXファイルを配置
2. 基本的なフォルダ構造を作成
3. 簡単なインデックスページを作成

### 問題2: 動的コンテンツ
以下の機能を持つページを作成してください：
1. 3つのボタン（朝の挨拶、昼の挨拶、夜の挨拶）
2. 各ボタンをクリックすると、適切な挨拶メッセージを表示
3. メッセージには現在時刻も含める

### 問題3: エラーハンドリング
意図的にエラーを発生させるボタンを作成し、エラーメッセージを適切に表示する仕組みを実装してください。

### 問題4: 複数の更新
1つのボタンクリックで、ページ内の2つの異なる領域を更新する方法を考えて実装してください。

### 問題5: 開発環境の最適化
HTMXアプリケーション開発のための、あなたなりの最適な開発環境構成を提案してください。使用するツール、フォルダ構造、ワークフローなどを含めてください。

---

**解答例は第3章の最後に掲載します。まずは自分で実装してみましょう！**