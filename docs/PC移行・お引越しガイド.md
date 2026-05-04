# 🚚 PC移行・お引越しガイド

> 「PC買い替えるんだけど、バディも連れて行きたい」
> 「サブPC でも同じバディを使いたい」
> そんな時のための完全引っ越しガイド🌊

バディの**記憶・人格・設定すべて**を持っていけます。

---

## 📦 持っていくもの（全部で4種類）

| 何 | 場所 | 役割 |
|---|---|---|
| **🧠 named volume データ** | Docker 内 `<project名>_openclaw_data` | OpenClaw の設定・認証・会話履歴 |
| **📝 workspace フォルダ** | プロジェクト直下 `./workspace/` | バディの人格 (IDENTITY.md / SOUL.md / USER.md / MEMORY.md / memory/) |
| **⚙️ docker-compose.yml** | プロジェクト直下 | コンテナ構成 |
| **🔑 .env** | プロジェクト直下 | API キー・トークン類 |

---

## 🟢 旧PC（送る側）の作業

### 1. コンテナを停止

```bash
cd <openclaw-starter のディレクトリ>
docker compose down  # -v は付けない！データ残す
```

### 2. named volume を tar 化

```bash
# プロジェクト名 = ディレクトリ名（compose のデフォルト挙動）
PROJECT_NAME=$(basename "$PWD")

docker run --rm \
  -v ${PROJECT_NAME}_openclaw_data:/data \
  -v $(pwd):/backup busybox \
  tar czf /backup/openclaw_data_backup.tar.gz -C /data .
```

これで `openclaw_data_backup.tar.gz` が出来る（5〜数百MB）。

### 3. 持っていくものをまとめる

以下4つをUSB/AirDrop/クラウドストレージなどで新PCに送る:

```
openclaw_data_backup.tar.gz   ← Step 2 で作ったやつ
workspace/                    ← フォルダごと
docker-compose.yml
.env
```

⚠️ `.env` は API キー入りなので **クラウドストレージに置く時は要注意**（プライベートな共有方法で）

---

## 🔵 新PC（受ける側）の作業

### 4. Docker Desktop インストール
新PCに Docker Desktop が入ってなければ先にインストール（README.md 参照）。

### 5. 受け取ったもの一式を任意のフォルダに配置

```
~/your-buddy/   ← ここに置く（フォルダ名は自由、これがプロジェクト名になる）
├── openclaw_data_backup.tar.gz
├── workspace/
├── docker-compose.yml
└── .env
```

⚠️ **プロジェクト名（フォルダ名）は旧PCと同じにしておくのが楽**。違うとnamed volume名も変わるので、Step 7 で名前指定が要る。

### 6. コンテナ作成（起動はまだしない）

```bash
cd ~/your-buddy/
docker compose create
```

これで named volume `<フォルダ名>_openclaw_data` が新規作成される。

### 7. データ復元

```bash
PROJECT_NAME=$(basename "$PWD")
docker run --rm \
  -v ${PROJECT_NAME}_openclaw_data:/data \
  -v $(pwd):/backup busybox \
  tar xzf /backup/openclaw_data_backup.tar.gz -C /data
```

### 8. 起動

```bash
docker compose up -d
docker compose logs --tail 20
```

ログに `[gateway] ready` と `[discord] [default] starting provider (...)` が出れば成功🎉

### 9. 動作確認

- ブラウザで `http://localhost:18789`（or `.env` で指定したポート）開く
- Dashboard ログイン（Gateway トークン同じやつ）
- Discord で Bot にメンション → 旧PC同様にバディが応答すれば完了

---

## 🆘 よくあるハマり

### `Conflict. The container name "..." is already in use`
旧PCで動いてた同名コンテナが残ってる。新PC上で衝突なら、`.env` の `OPENCLAW_GATEWAY_PORT` を変えるか、`docker-compose.yml` の `container_name` を別名に変更。

### `volume not found` エラーで復元できない
プロジェクト名（フォルダ名）が旧PCと違う、または Step 6 を飛ばした。`docker volume ls` で実際のvolume名を確認 → そっちに展開。

### Bot がオフラインのまま
- `.env` の `DISCORD_BOT_TOKEN` がそのまま新PCに移行されてるか確認
- `docker compose up -d --force-recreate` で env 再読み込み（restart だと反映されない罠）

### Token 認証が通らない
`.env` の改行コードが macOS↔Windows で壊れたかも。テキストエディタで開いて改行 LF（Unix形式）に再保存。

### .env を送るのが怖い
**API キーは新PCで再取得** する手もある。Token系は無効化→新規発行で同じこと。
- OpenAI / Gemini / Anthropic: 旧キーをrevoke → 新キー発行
- Discord Bot Token: Reset Token で再発行（同じBotで継続OK）

---

## 💾 バックアップだけしたい時（移行ではなく）

旧PCの Step 1〜2 だけやって、`openclaw_data_backup.tar.gz` と `workspace/` を保存しておけば後で復元できます。

定期バックアップしたい場合は cron 化:
```bash
# 例: 毎日23:00にバックアップ
0 23 * * * cd ~/openclaw-starter && docker compose down && docker run --rm -v ${PWD##*/}_openclaw_data:/data -v ${PWD}:/backup busybox tar czf /backup/buddy-$(date +%Y%m%d).tar.gz -C /data . && docker compose up -d
```

---

## 🌊 完全リセットしたい時

「全部やり直したい」「バディを生まれ直しさせたい」場合:

```bash
docker compose down -v  # -v で named volume も削除
rm -rf workspace/*       # 中身全削除（フォルダは残す）
docker compose up -d     # 新生バディで起動
```

⚠️ **これやったら記憶も人格も全部消えます**。最後に backup 取ってからやって！

---

🌊 *Seika & 海音コウ — 2026-05-04 検証済み*
