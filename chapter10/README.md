# 第10章: データベースとの連携

## 章の概要

この章では、HTMXアプリケーションとデータベースを効果的に連携させる方法を学びます。リレーショナルデータベース（MySQL、PostgreSQL）とNoSQLデータベース（MongoDB、Redis）の両方をカバーし、実践的なCRUD操作、パフォーマンス最適化、セキュリティ対策について詳しく解説します。

## 10.1 データベース連携の基礎

### なぜデータベース連携が重要か

HTMXはサーバーサイドレンダリングを基本とするため、データベースとの効率的な連携が重要です：

1. **データの永続性**: アプリケーションデータの長期保存
2. **スケーラビリティ**: 大量のデータの効率的な管理
3. **データ整合性**: トランザクションによる一貫性の保証
4. **検索とフィルタリング**: 複雑なクエリの実行
5. **同時実行制御**: 複数ユーザーの同時アクセス対応

### アーキテクチャパターン

```
[ブラウザ] <-- HTMX --> [Webサーバー] <-- ORM/ドライバー --> [データベース]
                              |
                              ├── ビジネスロジック
                              ├── バリデーション
                              └── HTMLレンダリング
```

## 10.2 SQLデータベースとの連携

### SQLAlchemy（Python）を使用した例

```python
# models.py - データベースモデル
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

db = SQLAlchemy()

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    posts = db.relationship('Post', backref='author', lazy=True)

class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    tags = db.relationship('Tag', secondary='post_tags', backref='posts')

class Tag(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True, nullable=False)

# 多対多の中間テーブル
post_tags = db.Table('post_tags',
    db.Column('post_id', db.Integer, db.ForeignKey('post.id'), primary_key=True),
    db.Column('tag_id', db.Integer, db.ForeignKey('tag.id'), primary_key=True)
)
```

```python
# app.py - Flaskアプリケーション
from flask import Flask, request, render_template_string
from models import db, User, Post, Tag
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get('DATABASE_URL', 'sqlite:///blog.db')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db.init_app(app)

# データベース初期化
with app.app_context():
    db.create_all()

# ユーザー一覧（ページネーション付き）
@app.route('/api/users')
def list_users():
    page = request.args.get('page', 1, type=int)
    per_page = 10
    
    users = User.query.paginate(
        page=page, 
        per_page=per_page, 
        error_out=False
    )
    
    return render_template_string('''
    <div class="user-list">
        {% for user in users.items %}
        <div class="user-card" id="user-{{ user.id }}">
            <h3>{{ user.username }}</h3>
            <p>{{ user.email }}</p>
            <p>投稿数: {{ user.posts|length }}</p>
            <div class="actions">
                <button hx-get="/api/users/{{ user.id }}/edit"
                        hx-target="#user-{{ user.id }}"
                        hx-swap="outerHTML">
                    編集
                </button>
                <button hx-delete="/api/users/{{ user.id }}"
                        hx-target="#user-{{ user.id }}"
                        hx-swap="outerHTML"
                        hx-confirm="このユーザーを削除しますか？">
                    削除
                </button>
            </div>
        </div>
        {% endfor %}
    </div>
    
    <!-- ページネーション -->
    <div class="pagination">
        {% if users.has_prev %}
        <button hx-get="/api/users?page={{ users.prev_num }}"
                hx-target="#user-container">
            前へ
        </button>
        {% endif %}
        
        <span>{{ users.page }} / {{ users.pages }}</span>
        
        {% if users.has_next %}
        <button hx-get="/api/users?page={{ users.next_num }}"
                hx-target="#user-container">
            次へ
        </button>
        {% endif %}
    </div>
    ''', users=users)

# ユーザー作成
@app.route('/api/users', methods=['POST'])
def create_user():
    username = request.form.get('username')
    email = request.form.get('email')
    
    # バリデーション
    if User.query.filter_by(username=username).first():
        return '<div class="error">ユーザー名は既に使用されています</div>', 400
    
    if User.query.filter_by(email=email).first():
        return '<div class="error">メールアドレスは既に登録されています</div>', 400
    
    # ユーザー作成
    user = User(username=username, email=email)
    db.session.add(user)
    
    try:
        db.session.commit()
        return render_template_string('''
        <div class="user-card" id="user-{{ user.id }}">
            <h3>{{ user.username }}</h3>
            <p>{{ user.email }}</p>
            <p>作成日: {{ user.created_at.strftime('%Y-%m-%d') }}</p>
        </div>
        ''', user=user)
    except Exception as e:
        db.session.rollback()
        return f'<div class="error">エラーが発生しました: {str(e)}</div>', 500

# 投稿の検索（全文検索）
@app.route('/api/posts/search')
def search_posts():
    query = request.args.get('q', '')
    
    if len(query) < 2:
        return '<div class="info">2文字以上入力してください</div>'
    
    # SQLiteの場合
    posts = Post.query.filter(
        db.or_(
            Post.title.contains(query),
            Post.content.contains(query)
        )
    ).order_by(Post.created_at.desc()).limit(20).all()
    
    # PostgreSQLの場合（全文検索）
    # posts = Post.query.filter(
    #     db.or_(
    #         Post.title.match(query),
    #         Post.content.match(query)
    #     )
    # ).all()
    
    if not posts:
        return '<div class="no-results">検索結果がありません</div>'
    
    return render_template_string('''
    <div class="search-results">
        {% for post in posts %}
        <article class="post-preview">
            <h3>
                <a href="/posts/{{ post.id }}">{{ post.title }}</a>
            </h3>
            <p>{{ post.content[:150] }}...</p>
            <div class="meta">
                <span>by {{ post.author.username }}</span>
                <span>{{ post.created_at.strftime('%Y-%m-%d') }}</span>
            </div>
        </article>
        {% endfor %}
    </div>
    ''', posts=posts)

# トランザクション処理の例
@app.route('/api/posts/<int:post_id>/transfer', methods=['POST'])
def transfer_post(post_id):
    new_user_id = request.form.get('user_id', type=int)
    
    try:
        # トランザクション開始
        post = Post.query.get_or_404(post_id)
        old_user = post.author
        new_user = User.query.get_or_404(new_user_id)
        
        # 投稿の所有者を変更
        post.user_id = new_user_id
        
        # 統計情報の更新（仮想的な例）
        old_user.post_count = len(old_user.posts)
        new_user.post_count = len(new_user.posts)
        
        db.session.commit()
        
        return '<div class="success">投稿の所有者を変更しました</div>'
        
    except Exception as e:
        db.session.rollback()
        return f'<div class="error">エラー: {str(e)}</div>', 500
```

### 高度なクエリパターン

```python
# 複雑なフィルタリング
@app.route('/api/posts/filter')
def filter_posts():
    # パラメータ取得
    tag = request.args.get('tag')
    author = request.args.get('author')
    date_from = request.args.get('from')
    date_to = request.args.get('to')
    sort_by = request.args.get('sort', 'created_at')
    order = request.args.get('order', 'desc')
    
    # クエリビルダー
    query = Post.query
    
    # タグフィルタ
    if tag:
        query = query.join(Post.tags).filter(Tag.name == tag)
    
    # 著者フィルタ
    if author:
        query = query.join(Post.author).filter(User.username == author)
    
    # 日付範囲フィルタ
    if date_from:
        query = query.filter(Post.created_at >= datetime.fromisoformat(date_from))
    if date_to:
        query = query.filter(Post.created_at <= datetime.fromisoformat(date_to))
    
    # ソート
    if order == 'desc':
        query = query.order_by(getattr(Post, sort_by).desc())
    else:
        query = query.order_by(getattr(Post, sort_by))
    
    # 実行
    posts = query.all()
    
    return render_posts(posts)

# 集計クエリ
@app.route('/api/stats/posts')
def post_statistics():
    # 月別投稿数
    monthly_stats = db.session.query(
        db.func.strftime('%Y-%m', Post.created_at).label('month'),
        db.func.count(Post.id).label('count')
    ).group_by('month').order_by('month').all()
    
    # タグ別投稿数
    tag_stats = db.session.query(
        Tag.name,
        db.func.count(Post.id).label('count')
    ).join(post_tags).join(Post).group_by(Tag.name).order_by('count').all()
    
    # ユーザー別投稿数（上位10人）
    top_authors = db.session.query(
        User.username,
        db.func.count(Post.id).label('post_count')
    ).join(Post).group_by(User.id).order_by('post_count').limit(10).all()
    
    return render_template_string('''
    <div class="statistics">
        <div class="stat-section">
            <h3>月別投稿数</h3>
            <div class="chart" id="monthly-chart">
                {% for month, count in monthly_stats %}
                <div class="bar" style="height: {{ count * 10 }}px">
                    <span class="value">{{ count }}</span>
                    <span class="label">{{ month }}</span>
                </div>
                {% endfor %}
            </div>
        </div>
        
        <div class="stat-section">
            <h3>人気タグ</h3>
            <div class="tag-cloud">
                {% for tag, count in tag_stats %}
                <span class="tag" style="font-size: {{ 10 + count * 2 }}px">
                    {{ tag }} ({{ count }})
                </span>
                {% endfor %}
            </div>
        </div>
    </div>
    ''', monthly_stats=monthly_stats, tag_stats=tag_stats)
```

## 10.3 NoSQLデータベースとの連携

### MongoDB（pymongo）の例

```python
from flask import Flask, request, jsonify
from pymongo import MongoClient
from bson import ObjectId
import datetime

app = Flask(__name__)

# MongoDB接続
client = MongoClient('mongodb://localhost:27017/')
db = client['htmx_app']

# コレクション
users = db.users
posts = db.posts
activities = db.activities

# ユーザーアクティビティの記録
@app.route('/api/activities', methods=['POST'])
def log_activity():
    activity = {
        'user_id': request.form.get('user_id'),
        'action': request.form.get('action'),
        'target': request.form.get('target'),
        'timestamp': datetime.datetime.utcnow(),
        'metadata': {
            'ip': request.remote_addr,
            'user_agent': request.headers.get('User-Agent'),
            'session_id': request.cookies.get('session_id')
        }
    }
    
    activities.insert_one(activity)
    
    # リアルタイムダッシュボード更新
    return render_activity_feed()

# アクティビティフィードの表示
@app.route('/api/activities/feed')
def activity_feed():
    # 最新のアクティビティを取得
    recent_activities = activities.find().sort('timestamp', -1).limit(20)
    
    return render_template_string('''
    <div class="activity-feed">
        {% for activity in activities %}
        <div class="activity-item">
            <div class="activity-icon">
                {% if activity.action == 'create' %}
                    <i class="fas fa-plus"></i>
                {% elif activity.action == 'update' %}
                    <i class="fas fa-edit"></i>
                {% elif activity.action == 'delete' %}
                    <i class="fas fa-trash"></i>
                {% endif %}
            </div>
            <div class="activity-content">
                <p>
                    ユーザー {{ activity.user_id }} が
                    {{ activity.target }} を
                    {{ activity.action }} しました
                </p>
                <time>{{ activity.timestamp.strftime('%Y-%m-%d %H:%M') }}</time>
            </div>
        </div>
        {% endfor %}
    </div>
    ''', activities=recent_activities)

# ドキュメントの柔軟な構造
@app.route('/api/products', methods=['POST'])
def create_product():
    product = {
        'name': request.form.get('name'),
        'price': float(request.form.get('price')),
        'category': request.form.get('category'),
        'attributes': {},  # 動的な属性
        'created_at': datetime.datetime.utcnow()
    }
    
    # カテゴリーに応じて異なる属性を追加
    if product['category'] == 'electronics':
        product['attributes'] = {
            'brand': request.form.get('brand'),
            'warranty': request.form.get('warranty'),
            'specs': {
                'cpu': request.form.get('cpu'),
                'ram': request.form.get('ram'),
                'storage': request.form.get('storage')
            }
        }
    elif product['category'] == 'clothing':
        product['attributes'] = {
            'size': request.form.getlist('sizes'),
            'color': request.form.getlist('colors'),
            'material': request.form.get('material')
        }
    
    result = db.products.insert_one(product)
    product['_id'] = str(result.inserted_id)
    
    return render_product_card(product)

# 複雑な集計パイプライン
@app.route('/api/analytics/sales')
def sales_analytics():
    pipeline = [
        # 期間フィルタ
        {
            '$match': {
                'created_at': {
                    '$gte': datetime.datetime.utcnow() - datetime.timedelta(days=30)
                }
            }
        },
        # カテゴリー別グループ化
        {
            '$group': {
                '_id': '$category',
                'total_sales': {'$sum': '$price'},
                'count': {'$sum': 1},
                'avg_price': {'$avg': '$price'}
            }
        },
        # ソート
        {
            '$sort': {'total_sales': -1}
        }
    ]
    
    results = list(db.products.aggregate(pipeline))
    
    return render_template_string('''
    <div class="analytics-dashboard">
        <h3>過去30日間の売上分析</h3>
        <table class="stats-table">
            <thead>
                <tr>
                    <th>カテゴリー</th>
                    <th>売上合計</th>
                    <th>販売数</th>
                    <th>平均価格</th>
                </tr>
            </thead>
            <tbody>
                {% for stat in stats %}
                <tr>
                    <td>{{ stat._id }}</td>
                    <td>¥{{ "{:,.0f}".format(stat.total_sales) }}</td>
                    <td>{{ stat.count }}</td>
                    <td>¥{{ "{:,.0f}".format(stat.avg_price) }}</td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
    ''', stats=results)
```

### Redis（キャッシュとリアルタイム機能）

```python
import redis
import json
from flask import Flask, request, Response

app = Flask(__name__)
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# キャッシュデコレーター
def cache_result(expiration=300):
    def decorator(f):
        def wrapper(*args, **kwargs):
            # キャッシュキーの生成
            cache_key = f"cache:{request.path}:{request.query_string.decode()}"
            
            # キャッシュから取得
            cached = r.get(cache_key)
            if cached:
                return cached
            
            # 関数実行
            result = f(*args, **kwargs)
            
            # キャッシュに保存
            r.setex(cache_key, expiration, result)
            
            return result
        
        wrapper.__name__ = f.__name__
        return wrapper
    return decorator

# リアルタイムランキング
@app.route('/api/posts/<int:post_id>/vote', methods=['POST'])
def vote_post(post_id):
    vote_type = request.form.get('type', 'up')
    user_id = request.form.get('user_id')
    
    # 重複投票チェック
    vote_key = f"vote:{post_id}:{user_id}"
    if r.exists(vote_key):
        return '<div class="error">既に投票済みです</div>', 400
    
    # 投票を記録
    r.setex(vote_key, 86400, vote_type)  # 24時間有効
    
    # スコアを更新
    if vote_type == 'up':
        score = r.zincrby('post_ranking', 1, post_id)
    else:
        score = r.zincrby('post_ranking', -1, post_id)
    
    return f'<div class="vote-count">{int(score)}</div>'

# ランキング表示
@app.route('/api/posts/ranking')
def post_ranking():
    # 上位10件を取得
    top_posts = r.zrevrange('post_ranking', 0, 9, withscores=True)
    
    # 投稿情報を取得
    posts_data = []
    for post_id, score in top_posts:
        # データベースから投稿情報を取得（SQLAlchemyと組み合わせ）
        post = Post.query.get(int(post_id))
        if post:
            posts_data.append({
                'post': post,
                'score': int(score)
            })
    
    return render_template_string('''
    <div class="ranking">
        <h3>人気の投稿</h3>
        {% for item in posts %}
        <div class="ranking-item">
            <span class="rank">{{ loop.index }}</span>
            <div class="post-info">
                <a href="/posts/{{ item.post.id }}">{{ item.post.title }}</a>
                <span class="score">{{ item.score }} ポイント</span>
            </div>
            <div class="vote-buttons">
                <button hx-post="/api/posts/{{ item.post.id }}/vote"
                        hx-vals='{"type": "up"}'
                        hx-target="find .score">
                    👍
                </button>
                <button hx-post="/api/posts/{{ item.post.id }}/vote"
                        hx-vals='{"type": "down"}'
                        hx-target="find .score">
                    👎
                </button>
            </div>
        </div>
        {% endfor %}
    </div>
    ''', posts=posts_data)

# リアルタイム通知（Pub/Sub）
@app.route('/api/notifications/subscribe')
def subscribe_notifications():
    user_id = request.args.get('user_id')
    
    def generate():
        pubsub = r.pubsub()
        pubsub.subscribe(f'notifications:{user_id}')
        
        for message in pubsub.listen():
            if message['type'] == 'message':
                data = json.loads(message['data'])
                yield f'data: {json.dumps(data)}\n\n'
    
    return Response(generate(), mimetype='text/event-stream')

# 通知の送信
@app.route('/api/notifications/send', methods=['POST'])
def send_notification():
    user_id = request.form.get('user_id')
    message = request.form.get('message')
    notification_type = request.form.get('type', 'info')
    
    notification = {
        'message': message,
        'type': notification_type,
        'timestamp': datetime.datetime.utcnow().isoformat()
    }
    
    # Pub/Subで送信
    r.publish(f'notifications:{user_id}', json.dumps(notification))
    
    return '<div class="success">通知を送信しました</div>'
```

## 10.4 データベース接続の最適化

### コネクションプーリング

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

# コネクションプールの設定
engine = create_engine(
    'postgresql://user:password@localhost/dbname',
    poolclass=QueuePool,
    pool_size=20,          # プールサイズ
    max_overflow=0,        # 最大オーバーフロー
    pool_pre_ping=True,    # 接続の事前確認
    pool_recycle=3600      # 1時間で接続をリサイクル
)

# Flask-SQLAlchemyの設定
app.config['SQLALCHEMY_ENGINE_OPTIONS'] = {
    'pool_size': 10,
    'pool_recycle': 3600,
    'pool_pre_ping': True
}
```

### 遅延読み込みとEager Loading

```python
# N+1問題の回避
@app.route('/api/posts/with-comments')
def posts_with_comments():
    # 遅延読み込み（N+1問題が発生）
    # posts = Post.query.all()
    # for post in posts:
    #     comments = post.comments  # 各投稿ごとにクエリが実行される
    
    # Eager Loading（結合して一度に取得）
    posts = Post.query.options(
        db.joinedload(Post.comments),
        db.joinedload(Post.author),
        db.joinedload(Post.tags)
    ).all()
    
    return render_template_string('''
    <div class="posts">
        {% for post in posts %}
        <article class="post">
            <h2>{{ post.title }}</h2>
            <p>by {{ post.author.username }}</p>
            <div class="tags">
                {% for tag in post.tags %}
                <span class="tag">{{ tag.name }}</span>
                {% endfor %}
            </div>
            <div class="comments">
                <h4>コメント ({{ post.comments|length }})</h4>
                {% for comment in post.comments %}
                <div class="comment">
                    {{ comment.content }}
                </div>
                {% endfor %}
            </div>
        </article>
        {% endfor %}
    </div>
    ''', posts=posts)
```

### クエリ最適化

```python
# インデックスの活用
class Post(db.Model):
    __tablename__ = 'posts'
    
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False, index=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow, index=True)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), index=True)
    
    # 複合インデックス
    __table_args__ = (
        db.Index('idx_user_created', 'user_id', 'created_at'),
        db.Index('idx_title_fulltext', 'title', postgresql_using='gin'),
    )

# EXPLAINを使用したクエリ分析
@app.route('/api/debug/query')
def debug_query():
    # クエリ実行計画の確認
    query = Post.query.filter(Post.title.contains('HTMX'))
    
    # SQLAlchemyでEXPLAINを実行
    explain = db.session.execute(
        f'EXPLAIN ANALYZE {query.statement.compile(compile_kwargs={"literal_binds": True})}'
    ).fetchall()
    
    return '<pre>' + '\n'.join([str(row) for row in explain]) + '</pre>'
```

## 10.5 セキュリティ対策

### SQLインジェクション対策

```python
# 危険な例（絶対に使用しない）
@app.route('/api/users/search/bad')
def bad_search():
    name = request.args.get('name')
    # SQLインジェクションの脆弱性
    query = f"SELECT * FROM users WHERE name = '{name}'"
    results = db.session.execute(query)
    
# 安全な例
@app.route('/api/users/search/safe')
def safe_search():
    name = request.args.get('name')
    
    # パラメータ化クエリ（SQLAlchemy）
    users = User.query.filter(User.name == name).all()
    
    # または生のSQLを使う場合
    results = db.session.execute(
        "SELECT * FROM users WHERE name = :name",
        {"name": name}
    ).fetchall()
    
    return render_users(users)
```

### アクセス制御

```python
from functools import wraps
from flask_login import current_user

def require_permission(permission):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not current_user.has_permission(permission):
                return '<div class="error">権限がありません</div>', 403
            return f(*args, **kwargs)
        return decorated_function
    return decorator

@app.route('/api/admin/users/<int:user_id>', methods=['DELETE'])
@require_permission('delete_users')
def delete_user(user_id):
    user = User.query.get_or_404(user_id)
    
    # 監査ログの記録
    audit_log = AuditLog(
        user_id=current_user.id,
        action='delete_user',
        target_id=user_id,
        timestamp=datetime.utcnow()
    )
    db.session.add(audit_log)
    
    db.session.delete(user)
    db.session.commit()
    
    return '<div class="success">ユーザーを削除しました</div>'
```

### データ暗号化

```python
from cryptography.fernet import Fernet
import base64

class EncryptedField:
    def __init__(self, key):
        self.cipher = Fernet(key)
    
    def encrypt(self, value):
        if value is None:
            return None
        return self.cipher.encrypt(value.encode()).decode()
    
    def decrypt(self, value):
        if value is None:
            return None
        return self.cipher.decrypt(value.encode()).decode()

# 使用例
encryption_key = base64.urlsafe_b64encode(b'your-32-byte-secret-key-here---!')
encrypted_field = EncryptedField(encryption_key)

class UserProfile(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'))
    _ssn = db.Column('ssn', db.String(255))  # 暗号化して保存
    
    @property
    def ssn(self):
        return encrypted_field.decrypt(self._ssn)
    
    @ssn.setter
    def ssn(self, value):
        self._ssn = encrypted_field.encrypt(value)
```

## 10.6 パフォーマンスモニタリング

### クエリプロファイリング

```python
import time
from flask import g

# リクエストごとのクエリ計測
@app.before_request
def before_request():
    g.query_count = 0
    g.query_time = 0
    g.queries = []

@app.after_request
def after_request(response):
    if hasattr(g, 'queries'):
        total_time = sum(q['duration'] for q in g.queries)
        response.headers['X-DB-Query-Count'] = str(len(g.queries))
        response.headers['X-DB-Query-Time'] = f"{total_time:.3f}s"
        
        # 開発環境でクエリログを表示
        if app.debug:
            print(f"\n=== Query Log ===")
            print(f"Total queries: {len(g.queries)}")
            print(f"Total time: {total_time:.3f}s")
            for q in g.queries:
                print(f"[{q['duration']:.3f}s] {q['sql'][:100]}...")
    
    return response

# SQLAlchemyイベントリスナー
from sqlalchemy import event

@event.listens_for(db.engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    g.query_start_time = time.time()

@event.listens_for(db.engine, "after_cursor_execute")
def after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    duration = time.time() - g.query_start_time
    g.queries.append({
        'sql': statement,
        'parameters': parameters,
        'duration': duration
    })
```

### キャッシュ戦略

```python
from flask_caching import Cache

cache = Cache(app, config={
    'CACHE_TYPE': 'redis',
    'CACHE_REDIS_URL': 'redis://localhost:6379/0'
})

# クエリ結果のキャッシュ
@app.route('/api/expensive-query')
@cache.cached(timeout=300, key_prefix='expensive_query')
def expensive_query():
    # 重いクエリ
    results = db.session.execute('''
        SELECT 
            u.username,
            COUNT(DISTINCT p.id) as post_count,
            COUNT(DISTINCT c.id) as comment_count,
            AVG(p.view_count) as avg_views
        FROM users u
        LEFT JOIN posts p ON u.id = p.user_id
        LEFT JOIN comments c ON u.id = c.user_id
        GROUP BY u.id
        ORDER BY post_count DESC
        LIMIT 100
    ''').fetchall()
    
    return render_leaderboard(results)

# 条件付きキャッシュ
@app.route('/api/user/<int:user_id>/profile')
def user_profile(user_id):
    # キャッシュキーに条件を含める
    cache_key = f'user_profile:{user_id}:{request.args.get("include_posts", False)}'
    
    cached = cache.get(cache_key)
    if cached:
        return cached
    
    user = User.query.get_or_404(user_id)
    include_posts = request.args.get('include_posts', False)
    
    if include_posts:
        # 投稿も含める
        user = User.query.options(
            db.joinedload(User.posts)
        ).get(user_id)
    
    result = render_user_profile(user, include_posts)
    cache.set(cache_key, result, timeout=300)
    
    return result
```

## 10.7 実践的な統合例

### 完全なブログシステム

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>HTMX Blog with Database</title>
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    <style>
        .blog-container {
            max-width: 1200px;
            margin: 0 auto;
            padding: 20px;
        }
        
        .post-editor {
            background: #f5f5f5;
            padding: 20px;
            border-radius: 8px;
            margin-bottom: 30px;
        }
        
        .post-list {
            display: grid;
            gap: 20px;
        }
        
        .post-card {
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        
        .post-meta {
            color: #666;
            font-size: 0.9em;
            margin-top: 10px;
        }
        
        .filters {
            display: flex;
            gap: 10px;
            margin-bottom: 20px;
        }
        
        .pagination {
            display: flex;
            justify-content: center;
            gap: 10px;
            margin-top: 30px;
        }
        
        .stats-widget {
            background: #e3f2fd;
            padding: 15px;
            border-radius: 8px;
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <div class="blog-container">
        <h1>HTMX Blog System</h1>
        
        <!-- 統計情報ウィジェット -->
        <div class="stats-widget" 
             hx-get="/api/stats/summary"
             hx-trigger="load, postUpdated from:body">
            読み込み中...
        </div>
        
        <!-- 投稿エディタ -->
        <div class="post-editor">
            <h2>新規投稿</h2>
            <form hx-post="/api/posts"
                  hx-target="#post-list"
                  hx-swap="afterbegin"
                  hx-on="htmx:afterRequest: if(event.detail.successful) { this.reset(); htmx.trigger('body', 'postUpdated'); }">
                
                <input type="text" 
                       name="title" 
                       placeholder="タイトル" 
                       required
                       class="form-control">
                
                <textarea name="content" 
                          placeholder="本文" 
                          required
                          rows="5"
                          class="form-control"></textarea>
                
                <input type="text" 
                       name="tags" 
                       placeholder="タグ（カンマ区切り）"
                       class="form-control">
                
                <button type="submit" class="btn btn-primary">投稿</button>
            </form>
        </div>
        
        <!-- フィルター -->
        <div class="filters">
            <select hx-get="/api/posts"
                    hx-target="#post-list"
                    hx-trigger="change"
                    hx-include="[name='search']"
                    name="category">
                <option value="">すべてのカテゴリー</option>
                <option value="tech">技術</option>
                <option value="life">生活</option>
                <option value="news">ニュース</option>
            </select>
            
            <input type="search"
                   name="search"
                   placeholder="検索..."
                   hx-get="/api/posts"
                   hx-trigger="keyup changed delay:500ms"
                   hx-target="#post-list"
                   hx-include="[name='category']">
        </div>
        
        <!-- 投稿リスト -->
        <div id="post-list"
             hx-get="/api/posts"
             hx-trigger="load">
            読み込み中...
        </div>
    </div>
    
    <!-- リアルタイム更新用のSSE -->
    <div hx-sse="connect:/api/posts/stream">
        <div hx-sse="newPost" 
             hx-target="#post-list" 
             hx-swap="afterbegin"></div>
    </div>
</body>
</html>
```

### バックエンドの統合実装

```python
from flask import Flask, request, render_template_string, Response
from flask_sqlalchemy import SQLAlchemy
from flask_caching import Cache
import json

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://localhost/blog'
db = SQLAlchemy(app)
cache = Cache(app, config={'CACHE_TYPE': 'redis'})

# 統計情報サマリー
@app.route('/api/stats/summary')
@cache.cached(timeout=60)
def stats_summary():
    stats = {
        'total_posts': Post.query.count(),
        'total_users': User.query.count(),
        'posts_today': Post.query.filter(
            Post.created_at >= datetime.utcnow().date()
        ).count(),
        'active_users': User.query.filter(
            User.last_seen >= datetime.utcnow() - timedelta(hours=24)
        ).count()
    }
    
    return render_template_string('''
    <div class="stats-grid">
        <div class="stat-item">
            <span class="stat-value">{{ stats.total_posts }}</span>
            <span class="stat-label">総投稿数</span>
        </div>
        <div class="stat-item">
            <span class="stat-value">{{ stats.posts_today }}</span>
            <span class="stat-label">今日の投稿</span>
        </div>
        <div class="stat-item">
            <span class="stat-value">{{ stats.total_users }}</span>
            <span class="stat-label">ユーザー数</span>
        </div>
        <div class="stat-item">
            <span class="stat-value">{{ stats.active_users }}</span>
            <span class="stat-label">アクティブユーザー</span>
        </div>
    </div>
    ''', stats=stats)

# Server-Sent Events for real-time updates
@app.route('/api/posts/stream')
def posts_stream():
    def generate():
        # Redis Pub/Subを使用
        pubsub = redis_client.pubsub()
        pubsub.subscribe('posts:new')
        
        for message in pubsub.listen():
            if message['type'] == 'message':
                post_data = json.loads(message['data'])
                html = render_post_card(post_data)
                yield f'event: newPost\ndata: {html}\n\n'
    
    return Response(generate(), mimetype='text/event-stream')
```

## まとめ

この章では、HTMXアプリケーションとデータベースの連携について学びました：

1. **SQLデータベース**: SQLAlchemyを使った効率的なORM操作
2. **NoSQLデータベース**: MongoDBとRedisの活用方法
3. **パフォーマンス最適化**: コネクションプーリング、クエリ最適化、キャッシュ
4. **セキュリティ**: SQLインジェクション対策、アクセス制御、暗号化
5. **リアルタイム機能**: Redis Pub/SubとSSEによる実装
6. **モニタリング**: クエリプロファイリングとパフォーマンス計測

これらの技術を組み合わせることで、スケーラブルで高性能なHTMXアプリケーションを構築できます。

## 練習問題

### 問題1: 複雑なフィルタリング
以下の要件を満たす商品検索システムを実装してください：
- 価格範囲、カテゴリー、タグによる複合フィルタ
- 並び順の切り替え（価格、人気、新着）
- ファセット検索（各フィルタの件数表示）
- URLパラメータとの同期

### 問題2: トランザクション処理
銀行口座間の送金システムを実装してください：
- ACID特性を満たすトランザクション
- 同時実行制御（楽観的ロック）
- 送金履歴の記録
- エラー時のロールバック

### 問題3: キャッシュ戦略
ニュースサイトのキャッシュシステムを設計してください：
- 記事の人気度に応じた動的なキャッシュ期間
- タグやカテゴリー別のキャッシュ無効化
- CDNとの連携
- キャッシュヒット率の計測

### 問題4: リアルタイムダッシュボード
管理者向けダッシュボードを作成してください：
- リアルタイムのアクセス統計
- データベースの負荷状況
- エラーログのストリーミング
- アラート機能

### 問題5: データ移行ツール
異なるデータベース間でのデータ移行ツールを作成してください：
- スキーマの自動マッピング
- 大量データの分割処理
- 進捗表示とレジューム機能
- データ整合性チェック

---

**解答例とヒント：**

前章（第7章）の練習問題の解答例：

問題1の解答（カスタムローディング表示）：
```html
<!-- 3種類のローディングアニメーション -->
<style>
.loader-spinner {
    animation: spin 1s linear infinite;
}
.loader-pulse {
    animation: pulse 1.5s ease-in-out infinite;
}
.loader-dots {
    animation: dots 1.5s steps(5, end) infinite;
}

@keyframes spin {
    to { transform: rotate(360deg); }
}
@keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.5; }
}
@keyframes dots {
    0%, 20% { content: '.'; }
    40% { content: '..'; }
    60%, 100% { content: '...'; }
}
</style>

<div hx-get="/api/data"
     hx-indicator=".loader-spinner"
     data-loader-type="spinner">
    <span class="loader-spinner htmx-indicator">⚙️</span>
</div>
```