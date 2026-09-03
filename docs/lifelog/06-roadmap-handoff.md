# 06. ロードマップ・コスト・Mac への引き渡し

---

## 1. フェーズ計画

### Phase 0 — 価値検証（1週間） ★ここを飛ばさない

**目的: 「本当に読み返すのか」を、iPhone アプリを書く前に確かめる。**

| やること | やらないこと |
|---|---|
| Mac の会議録音（`mac-recorder` の最小版、手動開始でも可） | iPhone アプリ |
| mlx-whisper で文字起こし（スクリプト実行でも可） | 話者分離 |
| Claude で要約 → Obsidian にノート生成 | 索引・ベクタ検索 |
| 1週間、実際の会議 5〜10 本を通す | UI |

**判定基準（1週間後に自問する）:**

| 問い | Yes なら |
|---|---|
| 生成されたノートを 1回でも読み返したか | 続行 |
| 「あれ何て言ってたっけ」を検索して解決した経験があるか | 続行 |
| 要約の精度は「読む価値がある」水準か | 続行。No なら enrich のプロンプトを直す |
| 会議以外の会話も録りたいと思ったか | Phase 1（iPhone）へ進む。No なら **Mac だけで完結させる**のが正解 |

> **最後の問いが重要。** Mac の会議録音だけで満足するなら、iPhone アプリの 3週間と
> 電池の不便と社会的リスクを丸ごと回避できる。それは負けではなく、正しい撤退。

### Phase 1 — iPhone 録音アプリ MVP（2〜3週間）

- Apple Developer Program 加入
- 背景録音 / VAD / チャンク / 転送キュー / Tailscale 送信
- UI は「今日」タブと停止導線だけ。検索・重要タブは後回し
- **完了条件**: 丸1日 iPhone を持ち歩いて、翌朝 Mac に全チャンクが揃っている

### Phase 2 — 構造化（2週間）

- 話者分離 + 声紋 DB + 未同定話者の問い合わせループ
- エピソード分割 + カレンダー突合 + デバイス間重複統合
- enrich（PII マスキング層を含む）
- Obsidian 公開 + Bases + デイリーノート
- SQLite FTS5 + ベクタ索引
- **完了条件**: 「先月 A社と価格の話をした回」を 30 秒以内に見つけられる

### Phase 3 — エージェント 5体（2週間）

- `.claude/agents/` 7ファイル + `/strategize`
- 実データで 10 回実行してプロンプトを調整
- **完了条件**: 自分で書くより速く、かつ出典が全部辿れるレポートが出る

### Phase 4 — 運用改善（継続）

- iOS 26 SpeechAnalyzer 統合（速報テキスト）
- 重要マークのウィジェット / Action Button / Apple Watch
- 週次自動実行、重要度スコアの学習
- 全文ビューア

**合計: 約 7〜9 週間**（Phase 0 の判定次第で Mac 完結なら 1〜2 週間）

---

## 2. コスト

### 初期

| 項目 | 金額 |
|---|---|
| Apple Developer Program（Phase 1 以降で必須） | ¥15,800 / 年 |
| 外付け暗号化 SSD 2TB（`LifelogStore` 用） | ¥15,000〜25,000 |
| Tailscale | 個人利用は無料 |
| **合計** | **約 ¥3〜4万** |

### ランニング（月額）

| 項目 | 金額 | 備考 |
|---|---|---|
| ASR | ¥0 | ローカル完結 |
| enrich（要約・論点抽出） | ¥2,000〜5,000 | 1日 6〜9 時間分。中位モデル使用 |
| エージェント実行 | ¥500〜3,000 | 週 2〜5 回想定 |
| Obsidian Sync（任意） | ¥600〜 | |
| **合計** | **約 ¥3,000〜9,000 / 月** | |

> enrich をローカル LLM（Ollama）に寄せれば月額はほぼゼロにできる。
> 精度とのトレードオフなので、Phase 2 で両方試して決める。

---

## 3. 技術スタック確定表

| レイヤ | 採用 | 代替（実測で覆してよい） |
|---|---|---|
| iOS 録音 | Swift / SwiftUI, AVAudioSession, `UIBackgroundModes: audio` | Omi をフォーク |
| iOS 端末内 ASR | iOS 26 `SpeechAnalyzer` / `SpeechTranscriber` (ja_JP) | なし（この API 一択） |
| iOS VAD | Silero VAD (Core ML) | エネルギー + ゼロ交差の簡易版 |
| 転送 | Tailscale + URLSession background transfer | iCloud Drive フォルダ |
| Mac 録音 | Swift, Core Audio Process Tap (macOS 14.2+) | BlackHole + Aggregate Device |
| Mac ASR | `mlx-whisper large-v3-turbo` | `kotoba-whisper-v2.0`, Qwen3-ASR (MLX) |
| 話者分離 | `pyannote/speaker-diarization-3.1` (mps) | — |
| パイプライン | Python 3.12 + uv, SQLite | — |
| enrich / エージェント | Claude API（Sonnet 中心、構造化と検証は上位モデル） | Ollama（sensitive 用） |
| 索引 | SQLite FTS5 (trigram) + `mlx-embeddings` | — |
| ノート | Obsidian + **Bases**（Dataview は使わない） | — |
| Obsidian 接続 | Vault フォルダ直接操作 | Local REST API 内蔵 MCP（補助） |
| 常駐 | launchd LaunchAgent | — |

---

## 4. Mac の Claude Code に渡す起動プロンプト

以下をそのまま貼れば着手できる。

```
このリポジトリの docs/lifelog/ に、常時録音 × 自動文字起こし × Obsidian ×
AIエージェントの構想書一式がある。まず README.md から 08 まで全部読んでほしい。

読んだうえで、Phase 0（価値検証）を実装する。

【Phase 0 のスコープ】
- Mac の会議音声を録音する最小のツール
  - マイク入力とシステム出力を別トラックで録る（Core Audio Process Tap, macOS 14.2+）
  - まずは手動の開始/停止でよい。メニューバー常駐は後回し
- 録音ファイルを mlx-whisper large-v3-turbo（--language ja）で文字起こし
- Claude API で要約・決定事項・論点・ToDo を JSON で抽出
  （PII マスキング層は Phase 0 では簡易版でよいが、モジュールとしては分離しておく）
- Obsidian Vault (~/Documents/DS9i) の 20_Voice/ にエピソードノートを生成
  frontmatter のスキーマは docs/lifelog/04-obsidian-vault.md のとおり
- 全文 transcript.json は Vault 外（~/LifelogStore/）に置く

【Phase 0 でやらないこと】
- iPhone アプリ、話者分離、索引、Bases、エージェント、UI

【進め方】
1. まず docs/lifelog/03-mac-pipeline.md の「ASR モデル選定」に従い、
   実際の会議音声で mlx-whisper large-v3-turbo と kotoba-whisper-v2.0 を実測比較して、
   結果を docs/lifelog/benchmarks.md に記録すること。決め打ちしない。
2. 実装は ~/dev/lifelog/ に置く（このリポジトリとは別）。構成は 01-architecture.md の §4 に従う。
3. 動いたら 1週間の実運用に入り、README.md の Phase 0 判定基準に沿って評価する。

【重要な制約】
- 音声は外部に送らない。ASR はローカル完結。
- 外部 LLM に渡すのは PII マスキング後のテキストのみ。
- 停止機構は最初から入れる。「止めたいときに止められない」は許容しない。
```

---

## 5. 引き渡し時に伝えるべき注意点

1. **Phase 0 の判定を真面目にやること。** 実装が楽しくて先に進みたくなるが、
   「読み返さないログ」を大量生産するのが最悪の結末。
2. **ASR モデルは実測で決めること。** 本構想書は `large-v3-turbo` を第一候補としているが、
   日本語の会議音声では kotoba-whisper が勝つ可能性が十分ある。
3. **PII マスキング層は最初から分離しておくこと。** 後から差し込むのは難しい。
   Phase 0 では簡易版でよいが、モジュール境界は切っておく。
4. **`privacy` フィールドは Phase 0 から frontmatter に入れること。** 後付けだと過去ノートの一括修正が要る。
5. **静かに壊れることを一番警戒すること。** 「6時間 1件も取り込まれていない」を異常として通知する。
