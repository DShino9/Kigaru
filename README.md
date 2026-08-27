# 入り口ページ

LINEに貼る用の、地図の前に置く1枚。

## なぜ要るのか

地図の本体は Google Apps Script で動いていて、**Google のページの枠（iframe）の中**にある。
そのため、外側のページに手が出せず、次の3つがどうしてもできなかった。

1. **LINEに貼っても、見出しも絵も出ない** — 長いURLだけが届いて、怪しいリンクに見える
2. **ホーム画面に追加しても、アイコンが「気」の一文字**になる
3. **URLが長すぎて読めない**

この1枚を別の場所に置くことで、3つとも片づく。
ここから地図へ送るだけの、ごく小さなページ。

## 中身

- `index.html` … 本体。OGP・apple-touch-icon・favicon をここで指定している
- `icon-512.png` / `apple-touch-icon.png` / `favicon.png` / `ogp.jpg` … 雲のアイコン
- `pin-*.png` … 地図のピン4種類

作り直すときは `道具/アイコンを作る.py` と `道具/ピンを作る.py`。

## 置き方（GitHub Pages）

置き場は **DShino9 / Kigaru**、公開されるURLは **https://dshino9.github.io/Kigaru/**。
`index.html` の og:image と og:url に、このURLを直接書いてある。
**置き場を変えたら、そこも直すこと**（LINEは絶対URLでないと絵を出さない）。

1. GitHub で新しい置き場を作る … 名前は `Kigaru`、**Public**、
   「Add a README file」は**チェックを外す**（この中の README を使うため）
2. 出てきた画面の **uploading an existing file** を押す
3. **この `入り口` フォルダの中身を全部**、まとめてドラッグして落とす
   （フォルダごとではなく、中のファイルを全部）
4. 下の **Commit changes** を押す
5. 上の **Settings** → 左の **Pages** →
   Branch を **main**、フォルダを **/ (root)** にして **Save**
6. 数分待つと https://dshino9.github.io/Kigaru/ で開けるようになる

## 直したあと、LINEで絵が出ないとき

LINEは一度読んだページの絵をしばらく覚えている。
直したのに古い絵が出るときは、URLの後ろに `?2` などを付けて貼ると読み直す。

## 数字を直すとき

`index.html` の「73 か所」「34 本の空撮動画」は手書き。
場所が増えたら、管理の「場所の一覧を出す」で数えて書き換える。
