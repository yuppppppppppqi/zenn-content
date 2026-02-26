---
title: "認証なしで公開APIを出したら12時間でBotに3000リンク生成された話"
emoji: "🛡️"
type: "tech"
topics: ["セキュリティ", "個人開発", "Python", "API", "Bot対策"]
published: true
---

## 何が起きたか

URL短縮サービスを個人開発して公開した。リンク作成APIに認証をかけていなかった。

公開から12時間後、管理画面を開いたらリンクが3,217件に増えていた。すべてアダルトサイトへの短縮リンク。スパムBotの仕業だった。

この記事では、事故の原因分析と、その後に実装した防御パターンを具体的なコード付きで書く。同じように公開APIを持つ個人開発者の参考になれば。

## 事故の原因

### 認証ゼロのエンドポイント

```python
# 事故当時のコード（やってはいけない例）
@app.route('/shorten', methods=['POST'])
def shorten():
    url = request.json['url']
    slug = generate_slug()
    db.execute("INSERT INTO links (slug, url) VALUES (?, ?)", (slug, url))
    return jsonify({"short_url": f"https://example.com/{slug}"})
```

このエンドポイントは認証なし、レートリミットなし、バリデーションなし。`POST` でJSONを投げれば誰でもリンクを作れる状態だった。

### なぜBotに見つかったか

公開APIがBotに発見される経路は主に3つ。

1. **ポートスキャン** — 既知のポート（80, 443）を巡回して、短縮URLっぽいレスポンスを返すサーバーを特定
2. **SNSの投稿** — Twitterに「URL短縮サービス作りました」と書いた。このツイートからドメインを拾われた可能性が高い
3. **Certificate Transparency Logs** — SSL証明書の発行記録は公開されている。新しいドメインの証明書発行をBotが監視している

自分の場合、2が一番怪しい。公開した夜にツイートして、翌朝にはもうやられていた。

## 実装した防御パターン

事故後に実装した対策を、レイヤーごとに書く。

### Layer 1: API認証

最初にやるべきこと。APIキーによるBearer認証。

```python
@app.before_request
def check_auth():
    if request.path.startswith('/api/'):
        auth = request.headers.get('Authorization', '')
        if not auth.startswith('Bearer '):
            return jsonify({"error": "Missing API key"}), 401

        api_key = auth.replace('Bearer ', '')
        user = db.execute(
            "SELECT * FROM users WHERE api_key = ?", (api_key,)
        ).fetchone()

        if not user:
            return jsonify({"error": "Invalid API key"}), 401

        g.user = user
```

**APIキーの生成に注意。** 最初は8文字のランダム文字列で生成していた。これだとブルートフォースで突破される。最低32文字、`secrets.token_urlsafe(32)` で生成すべき。

```python
import secrets

def generate_api_key():
    return secrets.token_urlsafe(32)  # 43文字のURL-safeな文字列
```

### Layer 2: レートリミット

認証だけでは足りない。正規のAPIキーが漏洩した場合や、アカウント登録を自動化されるケースがある。

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    storage_uri="redis://localhost:6379",
)

# グローバルリミット
@app.route('/api/links', methods=['POST'])
@limiter.limit("10/minute")          # IPあたり10件/分
@limiter.limit("100/hour")           # IPあたり100件/時
@limiter.limit("500/day", key_func=lambda: g.user['id'])  # ユーザーあたり500件/日
def create_link():
    ...
```

レートリミットは多段階にする。IP単位とユーザー単位の両方が必要。IPだけだとプロキシで回避され、ユーザーだけだと複垢で回避される。

### Layer 3: URLバリデーション

受け取るURLの基本的な検証。

```python
from urllib.parse import urlparse

def validate_url(url):
    try:
        parsed = urlparse(url)
    except Exception:
        return False, "Invalid URL format"

    # スキームチェック
    if parsed.scheme not in ('http', 'https'):
        return False, "Only http/https URLs are allowed"

    # ホスト名チェック
    if not parsed.hostname:
        return False, "Missing hostname"

    # プライベートIPの排除（SSRF対策）
    import ipaddress
    try:
        ip = ipaddress.ip_address(parsed.hostname)
        if ip.is_private or ip.is_loopback:
            return False, "Private/loopback IPs not allowed"
    except ValueError:
        pass  # ホスト名がIPでない場合は無視

    return True, None
```

### Layer 4: ドメインブラックリスト（と、その限界）

最初にやりがちなのがドメインブラックリスト。

```python
BLOCKED_DOMAINS = {'adult-site.com', 'spam-domain.net', ...}

def check_domain(url):
    hostname = urlparse(url).hostname
    return hostname not in BLOCKED_DOMAINS
```

**これは気休めにしかならない。** 事故2日目に学んだ。Botは以下の方法でブラックリストを回避する。

1. **2段階リダイレクト** — 短縮先を `innocent-blog.com/redirect?to=adult-site.com` のようにする
2. **新規ドメインの使い捨て** — 安いドメインを大量取得して使い回す
3. **正規サービスの悪用** — Google DriveやDropboxのURLに仕込む

ブラックリストは最低限の防御として残しつつ、他のレイヤーと組み合わせる必要がある。

### Layer 5: リンク先のコンテンツスキャン

2段階リダイレクトに対応するため、リンク先を実際にクロールして中身を判定する。

```python
import httpx

async def scan_url(url, max_redirects=5):
    """リダイレクトチェーンを追跡して最終URLを取得"""
    visited = []

    async with httpx.AsyncClient(follow_redirects=False) as client:
        current_url = url
        for _ in range(max_redirects):
            visited.append(current_url)
            try:
                res = await client.head(current_url, timeout=5.0)
                if res.status_code in (301, 302, 307, 308):
                    current_url = res.headers['location']
                else:
                    break
            except Exception:
                break

    # 最終URLのドメインをブラックリストと照合
    final_host = urlparse(current_url).hostname
    is_blocked = final_host in BLOCKED_DOMAINS

    return {
        "redirect_chain": visited,
        "final_url": current_url,
        "is_blocked": is_blocked,
    }
```

ただし、JavaScriptによるクライアントサイドリダイレクトは `HEAD` リクエストでは検出できない。完全にやるならヘッドレスブラウザが必要だが、個人開発でそこまでやるとインフラコストが跳ね上がる。

### Layer 6: Bot検知パターン

最終的に一番効果があったのは、リクエストのパターン分析。

```python
def detect_bot(request):
    signals = []

    # 1. User-Agentチェック
    ua = request.headers.get('User-Agent', '')
    if not ua or ua in KNOWN_BOT_UAS:
        signals.append('suspicious_ua')

    # 2. リクエスト間隔チェック
    last_request = redis.get(f"last_req:{request.remote_addr}")
    if last_request:
        interval = time.time() - float(last_request)
        if interval < 0.5:  # 0.5秒未満の連続リクエスト
            signals.append('rapid_fire')

    # 3. 同一IPからの大量作成
    hourly_count = redis.incr(f"hourly:{request.remote_addr}")
    redis.expire(f"hourly:{request.remote_addr}", 3600)
    if hourly_count > 50:
        signals.append('high_volume')

    # 4. URLパターンの類似性
    # Botは似たようなURLを大量に作る傾向がある
    recent_urls = redis.lrange(f"urls:{request.remote_addr}", 0, 20)
    if len(recent_urls) > 5:
        domains = [urlparse(u).hostname for u in recent_urls]
        if len(set(domains)) <= 2:  # 直近20件中2ドメイン以下
            signals.append('repetitive_pattern')

    return len(signals) >= 2  # 2つ以上のシグナルでBot判定
```

単一の指標で判定するのではなく、複数のシグナルを組み合わせてスコアリングする。これが一番誤検知が少なかった。

## 防御の優先順位

個人開発で全部を最初からやるのは現実的ではない。優先順位をつけるなら：

| 優先度 | 対策 | 実装コスト | 効果 |
|--------|------|-----------|------|
| **必須** | API認証 | 低 | 高 |
| **必須** | レートリミット | 低 | 高 |
| **推奨** | URLバリデーション | 低 | 中 |
| **推奨** | Bot検知 | 中 | 高 |
| **任意** | ドメインブラックリスト | 低 | 低 |
| **任意** | コンテンツスキャン | 高 | 中 |

上2つ（認証 + レートリミット）は **公開前に** 実装すべき。自分はこの2つを「後で」にした結果、3,000本のアダルトリンクを生成された。

## まとめ

公開APIのセキュリティは「やられてから対応」では遅い。Botは公開から12時間以内に来る。

最低限やること：
1. APIキー認証（32文字以上の `secrets.token_urlsafe`）
2. 多段階レートリミット（IP単位 + ユーザー単位）

余裕があれば：
3. URLバリデーション + SSRF対策
4. 複合シグナルによるBot検知

ドメインブラックリストだけに頼るのは危険。2段階リダイレクトで簡単にすり抜けられる。

これらの対策を実装して作り直したのが [Picoli](https://picoli.site/z-picoli-0002a) というURL短縮サービス。同じ失敗をする人が減れば幸い。

https://picoli.site/z-picoli-0002b
