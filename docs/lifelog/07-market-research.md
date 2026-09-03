# 07. 類似サービス調査（2026年9月時点）

「自作せずに済むなら済ませる」ために、既存の選択肢を先に潰しておく。

---

## 1. 常時録音デバイス / サービス

| サービス | 形態 | 状況（2026-09） | 本構想から見た評価 |
|---|---|---|---|
| **Limitless（旧 Rewind）** | ペンダント + アプリ | **2025年12月に Meta が買収。ペンダントは新規販売停止。** 既存所有者は最低1年 Unlimited プランに移行、その後は先行き不透明 | 24時間ライフログの完成度は最も高かった。**タイムライン UI は本構想の UI 参考にする**。ただし製品としては選べない |
| **Plaud（Note / NotePin）** | 専用ハード | 継続 | 対面キャプチャの定番。ただしデータはクラウド、Obsidian 連携が弱い |
| **Omi（BasedHardware）** | ペンダント + アプリ（**オープンソース**） | 継続 | **iOS アプリのコードが参考になる**（背景録音・Opus・オフライン録音・チャンク管理が実装済み）。Flutter 製 |
| **Bee** | ペンダント | Amazon が買収済み | 同上の理由でロックイン懸念 |
| **Hedy** | ソフトのみ（Mac/Win/iOS/Android） | Limitless 難民の受け皿として位置づけ | ハード不要は魅力。ただしクラウド前提 |
| **LUCI** | ローカルファースト | 会議 + 画面活動のキャプチャ | 思想は近い |

**結論: 「ローカルにデータを持ったまま常時録音する」選択肢が市場から減っている。**
Limitless の終売と Screenpipe の有料化が同時期に起きており、自作の合理性は 1 年前より上がっている。

---

## 2. 常時キャプチャの OSS

| プロジェクト | 状況 | 評価 |
|---|---|---|
| **Screenpipe** | ライセンスが MIT から **source-available に変更**、デスクトップアプリは有料化 | 画面 + 音声を 24/7 記録し Whisper でオンデバイス文字起こし、という構成は本構想とほぼ同じ。**アーキテクチャの参考にする価値は高いが、依存先としては選べない** |
| **OpenRecall** | AGPLv3 で完全にオープン | ローカル完結。ただし画面中心で音声は弱い |
| **Omi** | オープンソース | 上記のとおり iOS 実装の参考 |
| **whisper.cpp / mlx-whisper** | 活発 | **採用**。Mac 側 ASR の中核 |
| **pyannote.audio** | 活発 | **採用**。話者分離 |

---

## 3. 文字起こしツール（Mac / 日本語）

| ツール / モデル | 日本語 | Apple Silicon | 評価 |
|---|---|---|---|
| **mlx-whisper `large-v3-turbo`** | ○（`--language ja`）| MLX ネイティブ | **第一候補**。速度と精度のバランスが良い。量子化版（4bit/8bit/fp16）も選べる |
| **kotoba-whisper-v2.0** | ◎ 日本語特化 | ○ | **対抗馬**。「爆速」と評される distil 系。日本語のみの用途なら有力 |
| **Qwen3-ASR (MLX)** | ○（30以上の言語） | ○ | 新しめ。実測次第 |
| **Parakeet-TDT** | **×（25の欧州言語のみ）** | ◎ 非常に高速 | **日本語非対応のため除外**。英語なら圧倒的に速い |
| **iOS 26 SpeechAnalyzer** | ○（ja_JP） | オンデバイス | **iPhone 側の速報テキストに採用**。1分制限とオンライン依存が撤廃され、長時間音声に対応 |
| superwhisper / MacWhisper / Aiko | ○ | ○ | GUI ツール。手動運用には便利だが自動パイプラインには組み込みにくい |

> **Parakeet が日本語非対応**という点は見落としやすい。速いという評判だけで採用すると詰む。

---

## 4. 会議ノートツール（UI と要約の参考）

| ツール | 参考にする点 |
|---|---|
| **Granola** | 「要約が主・全文は折りたたみ」のレイアウト。人が読む前提の情報設計 |
| **Otter.ai** | 話者ラベル付きトランスクリプトの見せ方、ハイライト機能 |
| **tl;dv / Fathom** | 会議の自動検知と自動開始（Mac 側の自動トリガの参考） |
| **Notion AI ミーティングノート** | 元スライドの構成。議事録 → 構造化の流れ |

---

## 5. Obsidian 連携

| 方式 | 状況 | 評価 |
|---|---|---|
| **Vault フォルダを直接操作** | 常に可能 | **推奨**。Obsidian の起動有無に依存しない |
| **Local REST API プラグイン内蔵 MCP**（v4 以降 / 2026年5月） | 別途ブリッジサーバ不要になった。`http://127.0.0.1:27123/mcp/` に Bearer トークンで接続 | 対話中の操作に強い。**補助として採用** |
| Filesystem MCP を Vault に向ける | 可能 | フォルダ直接操作とほぼ同義 |
| Obsidian 公式 CLI / headless Sync (v1.12) | 登場 | 自動化の選択肢が増えた |
| **Bases**（コア機能） | Dataview の公式版という位置づけ。**5万ノート規模でもほぼ即座に描画**、frontmatter を直接編集可 | **採用**。Dataview はモバイルで顕著に遅くなるため使わない |

---

## 6. 「作らない」選択肢の検討

正直に評価しておく。

| 案 | 満たせるもの | 満たせないもの |
|---|---|---|
| Plaud を買う | 対面録音・文字起こし・要約 | データがクラウド。Obsidian に構造化して貯まらない。エージェントの入力にできない |
| Otter / tl;dv を使う | 会議の記録と要約 | 会議以外が録れない。データが外部 |
| Hedy を使う | ハード不要の常時記録 | クラウド前提。Vault 連携なし |
| **純正ボイスメモ + 手動で文字起こし** | ゼロコスト | 自動で貯まらない。運用が続かない |
| **本構想（自作）** | すべて | 7〜9週間の開発と継続的な保守 |

**判断:** 「会議の記録が欲しい」だけなら既製品で足りる。
本構想の価値は **「Obsidian に構造化されて貯まり、それをエージェントの入力にできる」** ことに尽きる。
逆に言えば、**エージェント活用（Phase 3）まで到達しないなら自作する意味は薄い。**
Phase 0 の判定でここを見極めること。

---

## 出典

- [Limitless Pendant Discontinued: Best Alternatives (2026) — LUCI](https://luci.memories.ai/blog/limitless-pendant-discontinued-alternatives)
- [【常時録音 + AI文字起こし】Limitless Pendant の代わりは何が良い？ — note](https://note.com/tonochan/n/n867c9bc3a9ce)
- [Best Limitless AI Alternative 2026: Open-Source Screenpipe vs Limitless — Screenpipe Blog](https://screenpipe.com/blog/screenpipe-vs-limitless-2026)
- [6 Best Screenpipe Alternatives in 2026](https://www.usecarly.com/blog/screenpipe-alternatives/)
- [BasedHardware/omi — GitHub](https://github.com/BasedHardware/omi)
- [爆速でローカル動作する日本語特化の文字起こしAI『kotoba-whisper-v2.0』の実力は？ — 窓の杜](https://forest.watch.impress.co.jp/docs/review/1635025.html)
- [Apple Silicon ネイティブの文字起こしツール MacScribe を設計、MLX Whisper の5モデルを M4 で実測 — DevelopersIO](https://dev.classmethod.jp/articles/20260506-macscribe-mlx-whisper/)
- [NVIDIA Parakeet-TDT × Apple Silicon で実現する爆速ローカル文字起こし — Qiita](https://qiita.com/faunsu/items/803eaca7431ff52c2a0c)
- [SpeechAnalyzerとは？iOS 26の音声認識APIの使い方とWhisper比較 — issoh](https://www.issoh.co.jp/tech/details/8929/)
- [Apple SpeechAnalyzerの日本語ストリーミング認識を検証 — Qiita](https://qiita.com/kiarina/items/ac054e60eecb410e748b)
- [iOS 26 SpeechAnalyzerによるAI音声認識処理の比較と落とし穴 — Sansan Tech Blog](https://buildersbox.corp-sansan.com/entry/2026/02/13/130000)
- [Enhance your app's audio recording capabilities — WWDC25](https://developer.apple.com/videos/play/wwdc2025/251/)
- [pyannote.audioとWhisperで構築する議事録自動化システム](https://media.tcdigital.jp/ai-knowledge-flow/articles/pyannote-whisper-speaker-diarization/)
- [obsidian-local-rest-api — GitHub](https://github.com/coddingtonbear/obsidian-local-rest-api)
- [Obsidian MCP Setup 2026: Local REST API Complete Guide — MCP.Directory](https://mcp.directory/blog/obsidian-mcp-complete-guide-2026)
- [Obsidian Bases Plugin vs Dataview: Which to Use in 2026 — locul.ai](https://locul.ai/blog/obsidian-bases-plugin)
- [個人でiOSアプリをApp Storeに公開する完全ガイド【2026年版】 — ぽちょ研究所](https://pochanglab.com/blog/personal_ios_app)
