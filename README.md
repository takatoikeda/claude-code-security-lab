# claude-code-security-lab
Local security testing environment using Claude Code, WSL2, Docker and WordPress.
# WSL2 + Docker + WordPress + Claude Code によるローカルセキュリティ検証環境構築

## 概要

Windows 11 上に WSL2(Ubuntu) を構築し、Docker 上で WordPress + MariaDB + nginx(HTTPS) 環境を作成した。

その後、Windows 側に Claude Code を導入し、ローカル WordPress 環境に対して初回のセキュリティ調査を実施した。

構成は以下。

```text
Windows 11
├─ Claude Code (Windows)
└─ WSL2 Ubuntu
   └─ Docker
      ├─ nginx (Reverse Proxy / TLS Termination)
      ├─ WordPress
      └─ MariaDB
```

---

## 1. WSL2 / Docker 環境構築

WSL Ubuntu 上で Docker Engine / Compose Plugin を導入。

インストール後、動作確認。

```bash
docker run hello-world
```

正常応答。

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

Docker ネットワーク確認。

```bash
ip a
```

確認内容。

- WSL eth0: `172.31.x.x`
- Docker bridge: `172.17.0.1/16`

Docker daemon 正常稼働を確認。:contentReference[oaicite:0]{index=0}

---

## 2. WordPress + MariaDB Docker Compose 環境作成

作業ディレクトリ。

```bash
~/docker-lab/wordpress
```

Compose 構成。

- `wordpress:latest`
- `mariadb:10.6`
- `nginx:latest`

初回起動。

```bash
docker compose up -d
```

正常に以下コンテナ作成。

```text
wp-db
wp-web
wp-nginx
```

---

## 3. HTTPS(Self-Signed Certificate) 対応

OpenSSL を使用し自己証明書を生成。

実行コマンド。

```bash
openssl req -x509 -nodes -days 365 \
-newkey rsa:2048 \
-keyout server.key \
-out server.crt
```

証明書設定例。

| 項目 | 値 |
|------|------|
| Country | JP |
| State | Tokyo |
| Locality | Chiyodaku |
| Organization | Tabetaro |
| CN | localhost |

生成物。

```text
server.crt
server.key
```

nginx を HTTPS Reverse Proxy として構成。

通信フロー。

```text
Browser
↓ HTTPS:443
nginx
↓ HTTP:80
WordPress(Apache/PHP)
↓
MariaDB
```

---

## 4. 構築時のトラブルシュート

### 4-1. YAML Syntax Error

compose.yml 編集時に YAML 構文エラー発生。

エラー。

```text
yaml: could not find expected ':'
```

原因。

- compose.yml のインデント / キー記述不整合

対応。

- compose.yml 修正
- 再起動

---

### 4-2. Docker Compose Variable Expansion 問題

WordPress HTTPS 対応のため PHP 設定追加。

記述。

```php
$_SERVER['HTTPS']='on';
```

Compose 起動時エラー。

```text
WARN[0000] The "_SERVER" variable is not set.
```

原因。

Docker Compose が `$` を環境変数展開対象として解釈。

対応。

```php
$$_SERVER['HTTPS']='on';
```

へ変更。

---

### 4-3. Database Connection Error

WordPress 初回起動時。

```text
Error establishing a database connection
```

確認内容。

```bash
docker logs wp-db
```

MariaDB ログ確認。

正常に以下を確認。

```text
Creating database wordpress
Creating user wpuser
Giving user wpuser access to schema wordpress
MariaDB init process done
ready for connections
```

対処。

```bash
docker compose down -v
docker compose up -d
docker compose restart wordpress
```

DB 初期化後、正常起動。

MariaDB 初期化ログも正常。:contentReference[oaicite:1]{index=1}

---

## 5. Claude Code 導入 (Windows)

PowerShell 実行。

```powershell
irm https://claude.ai/install.ps1 | iex
```

インストール成功。

```text
Claude Code successfully installed
Version: 2.1.150
```

初回問題。

```text
claude : The term 'claude' is not recognized
```

原因。

PATH 未登録。

対応。

`C:\Users\<user>\.local\bin`

を PATH に追加。

再起動後。

```powershell
claude
```

正常起動。

---

## 6. Claude Code による初回セキュリティ調査

調査対象。

```text
https://localhost
```

実行プロンプト。

```text
Analyze https://localhost as my authorized local security lab.
Ignore self-signed certificate warnings.
Tell me what you would check first.
```

Claude Code は並列で複数 shell command を実行しながら初回調査を実施。

実施内容。

- HTTP Header Inspection
- TLS Inspection
- WordPress Fingerprinting
- REST API Enumeration
- XML-RPC Detection
- Security Header Check

---

## 7. 初回調査 Findings

### 7-1. Stack Fingerprinting

検出結果。

| Component | Version | Notes |
|------|------|------|
| Reverse Proxy | nginx 1.31.1 | Frontend |
| Backend | Apache 2.4.67 | WordPress Container |
| Language | PHP 8.3.31 | Header Leakage |
| CMS | WordPress 7.0 | Meta Generator |
| Theme | twentytwentyfive | Default Theme |

---

### 7-2. REST API User Enumeration

検出。

```http
GET /?rest_route=/wp/v2/users
```

結果。

```text
ID:1
name:test1
slug:test1
```

匿名ユーザーから WordPress Username 列挙可能。

---

### 7-3. XML-RPC Enabled

確認。

```text
/xmlrpc.php
```

利用可能メソッド。

```text
system.listMethods
system.multicall
```

確認。

潜在リスク。

- Authentication brute force amplification
- XML-RPC abuse
- upload API exposure

---

### 7-4. Missing Security Headers

未設定確認。

```text
Strict-Transport-Security
X-Content-Type-Options
```

また、以下ヘッダ露出。

```text
X-Powered-By: PHP/8.3.31
```

---

### 7-5. Reverse Proxy Backend Leakage

404 系レスポンス経由で Apache backend 情報露出。

検出対象例。

```text
robots.txt
.env
wp-content/debug.log
```

結果。

```text
Apache/2.4.67 (Debian)
```

Reverse Proxy 層で完全にサニタイズされていないことを確認。

---

## 8. 所感

Claude Code は通常 CLI というより、半自律的な Agent 型インターフェースに近い。

特徴。

- shell command 自動提案
- 並列実行
- 実行許可確認
- findings 要約
- 攻撃パス提案

初学習用途としては、

- Docker
- Reverse Proxy
- HTTPS
- WordPress Hardening
- Web Enumeration
- AI Agent Workflow

を同時に確認できる環境になった。
