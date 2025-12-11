# TunnelHub

**セルフホスト型SSHリバーストンネルサーバー**

[![License: MIT](https://img.shields.io/badge/License-MIT-ff6b6b.svg?style=for-the-badge)](LICENSE)
[![Debian](https://img.shields.io/badge/Debian-11%20%7C%2012-d70a53?style=for-the-badge&logo=debian&logoColor=white)](https://www.debian.org/)
[![SSH](https://img.shields.io/badge/SSH-Tunnel-2d3748?style=for-the-badge)](https://www.openssh.com/)

**ローカル開発環境を一瞬でインターネットに公開**

---

## 概要

TunnelHubは、Debian VPS上に構築する軽量なSSHリバーストンネルサーバーです。ローカルPCで動作している開発サーバーを、SSH一本で世界中からアクセス可能にします。

```
┌─────────────────────────────┐
│  ローカルPC                  │
│  (Mac/Linux/Windows)        │
│  localhost:8080             │
└──────────────┬──────────────┘
               │
               │  ssh -R
               ▼
┌─────────────────────────────┐
│  Debian VPS                 │
│  Public Port (自動割当)      │
└──────────────┬──────────────┘
               │
               ▼
          Webブラウザ
```

## 機能

* 超シンプル - OpenSSHだけで動作、複雑な設定不要
* 5分でセットアップ - すぐに本番環境が完成
* 自動ポート割当 - ポート管理の手間なし
* ドメイン不要 - IP:PORTで即座にアクセス
* 超軽量 - DockerもDBも不要
* セキュア - OpenSSHの実績ある暗号化
* クロスプラットフォーム - Mac、Linux、Windows (WSL) 対応

## VPS環境

* OS: Debian 11 / 12
* 権限: root権限またはsudo権限必須
* ネットワーク: パブリックIPアドレス

## セットアップ手順

### 1. パッケージのインストール（VPS側）

```bash
sudo apt update
sudo apt install -y openssh-server ufw curl net-tools
```

### 2. tunnelユーザーの作成（VPS側）

```bash
sudo adduser tunnel
```

パスワードは任意で設定してください。

### 3. SSHサーバー設定（VPS側）

設定ファイルを編集：

```bash
sudo nano /etc/ssh/sshd_config
```

以下の行を探すか、無ければ追記してください：

```ssh-config
AllowTcpForwarding yes
GatewayPorts clientspecified
PermitOpen any
PasswordAuthentication yes
```

ファイルの一番下に追加：

```ssh-config
Match User tunnel
    AllowTcpForwarding yes
    GatewayPorts yes
    X11Forwarding no
    PermitTunnel no
```

保存して反映：

```bash
sudo systemctl restart ssh
```

### 4. ファイアウォール設定（VPS側）

```bash
sudo ufw allow 22/tcp
sudo ufw allow 20000:65000/tcp
sudo ufw enable
sudo ufw reload
```

確認：

```bash
sudo ufw status
```

### 5. ローカルPCの準備（Mac/Linux）

フォルダ作成：

```bash
mkdir ~/tunnel-test
cd ~/tunnel-test
```

テスト用HTMLを作成：

```bash
nano index.html
```

中身：

```html
<h1>SSH Tunnel Works!</h1>
```

### 6. ローカルWebサーバー起動（ローカルPC）

```bash
cd ~/tunnel-test
python3 -m http.server 8080
```

この状態：

```
Serving HTTP on :: port 8080 ...
```

このターミナルは閉じないでください。

### 7. SSHリバーストンネル接続（ローカルPC）

別のターミナルを開き：

```bash
ssh -R 0.0.0.0:0:localhost:8080 tunnel@VPSのIP
```

成功すると：

```
Allocated port 83493 for remote forward to localhost:8080
```

このポート番号が公開URLのポートです。

### 8. ブラウザからアクセス

以下のURLを開きます：

```
http://VPSのIP:割り当てられたポート
```

完成です。あなたのローカル環境がインターネットに公開されました。

## 使い方

### 基本コマンド

```bash
ssh -R 0.0.0.0:0:localhost:ローカルポート tunnel@VPSのIP
```

### よく使うフレームワーク

| フレームワーク | デフォルトポート | コマンド例 |
|--------------|----------------|-----------|
| Next.js | 3000 | `ssh -R 0.0.0.0:0:localhost:3000 tunnel@VPSのIP` |
| React | 3000 | `ssh -R 0.0.0.0:0:localhost:3000 tunnel@VPSのIP` |
| Vue.js | 8080 | `ssh -R 0.0.0.0:0:localhost:8080 tunnel@VPSのIP` |
| Django | 8000 | `ssh -R 0.0.0.0:0:localhost:8000 tunnel@VPSのIP` |
| Flask | 5000 | `ssh -R 0.0.0.0:0:localhost:5000 tunnel@VPSのIP` |
| Express | 3000 | `ssh -R 0.0.0.0:0:localhost:3000 tunnel@VPSのIP` |

### 便利なオプション

接続を維持する：

```bash
ssh -R 0.0.0.0:0:localhost:8080 \
    -o ServerAliveInterval=30 \
    -o ServerAliveCountMax=3 \
    tunnel@VPSのIP
```

バックグラウンドで実行：

```bash
ssh -fN -R 0.0.0.0:0:localhost:8080 tunnel@VPSのIP
```

特定のポートを指定：

```bash
ssh -R 0.0.0.0:8080:localhost:3000 tunnel@VPSのIP
```

## 動作確認用コマンド

### VPS側

```bash
ss -ltnp | grep ポート番号
```

理想の出力：

```
LISTEN 0 128 0.0.0.0:ポート番号
```

### ローカルPC側

```bash
lsof -i :8080
```

## トラブルシューティング

### よくあるトラブル

| 問題 | 対処 |
|------|------|
| ポートに接続できない | ufwとVPS管理画面のファイアウォールを確認 |
| 127.0.0.1:PORTにバインドされる | /etc/ssh/sshd_configのGatewayPorts clientspecifiedを確認 |

### 詳細な診断

VPS側で確認：

```bash
# SSHサービスの状態
sudo systemctl status ssh

# アクティブな接続
sudo netstat -tulpn | grep sshd

# SSHログを確認
sudo tail -f /var/log/auth.log

# ファイアウォール状態
sudo ufw status verbose
```

ローカルPC側で確認：

```bash
# 詳細なSSH接続ログ
ssh -v -R 0.0.0.0:0:localhost:8080 tunnel@VPSのIP

# ローカルサービスが起動しているか
lsof -i :8080
curl localhost:8080
```

VPSプロバイダーのファイアウォール確認：

* AWS: Security Groupsをチェック
* Google Cloud: Firewall Rulesをチェック
* DigitalOcean: Cloud Firewallをチェック
* Vultr: Firewall settingsをチェック

### SSH接続が切れる場合

~/.ssh/configに設定を追加：

```ssh-config
Host tunnelhub
    HostName VPSのIP
    User tunnel
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

接続：

```bash
ssh -R 0.0.0.0:0:localhost:8080 tunnelhub
```

## セキュリティ注意

デフォルト設定はテスト用です。本番運用時は必ず以下を実施してください。

### 本番運用時の推奨設定

#### 1. パスワード認証から鍵認証に変更

ローカルPCで鍵を生成：

```bash
# SSH鍵を生成（まだ持っていない場合）
ssh-keygen -t ed25519 -C "your_email@example.com"

# 公開鍵をVPSにコピー
ssh-copy-id tunnel@VPSのIP
```

VPS側でパスワード認証を無効化：

```bash
sudo nano /etc/ssh/sshd_config
```

```ssh-config
PasswordAuthentication no
```

```bash
sudo systemctl restart ssh
```

#### 2. tunnelユーザーのログイン制限

```bash
# シェルアクセスを無効化
sudo usermod -s /usr/sbin/nologin tunnel

# または制限付きシェルを使用
sudo usermod -s /bin/rbash tunnel
```

#### 3. ufwによるポート制限

```bash
# より狭い範囲に制限
sudo ufw delete allow 20000:65000/tcp
sudo ufw allow 30000:40000/tcp
sudo ufw reload
```

#### 4. fail2banでブルートフォース対策

```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

#### 5. アクティブなトンネルを監視

```bash
# リバーストンネルの確認
ss -ltnp | grep LISTEN

# tunnelユーザーの接続を確認
sudo ps aux | grep tunnel
```

### セキュリティチェックリスト

* SSH鍵認証を使用する
* tunnelユーザーのシェルアクセスを無効化
* ファイアウォールでポート範囲を制限
* fail2banを導入
* SSHサーバーを常に最新に保つ
* 接続ログを定期的に確認
* 弱いパスワードを使わない
* tunnelユーザーの認証情報を共有しない
* 機密性の高いサービスを公開しない

## 将来拡張

この構成は以下に拡張可能です：

### 拡張アイデア

* HTTPS対応 - Let's EncryptでSSL/TLS証明書を取得
* 自動サブドメイン発行 - abc123.yourdomain.com形式の自動割当
* Docker化 - コンテナベースのデプロイ
* Web管理画面 - トンネルをGUIで管理
* カスタムドメイン - 独自ドメインの持ち込み
* 認証API - トークンベースのトンネル作成
* 帯域制限 - ユーザーごとのトラフィック制限
* 接続ログ - トンネル使用状況の追跡

### HTTPS化の例（Nginx + Let's Encrypt）

```bash
# NginxとCertbotをインストール
sudo apt install -y nginx certbot python3-certbot-nginx

# Nginxをreverse proxyとして設定
sudo nano /etc/nginx/sites-available/tunnelhub
```

```nginx
server {
    listen 80;
    server_name tunnel.yourdomain.com;
    
    location / {
        proxy_pass http://localhost:トンネルポート;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# サイトを有効化してSSL証明書を取得
sudo ln -s /etc/nginx/sites-available/tunnelhub /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo certbot --nginx -d tunnel.yourdomain.com
```

## 参考資料

* OpenSSH公式ドキュメント: https://www.openssh.com/manual.html
* SSHリバーストンネリングガイド: https://www.ssh.com/academy/ssh/tunneling/example
* UFWファイアウォールチュートリアル: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian
* Let's Encrypt公式サイト: https://letsencrypt.org/

## 開発支援

このツールが役立った場合、継続的な開発を支援することをご検討ください:

**Bitcoin (BTC):**
```
151feG2x2pUqG97p9kSKL7E3LgpukNWozT
```

すべての寄付は、無料で利用可能な開発ツールの維持と改善に役立ちます。

---

## 教育理念

本プロジェクトは、すべてのユーザーが無料で利用できる高品質なツールの提供に取り組んでいます。すべての機能は無料で提供され、今後も無料で利用可能です。

---

## 免責事項

本ソフトウェアは教育目的で提供されています。ユーザーは、使用が適用される法律および規制に準拠していることを確保する責任を負います。

---

*責任を持って使用してください*
