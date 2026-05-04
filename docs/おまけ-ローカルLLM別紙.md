# 🏠 おまけ：自分のPCでLLMを動かす（完全オフライン化）

> 「APIキー要らない、ネット切れても動く、データは絶対外に出ない」
> ただし良いGPU（NVIDIA 8GB+）または Mac Apple Silicon が必要。

## 🚨 まず警告

**「Qwen 3.6 が強いって聞いたから `ollama pull qwen3.6` する！」 → 現状動きません！**

Qwen 3.6 はマルチモーダル対応でファイル構造が特殊（vision用 mmproj が別ファイル）なため、
2026年5月時点で **Ollama 公式は対応中**。
使いたい場合は **llama.cpp** か **Unsloth Studio** 経由で動かしてください。

ステータス: https://github.com/ollama/ollama/issues/ で「qwen3.6」検索

---

## 3つのツールの選択肢

| ツール | OS対応 | UI | GPU要件 | 講座おすすめ度 |
|---|---|---|---|---|
| **🥇 LM Studio** | Win/Mac/Linux | GUI完結 | 無くてもCPU可 | ⭐⭐⭐ ノーコード派 |
| **🥈 Ollama** | Win/Mac/Linux | CLI + WebUI別 | NVIDIA 6GB+ 推奨 | ⭐⭐ 開発寄り |
| **🥉 llama.cpp** | 全部 | CLI | 何でも動く | ⭐ 玄人向け |

---

## 推奨モデル早見表（2026年5月版）

| モデル | サイズ | 必要VRAM | 日本語 | エージェント | Ollama対応 | 用途 |
|---|---|---|---|---|---|---|
| **Gemma 4 E4B** ⭐推し | 4.5B | 6GB | ◎(140言語) | ◎ native | ✅ 即対応 | お試し・日常 |
| **Gemma 4 26B MoE** | 26B(active 3.8B) | 8GB | ◎ | ◎ | ✅ 即対応 | 8GB GPU で本格運用 |
| **Gemma 4 31B** | 31B | 24GB | ◎ | ◎ | ✅ | RTX 3090/4090 持ち |
| **Qwen 3.6 27B** | 27B | 17GB | ◎ | ⭐⭐⭐ 最強 | ❌ llama.cpp 必須 | エージェント本気 |
| **Qwen 3.6 35B-A3B** | 35B(active 3B) | 22GB | ◎ | ⭐⭐⭐ | ❌ llama.cpp | 推論ガチ勢 |
| **Llama 3.2 3B** | 3B | 4GB | ○ | ○ | ✅ | 軽量・気軽 |
| **Phi-4 14B** | 14B | 12GB | ○ | ○ | ✅ | 賢さ重視 |

---

## 💡 用途別オススメ早見表

| やりたいこと | 推奨モデル | 理由 |
|---|---|---|
| **とりあえず試す** | Gemma 4 E4B | 6GB VRAM、日本語◎、Ollamaで即動く |
| **8GB GPU 持ち** | Gemma 4 26B MoE | 26B 品質を 8GB に圧縮 |
| **エージェント本気運用** | Qwen 3.6 27B | tool calling 最強、ただし llama.cpp |
| **音声・動画も扱いたい** | Gemma 4 E2B/E4B | audio/video ネイティブ唯一 |
| **超低スペックPC** | Gemma 4 E2B | ~3GB、ノートPCでも動く |
| **多言語混在で使う** | Gemma 4 | 140言語ネイティブ |

---

## OpenClaw との接続

ローカルLLMを起動した状態で、OpenClaw の Dashboard (`http://localhost:18789`) から
Provider 設定で接続先を指定します。

### LM Studio の場合
- LM Studio で「Local Server」を起動 → ポート 1234
- 重要: 起動時に **`lms server start --bind 0.0.0.0`** で外部公開
- OpenClaw の API URL: `http://host.docker.internal:1234/v1`

### Ollama の場合
- `ollama serve` でサービス起動 → ポート 11434
- OpenClaw の API URL: `http://host.docker.internal:11434`
- ⚠️ **`/v1` を付けないこと！** OpenAI互換URLだと tool calling が壊れます（公式警告）
- ネイティブ Ollama API のままで OK

---

## ⚠️ よくあるハマりポイント

### WSL2 で `/mnt/c/` にモデル置いたら遅い
WSL2 のネイティブファイルシステム（`/home/user/...`）に置けば 3〜5倍速くなる。
Ollama は `~/.ollama/models/` にモデル保存される。

### GPU が認識されない
Windows 側の **NVIDIAドライバを最新化** すれば WSL2 内で自動認識される。
WSL2 内に CUDA を別途インストールする必要は **無い**（公式非推奨）。

### メモリ不足で落ちる
量子化版（**Q4_K_M**）を使う。同じモデル名でも `:Q4_K_M` 等タグで指定可。
例: `ollama pull gemma4:e4b-q4_K_M`

---

## 🔬 実測データ（2026-05-04 Seika & 海音コウ検証）

### Mac mini M4 32GB Unified Memory

| モデル | 状態 | 初回応答 | 賢さ感 | メモ |
|---|---|---|---|---|
| **Gemma 4 E4B (8B Q4)** | ⭕ 快適 | **6秒** | ⭐⭐⭐ | 🥇 普段使い最適 |
| **Qwen 3.5 9B (Q4)** | ⭕ 動く | **15秒** | ⭐⭐⭐⭐ | thinking mode で遅め、賢さは上 |
| **Gemma 4 26B MoE (Q4)** | ❌ 詰む | timeout | - | 256K context で 39.8GB 要求 → メモリ溢れ |

### 重要な発見

#### 1. context length のデフォルトが大きすぎる罠
Gemma 4 26B のデフォは **context_length: 262144 (256K)** で、これが KV cache を巨大化させて Mac mini M4 32GB でも溢れる。
**対策**: Modelfile で `PARAMETER num_ctx 8192` 指定したカスタムモデルを `ollama create` する。

```Modelfile
FROM gemma4:26b
PARAMETER num_ctx 8192
PARAMETER num_predict 1024
```

```bash
ollama create gemma4:26b-ctx8k -f Modelfile
```

これで KV cache が 1/30 になって M4 32GB に収まる可能性。

#### 2. OpenClaw との組み合わせは context削減が必須
OpenClaw は workspace files で 35K+ tokens 注入する仕様（Issue #9157）。
ローカルLLMだと prompt eval が遅くなり、応答待ちが伸びる。

**対策**: openclaw.json で
```json5
{ agents: { defaults: { bootstrapMaxChars: 4000, bootstrapTotalMaxChars: 15000 } } }
```

#### 3. Qwen 系は thinking mode で遅い
Qwen 3.5 / Qwen 3 系は default で `thinking` を生成する。OpenAI互換 API (`enable_thinking: false`) でも Ollama 経由では効かないクセあり。
**回避策**: User message 冒頭に `/no_think` を入れる、または Modelfile で system prompt 制御。

### モデルサイズ別 環境適合表（実測ベース）

| モデル | Mac mini M4 32GB | WSL2 RTX 2080S 8GB |
|---|---|---|
| Gemma 4 E2B (~3GB) | ⭕ 余裕 | ⭕ 余裕 |
| Llama 3.2 3B | ⭕ 余裕 | ⭕ 余裕 |
| **Qwen 3.5 9B (~6.6GB)** | ⭕ 余裕 | ⭕ 乗る |
| **Gemma 4 E4B (~9GB)** | ⭕ 動く | ⚠️ ギリ |
| Phi-4 14B (~9GB) | ⭕ | ❌ CPU offload必須 |
| **Gemma 4 26B MoE (~17GB+)** | ⚠️ context削減必須 | ❌ 無理 |

### 結論

- **「とりあえず無料で賢いの欲しい」: Gemma 4 E4B が現実解**（M4 32GBなら）
- **「賢さ重視・速度二の次」: Qwen 3.5 9B**（thinking mode の遅さ許容できれば）
- **「26B 級を動かす」: Modelfile で context 削減 + 32GB+ メモリ環境**
- **WSL2 (8GB) で本気出すなら**: Qwen 3.5 9B が現実ライン

---

## 🔗 参考リンク

- [Ollama 公式](https://ollama.com/)
- [LM Studio 公式](https://lmstudio.ai/)
- [Gemma 4 (Google DeepMind)](https://deepmind.google/models/gemma/gemma-4/)
- [Qwen 3.6 Unsloth Docs](https://unsloth.ai/docs/models/qwen3.6) — Ollama 非対応の出典
- [WSL2 + Ollama Setup Guide](https://insiderllm.com/guides/wsl2-ollama-windows-setup-guide/)
