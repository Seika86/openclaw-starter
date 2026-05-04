# 🌊 OpenClaw Starter — AI秘書/執事/バディを Docker で動かすスターターパック

DiscordやLINEから使える「あなた専用のドラえもん」を、安全な箱（Docker）の中で動かすための設定集です。

## 🎯 これは何？

[OpenClaw](https://github.com/openclaw/openclaw) という OSS の AI エージェント基盤を、**初心者でも 5 分で動かせる**ように設定をまとめたパッケージです。

**特徴:**
- 🛡️ **安全** — バディはこの `workspace/` フォルダだけしか触れない（あなたのデスクトップや書類フォルダは見えない）
- 💾 **データはあなたのPCに残る** — 会話・記憶・設定すべてローカル
- 🚚 **お引越し簡単** — フォルダごと別PCにコピーすれば動く
- 🤖 **モデル自由** — Claude, ChatGPT, Gemini など好きなLLMで動かせる
- 💬 **Discord/LINE 対応** — 普段使うチャットアプリから話しかけられる

---

## 📦 必要なもの

### 必須

1. **Docker Desktop** — 無料、WindowsもMacもOK
   - Windows: https://docs.docker.com/desktop/install/windows-install/
   - Mac: https://docs.docker.com/desktop/install/mac-install/
   - インストール後、**起動して画面に「Engine running」が出るまで待つ**

2. **LLM の API キー（どれか1つ）**
   - **🥇 一番カンタン: Google Gemini** — Googleアカウントだけで取れる、無料
     - https://aistudio.google.com/apikey で「Create API key」
   - 他: Anthropic Claude（有料）、OpenAI（有料）、OpenRouter（無料モデルあり）など

### 推奨

3. **Discord Bot Token** — Discordから話しかけたい場合
   - 取り方は下の「Discord Bot の作り方」セクション参照

---

## 🚀 セットアップ手順（5分）

### 1. このリポジトリをダウンロード

**方法A: git が使える人**
```bash
git clone https://github.com/seika86/openclaw-starter.git
cd openclaw-starter
```

**方法B: zipでダウンロード**
GitHub の画面から「Code → Download ZIP」→ 解凍 → そのフォルダに入る

### 2. APIキー & 認証トークンを設定

`.env.example` を `.env` という名前でコピーして、エディタで開く。

**Windows (PowerShell):**
```powershell
Copy-Item .env.example .env
notepad .env
```

**Mac/Linux:**
```bash
cp .env.example .env
open .env  # またはお好きなエディタで
```

埋めるのは2つ:

**(A) `GEMINI_API_KEY=` の右側に Gemini APIキー**

**(B) `OPENCLAW_GATEWAY_TOKEN=` の右側に Dashboard 認証トークン**

トークンはターミナルで以下を実行して出てきた長い文字列を貼る:

```bash
# Mac/Linux
openssl rand -hex 32

# Windows PowerShell
-join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) })
```

このトークンは後で Dashboard ログイン時にも使うので、`.env` ファイルを閉じずに開いたままにしておくと楽。

### 3. 起動

```bash
docker compose up -d
```

初回はイメージのダウンロードに **3〜5分**かかります（コーヒー☕）。

### 4. LLMプロバイダ登録（初回のみ）

`.env` の API キーを登録するため、以下を1回だけ実行:

```bash
# Gemini を使う場合
docker compose exec buddy openclaw onboard \
  --non-interactive --accept-risk --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"

# デフォルトモデルを Flash に変更（Pro は無料枠で使えないため）
docker compose exec buddy node -e "
const fs=require('fs'),p='/home/node/.openclaw/openclaw.json';
const c=JSON.parse(fs.readFileSync(p));
c.agents=c.agents||{};c.agents.defaults=c.agents.defaults||{};
c.agents.defaults.model={primary:'google/gemini-2.5-flash'};
fs.writeFileSync(p,JSON.stringify(c,null,2));console.log('OK');
"

# 設定反映のため再起動
docker compose restart
```

> Anthropic / OpenAI 等を使う場合は `--auth-choice` と `--gemini-api-key` を該当プロバイダのものに置き換え（公式ドキュメント参照）。

### 5. ブラウザで開く

http://localhost:18789

「OpenClaw Gateway ダッシュボード」が表示されたら、**Gateway トークン** 欄に手順2で生成した値を貼って「**接続**」をクリック。

接続できれば成功！🎉

---

## 💬 Discord Bot の作り方

1. https://discord.com/developers/applications にアクセス
2. 右上の「**New Application**」→ 名前は「My Buddy」とか好きに
3. 左メニュー「**Bot**」→「**Reset Token**」→「**Yes, do it!**」
4. 表示されたトークンをコピー → `.env` の `DISCORD_BOT_TOKEN=` に貼る
   - ⚠️ Token は1回しか表示されないので、必ずコピー！見失ったらまた Reset Token で再生成
5. 同じページの「**Privileged Gateway Intents**」で以下を **全てON**:
   - `PRESENCE INTENT`
   - `SERVER MEMBERS INTENT`
   - `MESSAGE CONTENT INTENT`
6. **🔒 Bot を非公開にする（推奨）** — 他人が勝手に招待できないようにする
   - **必ずこの順序で**（逆だとエラー）:
     1. 左メニュー「**Installation**」 → 「Install Link」を **「None」** に変更 → Save
     2. 左メニュー「**Bot**」 → 「**Public Bot**」を **OFF** → Save
   - エラー `Cannot have install fields on a private application` が出たら順序逆。Installation を先に None にしてから
7. 左メニュー「**OAuth2**」→「**URL Generator**」→
   - SCOPES: `bot` にチェック
   - BOT PERMISSIONS: `Send Messages`, `Read Message History`, `Add Reactions`, `Embed Links`, `Attach Files` にチェック
   - 下に出てきたURLをコピー → ブラウザで開いて自分のサーバーに招待
8. ターミナルで `.env` を保存後、`docker compose restart` で反映

---

## 🛠️ よく使うコマンド

| やりたいこと | コマンド |
|---|---|
| 起動 | `docker compose up -d` |
| 停止 | `docker compose stop` |
| 完全終了 | `docker compose down` |
| ログを見る | `docker compose logs -f` |
| 再起動 | `docker compose restart` |
| 最新版に更新 | `docker compose pull && docker compose up -d` |
| 完全リセット（記憶も消える） | `docker compose down -v` |

---

## 📁 workspace フォルダって何？

このフォルダはバディが**唯一触れる場所**です。

```
あなたのPC
├── 📁 デスクトップ      ← バディは見えない
├── 📁 ドキュメント      ← バディは見えない
├── 📁 写真              ← バディは見えない
└── 📁 openclaw-starter
    └── 📁 workspace     ← ⭐ ここだけバディが見える
```

**使い方の例:**
- Amazonの注文履歴CSVを置いて「持ち物リストにまとめて」
- 領収書PDFを置いて「家計簿に登録して」
- 「今日の日記をここに書いて」

エクスプローラ（Mac は Finder）から普通にドラッグ＆ドロップで置けます。

---

## 💾 お引越し / バックアップ

設定や会話の記憶は Docker の中（named volume）に保存されています。別PCに引っ越したい・バックアップ取りたい時はこちら。

**バックアップを作る:**
```bash
docker run --rm -v openclaw-starter_openclaw_data:/data -v $(pwd):/backup busybox tar czf /backup/buddy-backup.tar.gz /data
```

**復元する:**
```bash
docker run --rm -v openclaw-starter_openclaw_data:/data -v $(pwd):/backup busybox tar xzf /backup/buddy-backup.tar.gz -C /
```

`workspace/` フォルダはホスト側にあるので、フォルダごとコピーすればOK。

---

## ❓ トラブルシューティング

### `docker compose up` で「port is already allocated」
ポート 18789 を別アプリが使ってる。`.env` に以下を追加して再起動:
```
OPENCLAW_GATEWAY_PORT=28789
```
アクセス先も `http://localhost:28789` に変える。

⚠️ **追加で必要な設定**: ポート変更した場合、Dashboard が「origin not allowed」エラーを出します。
以下のコマンドで許可リストに追加してください（`28789` は自分が使うポートに置き換え）:

```bash
docker exec my-buddy node -e "
const fs=require('fs'),p='/home/node/.openclaw/openclaw.json',c=JSON.parse(fs.readFileSync(p));
const u='http://localhost:28789';
if(!c.gateway.controlUi.allowedOrigins.includes(u))c.gateway.controlUi.allowedOrigins.push(u);
fs.writeFileSync(p,JSON.stringify(c,null,2));console.log('OK');
"
docker compose restart
```

### ブラウザで開いても「接続できません」
- Docker Desktop が起動してる？（画面右下のクジラアイコン🐳）
- `docker compose ps` で `(healthy)` になってる？ → ならない場合は `docker compose logs` で確認

### Discord Bot がオフラインのまま
- `DISCORD_BOT_TOKEN` の値、貼り間違えてない？（前後にスペース入ってない？）
- Privileged Intents 全部ON にした？
- `.env` 保存後に `docker compose up -d --force-recreate` した？（`restart` だと env 再読み込みされない罠）
- Discord channel 登録した？（`.env` に Token 置くだけでは登録されない、下記のコマンド要）

```bash
# Discord channel を登録（初回のみ）
docker compose exec buddy openclaw channels add --channel discord --token "$DISCORD_BOT_TOKEN"
docker compose restart
```

### Discord で Bot から「openclaw pairing approve …」と言われた
これは **OpenClaw のペアリング承認** が必要な状態。Bot がメッセージで指示してくる長いコマンドをそのままホストのターミナルで打っても `openclaw: command not found` になる。

**正しい実行方法**: Docker コンテナ内で実行
```bash
docker compose exec buddy openclaw pairing approve discord <ペアリングコード>
```

承認後、Bot が会話に応じるようになる。

### LLM の API キーが効かない
- `.env` のキー値の **前後にスペースが入ってない**か確認
- API キーが有効か、各プロバイダの管理画面で確認
- 無料枠の上限超えてないか確認

---

## 🎓 もっと詳しく / 次の一歩

このリポジトリ内の追加資料:
- 📋 [docs/事前準備チェックリスト.md](./docs/事前準備チェックリスト.md) — 講座1週間前にやること
- 🎤 [docs/当日進行台本.md](./docs/当日進行台本.md) — 講座当日のフロー（講師サイド）
- 🚀 [docs/次の一歩.md](./docs/次の一歩.md) — 講座後にやってみる拡張アイデア
- 🔒 [docs/プライバシー早見表.md](./docs/プライバシー早見表.md) — どのLLMが安心か用途別に判断
- 🏠 [docs/おまけ-ローカルLLM別紙.md](./docs/おまけ-ローカルLLM別紙.md) — 完全オフライン化したい人向け
- 🚚 [docs/PC移行・お引越しガイド.md](./docs/PC移行・お引越しガイド.md) — PC買い替え/サブPC運用時の完全引っ越し手順

公式ドキュメント:
- [OpenClaw 公式ドキュメント](https://docs.openclaw.ai/)
- [対応LLMプロバイダ一覧](https://docs.openclaw.ai/providers/) — Cerebras / Mistral / ローカルLLM など 60+ プロバイダ対応
- [LINE 連携の方法](https://docs.openclaw.ai/channels/line/) — Webhook (HTTPS) が必要
- [サンドボックス機能](https://docs.openclaw.ai/gateway/sandboxing/) — もっと厳しい権限制御

---

## 📜 ライセンス

MIT License — 自由に使ってください。

OpenClaw 本体のライセンスは [openclaw/openclaw](https://github.com/openclaw/openclaw) を参照。

---

🌊 *Created with care by Seika & 海音コウ for ウミネコロン作業日 2026-05-10*
