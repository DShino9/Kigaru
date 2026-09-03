# 03. Mac 側パイプライン仕様

対象: Apple Silicon Mac（M1 以降 / 統合メモリ 16GB 以上推奨）/ macOS 26
実装: Python 3.12 + uv（パイプライン）、Swift（録音デーモン）

---

## 1. Mac 自身の録音（`mac-recorder`）

**在宅時の主役はこちら。** iPhone は保険という位置づけ。

理由: Zoom / Meet / Teams の会議では、iPhone のマイクは「スピーカーから出た相手の声」を空気越しに拾うだけで音質が悪い。
Mac なら**システム出力を直接タップできる**ため、相手の声が原音のまま録れる。文字起こし精度が段違いになる。

### 技術

- **Core Audio Process Tap**（`CATapDescription`, macOS 14.2+）でシステム出力を取得
- マイク入力は `AVAudioEngine`
- **2トラック別々に保存**する（mic.opus / system.opus）
  - 話者分離が劇的に楽になる。system トラック = 相手、mic トラック = 自分、と最初から分かっている
  - 従来の BlackHole + Aggregate Device 構成でも代替可能だが、Process Tap の方が設定が要らず堅牢

### 自動開始 / 停止

| トリガ | 動作 |
|---|---|
| 会議アプリのプロセス起動を検知（zoom.us / Teams / Google Meet のタブ / Slack huddle） | 録音開始 |
| カレンダーに会議予定がある時間帯 | 録音開始（予定名をメタに付与） |
| マイクが他アプリに使用中（`AVCaptureDevice` の in-use 検知） | 録音開始 |
| 上記がすべて解除 | 60 秒後に停止 |
| 手動（メニューバーアイコン） | 開始 / 停止 / 一時停止 |

**メニューバー常駐**とし、録音中はアイコンを赤く変える。macOS もマイク使用中はオレンジのインジケータを出す。

### 常時録音するか、会議のときだけか

**推奨: 既定は「会議のときだけ」。設定で常時に切り替え可能にする。**

在宅の 1日中を Mac で録ると、家族の会話や独り言が大量に混ざる。
価値密度が高いのは会議で、そこを確実に押さえる方が費用対効果が高い。
常時録音は iPhone に任せる。

---

## 2. パイプライン全体

launchd の LaunchAgent 1本で常駐デーモンを起動し、内部で SQLite のジョブキューを回す。
（ステージごとに別プロセスにすると障害切り分けが楽だが、まずは 1 プロセス + ステージ関数で十分）

```
inbox/ を FSEvents で監視
   ↓
[1] ingest    重複排除 → 16kHz mono wav 正規化 → jobs テーブルに登録
   ↓
[2] asr       mlx-whisper large-v3-turbo（word timestamps 付き）
   ↓
[3] diarize   pyannote/speaker-diarization-3.1 (device=mps)
   ↓
[4] merge     ASR × 話者 を時刻で突合 → transcript.json
   ↓
[5] segment   エピソード分割・デバイス間重複統合
   ↓
[6] enrich    PII マスキング → LLM で要約/論点/決定/ToDo/タグ → アンマスク
   ↓
[7] publish   Obsidian に Markdown 書き出し・デイリーノート追記
   ↓
[8] index     SQLite FTS5 + ベクタ索引を更新
```

各ステージは**冪等**にする。ジョブは `(chunk_id, stage)` で一意、失敗したら `retry_count` を上げて再投入。
3 回失敗したら `dead` にして通知（macOS 通知センター）。

---

## 3. ステージ詳細

### [2] ASR — モデル選定

| 候補 | 評価 |
|---|---|
| **mlx-whisper `large-v3-turbo`（第一候補）** | Apple Silicon ネイティブ（MLX）で高速。多言語・日本語良好。`--language ja` 指定 |
| **kotoba-whisper-v2.0（対抗）** | 日本語特化。distil 系で「爆速」と評される。日本語のみの用途なら有力 |
| Qwen3-ASR (MLX) | 30以上の言語対応で日本語あり。新しめなので実測次第 |
| Parakeet-TDT | **除外**。25の欧州言語のみで日本語非対応 |
| クラウド（Deepgram / AssemblyAI / OpenAI） | バースト時のフォールバックとしてのみ。既定では使わない（音声を外に出さない方針） |

> **実装時に必ず実測比較すること。** 自分の実際の会議音声 30 分 × 3 本で、
> WER（またはCER）、処理時間、話者交代時の崩れ方を比較して決める。
> 事前に決め打ちせず、`config.yaml` でモデルを差し替えられる構造にしておく。

出力: word-level タイムスタンプ付き JSON。

### [3] 話者分離

- `pyannote/speaker-diarization-3.1` を `device="mps"` で実行（Hugging Face のトークンと利用規約同意が必要）
- M1 Max で 1時間音声が数十秒〜数分のオーダー。実用範囲
- **Mac 録音の場合は 2トラック分かれているので diarize は system トラックにのみ適用**（自分の声は確定しているため）

#### 話者の名前解決（声紋 DB）

```
speakers テーブル
  speaker_id | name        | note_link          | embedding | sample_count
  spk_001    | 自分         | [[自分]]           | <vector>  | 1240
  spk_002    | 山田太郎     | [[山田太郎]]        | <vector>  |  87
  spk_017    | (未同定)     | null               | <vector>  |   3
```

1. pyannote の埋め込みを既知話者とコサイン類似度で照合（閾値 0.75）
2. 一致すれば名前を付与、しなければ `未同定_N` として登録
3. 未同定話者が一定回数（例: 5回）出現したら、Obsidian の `00_Inbox/` に
   「この人は誰ですか？」ノートを作り、ユーザーが名前を書くと以後自動適用
4. カレンダーの参加者リストが取れる会議では、それを候補として優先提示

**この「未同定話者の名寄せを人に聞く」ループが精度を決める。** 完全自動を狙わない。

### [5] エピソード分割

チャンクの列を「意味のある会話のかたまり」に切る。

分割ルール（いずれかに該当したら切る）:

| 条件 | 閾値 |
|---|---|
| 無音ギャップ | 30 分以上 |
| 話者集合の変化 | Jaccard 類似度 < 0.3 |
| 場所の変化 | 500m 以上の移動 |
| モーション状態の変化 | stationary ↔ automotive など |
| カレンダー予定の境界 | 予定の開始/終了時刻 |
| 上限 | 1 エピソード最大 3 時間で強制分割 |

分割後、**カレンダー突合**でタイトルを付ける（予定名 > 場所名 + 参加者 > LLM が生成した見出し の優先順）。

デバイス間重複統合は [01-architecture.md](01-architecture.md) の判断5 のとおり。

### [6] enrich（要約・論点抽出）

**PII マスキング層を必ず経由する。** 実装は `pipeline/masking.py` に独立させ、単体テストを厚く書く。

```
マスキング対象:
  人名（声紋DBの登録名 + 固有表現抽出）→ [PERSON_n]
  社名（許可リストにない社名）           → [COMPANY_n]
  電話番号 / メール / 住所 / 口座 / 生年月日 → 削除
  privacy: sensitive のエピソード         → enrich 自体をスキップ（ローカル LLM のみ）
```

LLM への指示（出力は JSON スキーマで固定）:

```json
{
  "title": "40字以内の見出し",
  "summary": ["3行の要約"],
  "decisions": ["決まったこと"],
  "issues": ["未解決の論点・問い"],
  "todos": [{"text": "…", "owner": "[PERSON_1]", "due": "2026-09-10"}],
  "people": ["[PERSON_1]", "[PERSON_2]"],
  "topics": ["タグ候補"],
  "importance": 3,
  "highlights": [{"time": "14:22", "speaker": "[PERSON_1]", "text": "…"}]
}
```

- 使用モデル: Claude（`claude-sonnet-5` 相当をコスト効率で。重要度の高いものだけ上位モデル）
- `privacy: sensitive` の場合はローカル LLM（Ollama の `qwen3` 等）にフォールバック
- 出力を受け取ったらローカルの対応表でアンマスクして実名に戻す

### [7] publish

Obsidian Vault への書き出し。詳細は [04-obsidian-vault.md](04-obsidian-vault.md)。

- エピソードノートを新規作成（既存があれば frontmatter をマージ更新）
- デイリーノート `10_Daily/YYYY-MM-DD.md` の該当セクションに追記
- 人物ノート `25_People/` の「最近の会話」に逆リンクを追記
- **Obsidian が開いていても安全に書けるよう、一時ファイル + atomic rename で書く**

### [8] index

- SQLite FTS5 に全文（発話単位）を投入。日本語は N-gram トークナイザ（trigram）を使う
- ベクタ索引: 発話を 500 字程度のチャンクに切って埋め込み。`mlx-embeddings` でローカル生成
- エージェントの Step1・Step3 はここに問い合わせる

---

## 4. 日次バッチ（深夜 02:00）

| 処理 | 内容 |
|---|---|
| デイリーノート仕上げ | その日のエピソード一覧、決定事項、ToDo をまとめて `10_Daily/` に |
| 重要度の再計算 | 手動マーク・発話量・登場人物・カレンダー種別からスコア再計算 |
| 音声の保持期限処理 | 90日超かつ `importance < 4` かつ `pin != true` の音声を削除 |
| 未同定話者の問い合わせノート生成 | `00_Inbox/` に作成 |
| 週次（日曜） | 「今週の論点トップ5」を生成し、エージェント起動候補として提示 |
| ヘルスチェック | dead ジョブ、未転送チャンク滞留、ディスク残量を通知 |

---

## 5. 運用・信頼性

### launchd

```
~/Library/LaunchAgents/
  com.ds9i.lifelog.daemon.plist    # 常駐（KeepAlive=true）
  com.ds9i.lifelog.nightly.plist   # 02:00 起動
  com.ds9i.lifelog.recorder.plist  # mac-recorder 常駐
```

### 監視

- 各ステージの処理件数・所要時間・失敗数を `index.db` の `metrics` テーブルに記録
- 異常時は macOS 通知 + Obsidian `90_Meta/health.md` に追記
- **「静かに壊れて何も記録されていなかった」が最悪**なので、
  「直近 6 時間で 1 チャンクも取り込まれていない」を異常として通知する

### セキュリティ

- `~/LifelogStore/` は **FileVault 有効な内蔵ディスク**、または暗号化した外付け APFS ボリューム
- Time Machine のバックアップ対象に含める（ただし暗号化バックアップに限る）
- 受信 API は Tailscale 内のみに bind（`0.0.0.0` に晒さない）
- API トークンは Keychain に保存

### バックアップ

| 対象 | 方法 | 頻度 |
|---|---|---|
| Obsidian Vault | Git（プライベートリポジトリ、E2E は別途）+ Obsidian Sync | 随時 |
| `transcript.json` 全文 | 暗号化して外付け SSD へ rsync | 日次 |
| 音声 | バックアップしない（90日で消える前提） | — |
| `index.db` | 再生成可能なのでバックアップ不要（transcript から再構築できる設計にする） | — |
