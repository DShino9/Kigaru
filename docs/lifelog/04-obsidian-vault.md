# 04. Obsidian Vault 設計

Vault 名: **DS9i**（既存の Vault を使う。会話ログ用の名前空間を追加する形）

---

## 1. 設計原則

1. **Vault は「人が読むもの」だけを置く。** 全文と音声は Vault 外（`~/LifelogStore/`）。
2. **1エピソード = 1ノート。** 1ノートは 1〜3 KB に収める。
3. **frontmatter がデータベースの実体。** 本文は人が読むための表現。
4. **リンクは人・組織・テーマに張る。** 日付リンクは自動、テーマリンクは enrich が付ける。
5. **DB ビューは Dataview ではなく Bases。** 数万ノート規模で描画性能が実用に耐える。
6. **自動生成ノートと手書きノートを混ぜない。** `20_Voice/` は機械が書く領域と決め、人は手を入れない（手を入れたい内容は別ノートに切り出してリンクする）。

---

## 2. フォルダ構成

```
DS9i/
├─ 00_Inbox/                    人への問い合わせ（未同定話者の確認など）
├─ 10_Daily/
│   └─ 2026-09-03.md            デイリーノート（自動生成 + 手書き追記可）
├─ 20_Voice/                    会話エピソード（機械が書く領域）
│   └─ 2026/09/
│       ├─ 2026-09-03_1405_A社定例.md
│       └─ 2026-09-03_1230_移動中メモ.md
├─ 25_People/                   人物ノート（声紋IDと紐付く）
│   ├─ 山田太郎.md
│   └─ 自分.md
├─ 27_Orgs/                     組織ノート
├─ 30_Issues/                   論点ノート ← エージェント Step2 の出力
├─ 40_Research/                 リサーチノート ← Step3 の出力
├─ 50_Reports/                  戦略レポート ← Step4/5 の出力
│   └─ _work/<run_id>/          エージェント実行の中間成果物
├─ 90_Meta/
│   ├─ Bases/
│   │   ├─ voice.base
│   │   ├─ issues.base
│   │   ├─ people.base
│   │   └─ reports.base
│   ├─ health.md                パイプラインの稼働状況
│   └─ Templates/
└─ .claude/
    ├─ agents/                  5体のサブエージェント定義
    ├─ commands/strategize.md   スラッシュコマンド
    └─ skills/
```

---

## 3. エピソードノートのスキーマ

`20_Voice/2026/09/2026-09-03_1405_A社定例.md`

```markdown
---
type: voice-episode
id: ep_20260903_1405_a1b2
date: 2026-09-03
start: 2026-09-03T14:05:12+09:00
end: 2026-09-03T15:02:40+09:00
duration_min: 57
context: meeting               # meeting | call | field | solo | ambient
device_primary: mac
device_all: [mac, iphone]
location: 自宅オフィス
calendar_event: "A社 定例MTG"
people:
  - "[[自分]]"
  - "[[山田太郎]]"
  - "[[未同定_03]]"
orgs: ["[[A社]]"]
speaker_count: 3
asr_model: mlx-whisper-large-v3-turbo
asr_stage: final               # draft | final
asr_confidence: 0.91
transcript_path: "~/LifelogStore/episodes/2026/09/ep_20260903_1405_a1b2/transcript.json"
audio_path: "~/LifelogStore/episodes/2026/09/ep_20260903_1405_a1b2/audio.opus"
audio_retention_until: 2026-12-02
importance: 4                  # 0-5（手動マーク優先、なければ自動スコア）
pin: false
privacy: normal                # normal | sensitive | excluded
status: enriched               # raw | enriched | issued | reported
tags: [voice, 顧客/A社, テーマ/価格戦略]
---

## 要約
- 新プランの価格帯を月額 3 万円台に置く案が中心。
- 競合 B 社の値下げをどう見るかで意見が割れた。
- 次回までに顧客 5 社へのヒアリング結果を持ち寄る。

## 決定事項
- 価格の最終決定は次回定例に持ち越し
- ヒアリング項目は山田さんが 9/6 までにドラフト

## 論点
- 値下げ追随は短期売上と長期ブランドのどちらを優先するか
- 3万円台という価格が「誰にとって」妥当なのかが未検証

## ToDo
- [ ] ヒアリング項目ドラフト（山田太郎 / 9-06）
- [ ] 競合 B 社の価格改定の背景を調べる（自分 / 9-05）

## ハイライト
> 14:22 山田太郎「追随したら、うちは何屋なのか分からなくなる」
> 14:48 自分「そもそも 3万円が高いと言っているのは誰なのか」

## 参照
- 全文: `transcript.json`（[[90_Meta/全文の開き方]]）
- 前回: [[2026-08-27_1400_A社定例]]
```

### `status` のライフサイクル

```
raw ──enrich──> enriched ──/strategize の Step2──> issued ──Step5──> reported
```

エージェントは `status: enriched` かつ `importance >= 3` のノートを主な入力候補とする。

---

## 4. 人物ノート

`25_People/山田太郎.md`

```markdown
---
type: person
speaker_id: spk_002
org: "[[A社]]"
role: 事業開発部長
first_seen: 2025-11-12
last_seen: 2026-09-03
episode_count: 87
aliases: [山田, Yamada]
privacy: normal
---

## メモ
（手書き。人の判断で書く）

## 最近の会話
（publish が自動更新する直近10件のリンク）
```

**声紋 ID と Obsidian ノートを 1:1 で紐付ける**のがこの設計の要。
`speaker_id` が入っているノートを見て、パイプラインが自動で名前解決する。

---

## 5. Bases（DB ビュー）

`90_Meta/Bases/voice.base` の想定ビュー:

| ビュー名 | フィルタ | 表示列 |
|---|---|---|
| 今週の会話 | `date >= 今週の月曜` | date, title, people, duration_min, importance |
| 重要（未処理） | `importance >= 4 AND status == "enriched"` | date, title, issues 数 |
| 顧客別 | `tags contains "顧客/"` でグループ化 | org, date, title |
| 未同定話者あり | `people contains "未同定"` | date, title, people |
| 音声が消える直前 | `audio_retention_until <= 7日後 AND importance >= 3` | 保存するか判断する導線 |

`issues.base` / `reports.base` も同様に、エージェント成果物の一覧性を確保する。

> Dataview を使わない理由: 5万ノート規模で Bases はほぼ即座に描画するが、
> Dataview は特にモバイルで顕著に遅くなる。会話ログは年 3,000〜4,000 ノート増えるので、
> 数年で確実にその規模に到達する。

---

## 6. デイリーノート

`10_Daily/2026-09-03.md`

```markdown
---
type: daily
date: 2026-09-03
episode_count: 7
speech_minutes: 312
---

## 今日の会話
| 時刻 | エピソード | 人 | 重要度 |
|---|---|---|---|
| 10:00 | [[2026-09-03_1000_社内定例]] | 3人 | ⭐️⭐️ |
| 14:05 | [[2026-09-03_1405_A社定例]] | 3人 | ⭐️⭐️⭐️⭐️ |

## 今日決まったこと
- （各エピソードの decisions を集約）

## 今日出た論点
- （各エピソードの issues を集約）

## ToDo
- [ ] （集約）

---
## 日記（手書き）
（この線から下は人が書く。パイプラインは触らない）
```

**「手書き領域」と「自動領域」を水平線で明示的に分ける。**
パイプラインは水平線より上だけを書き換える。これで自動更新と手書きが衝突しない。

---

## 7. 全文へのアクセス方法

Vault に全文を置かないので、代わりに以下を用意する:

1. **Obsidian の URI ハンドラ or シェルコマンドプラグイン**で、
   ノートから `transcript.json` を専用ビューアで開けるようにする
2. **簡易ビューア**（ローカルの静的 HTML）: 話者色分け・タイムスタンプ・音声再生・検索
3. **エージェントは索引経由**でアクセス（`~/LifelogStore/index.db`）

---

## 8. Obsidian と Claude Code の接続

| 方式 | 評価 |
|---|---|
| **Vault フォルダを直接 Claude Code の作業ディレクトリにする（推奨）** | API キー不要、Obsidian が起動していなくても動く、Git 管理と相性が良い |
| Local REST API プラグインの内蔵 MCP（v4 以降） | Obsidian のライブメタデータ・アクティブファイル・コマンドパレットに触れる。「今開いているノートについて」のような操作に強い |
| Filesystem MCP を Vault に向ける | 上記1とほぼ同じ。Claude Desktop から使うなら有効 |

**推奨: 基本はフォルダ直接操作。** Obsidian のプラグイン状態に依存しないのが最大の利点で、
夜間バッチやエージェントの自動実行が Obsidian の起動有無に左右されなくなる。

Local REST API の MCP は「対話中に今見ているノートを操作したい」用途で**追加的に**入れる。
接続例:

```
claude mcp add --transport http obsidian http://127.0.0.1:27123/mcp/ \
  --header "Authorization: Bearer <api-key>"
```

---

## 9. Vault の肥大化対策

| 対策 | 内容 |
|---|---|
| 全文を Vault 外に置く | 最重要。これだけでノートサイズが 1/50 になる |
| `20_Voice/` を年/月で階層化 | 1フォルダあたりのファイル数を数百に抑える |
| 低重要度エピソードの週次ロールアップ | `importance <= 1` は 90日後に週次サマリノートへ統合し、個別ノートを削除 |
| Bases を使う | Dataview の全ノート走査を避ける |
| 添付ファイルを Vault に置かない | 音声・画像は `LifelogStore` へ |
| Obsidian Sync のフォルダ除外 | `50_Reports/_work/` は同期対象から外す |
