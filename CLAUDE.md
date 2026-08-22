# KMT業務管理システム — 開発ガイド

⚠️ **このファイルは公開リポジトリにコミットされます。トークン・パスワード・個人情報は絶対に書かないこと。**

## 1. このリポジトリについて

- アプリ本体は `kmt.html` の**1ファイルのみ**（約18,000行 / HTML + JS + CSS 埋め込み、ビルド不要）
- 外国人材（特定技能）の就労管理システム。所属機関・人材・求人・候補者・売上・チャット等を管理
- バックエンド: Supabase（project_id `oyblvyocvdnhcxeeokuw`）。接続キーは `kmt.html` 内の `SB_KEY`（publishableキー）
- `sales.html` は旧・売上単体ページ（現在はメンテ対象外）

## 2. 作業ルール

- **コードを編集する前に、必ず該当箇所を Read してから Edit する。** 1ファイルが巨大なので、当てずっぽうの置換は事故になる
- **push はユーザーが明示的に「プッシュ」と言ったときだけ。** 通常は実装 → 構文チェック → 動作確認 → 差分／ファイルを見せて確認してもらう
- ユーザーはカジュアルな日本語。回答は簡潔に。技術的な判断理由は短く添える
- 仕様が曖昧なときは実装前に確認する（特に「どこまで適用するか」の範囲）

## 3. バージョン表記は毎回3箇所を更新

新しい変更を入れたら必ず全部そろえる。

1. 7行目付近 `<!-- KMT_VERSION ver.YYYYMMDD.N -->`
2. `logo-ver` の表示テキスト（370行付近）
3. `WHATS_NEW` 配列の**先頭**に新エントリを追加
   ```js
   { version:'ver.20260805.29', date:'2026-08-05', notify:true, items:[
     'ユーザー向けの変更点を日本語で。何がどう変わるかを具体的に',
   ]},
   ```

`WHATS_NEW` はアプリ内「アップデート履歴」に出る＝**社内ユーザーが読む文章**。技術用語ではなく操作ベースで書く。

## 4. 構文チェック（納品・コミット前に必ず）

最大の `<script>` ブロックを抜き出して `node --check` にかける。

```bash
cd <リポジトリ> && python3 -c "
import re
b=max(re.findall(r'<script[^>]*>(.*?)</script>', open('kmt.html',encoding='utf-8').read(), re.S),key=len)
open('/tmp/_chk.js','w',encoding='utf-8').write(b)" && node --check /tmp/_chk.js
```

## 5. Playwright での動作確認

UI変更やロジック変更のときは実際に描画して確かめる。

- ログインをスキップ: `addInitScript` で
  `localStorage.setItem('kmt_current_user', JSON.stringify({id,name,role}))`
- DBは `window.sbFetch` をスタブに差し替えるのが確実
  （モジュールスコープの `let`（`allWorkers` 等）は `window.x=` では書き換わらない。
  `page.evaluate` 内で直接代入するか、`sbFetch` をモックする）
- 画面の初期化（`showScreen(...)`）が `allSales` 等を上書きするので、
  **テストデータの流し込みは初期化の後**に行う
- UI変更はスクリーンショットを撮ってユーザーに見せると早い

## 6. Supabase

- スキーマ変更・データ確認は Supabase MCP（`execute_sql`）または管理画面から
- 新テーブルは既存に合わせて `DISABLE ROW LEVEL SECURITY` ＋ `GRANT ALL TO anon, authenticated, service_role`
- アプリ→Supabase はブラウザから通るので、大量データの初期投入は
  「アプリ埋め込みシード＋初回アクセス時に自動投入」方式が使える（`JOB_PROGRESS_SEED` が実例）
- 更新者は各テーブルの `updated_by` に記録する（`_curUserName()` を使う）

## 7. 主要な仕組み

### 手動保存と未保存リマインド

- レジストリ `_manualSave`（modalId → `{dirty, prefix, save, metaSlot, label}`）
- `bindDirtyTracking(modalId)` … 入力を監視して dirty を立てる
- `promptSaveModal(modalId, afterFn)` … 「保存しますか？」の共通ダイアログ
- `closeGenModal` / `showScreen` の両方が `_manualSave` を見るので、
  ✕ボタン・オーバーレイ・［閉じる］・画面移動のどれでもリマインドが出る
- 外国人材・所属機関・履歴書はスナップショット比較で dirty 判定
  （`_snapshotDirty()` / `_resumeFormDirty()`）＝実際に値が変わったときだけ出る
- 保存ボタンの隣に `renderSaveMeta(slotId, updated_at, updated_by)` で
  「✓ 最終更新 日時（ユーザー名）」を表示。編集中は `markSaveMetaDirty()`
- **手動保存フォームは保存してもモーダルを閉じない**（最終更新をその場で見せるため）
- 削除直後など、チェックを飛ばして閉じたいときは `forceCloseModal(id)`

### 自動保存のモーダル

`_AUTOSAVE_MAP` を参照。expense / partner / fee / user / sending / aiKn / intro は自動保存
（入力1.5秒後に保存、右下に「✅保存済」）。`null` のものは手動保存。

### チャット

#### ルームの取得は2系統に分かれている（絶対に1本化しない）

チャットは画面が2つあり、**取得も完全に別々**にしてある。片方が失敗しても
もう片方が巻き添えで消えないようにするため。

| 系統 | 出る画面 | 対象ルーム | embed |
|---|---|---|---|
| `general` | 💬 チャット | `worker_id` か `company_id` を持つ | `workers(...)` / `companies(...)` |
| `job` | 💬 求人チャット | 上記以外（`job_src_row` / `candidate_id`） | `candidates(...)` |

- 定義は **`CHAT_FETCH_GROUPS` の1箇所だけ**。取得する列を足す・変えるときはここを直す
- `_fetchChatRooms(key)` が ①詳細つき → ②embedなし → ③絞り込みもembedも無しの最小
  の順に読み直す。**どこかが失敗しても一覧が空になることはない**
- 取得に失敗した系統は `_chatFetchDegraded` に入り、`_chatDegradedNotice()` が
  一覧の上に⚠️の注意書きを出す（黙って壊れないように）
- 2系統の結果は `_mergeChatRoomRows()` で `chatRooms` 1本にまとめる。
  以降の処理（未読集計・絞り込み・表示）は今まで通り `chatRooms` を見る

> ⚠️ **過去の事故**: ver.20260814.143 で `candidates(...)` に存在しない列名
> （`jp_level`／正しくは `jp_language_level`）を書いたところ、chat_rooms の取得が
> まるごと400エラーになり、**チャットが1件も出なくなった**。
> `node --check` は列名の間違いを検出できないので、
> **embed に列を足したら必ず Supabase MCP の `execute_sql` で
> `information_schema.columns` を引いて実在を確かめること。**
>
> ```sql
> select column_name from information_schema.columns
> where table_name='candidates' and column_name in ('足したい列名');
> ```

- **KMTドライブのリンク**: そのチャットの「元の欄」に書き戻す。人材＝`workers.drive_url_kmt`／
  求人＝`job_progress.drive_url_kmt`（求人進捗 情報タブ）／候補者＝`candidates.drive_url_kmt`（候補者情報タブ）。
  **どこに書くかは `_chatDriveTarget()` の1箇所だけ。** こうしておくと、画面の欄とチャットの
  「📁 KMT」ボタンが必ず同じものを指す（片方だけズレない）。
  大房側ドライブは人材チャット・企業チャットだけ（求人・候補者はKMT側のみ）
- **メンション**: `chat_messages.mentions`（jsonb の名前配列）。`isMentionForMe()` が部分一致で判定し、
  一覧の @バッジ・「@メンション」絞り込み・ブラウザ通知に使われる。
  返信（↩インライン／🧵スレッド）は `_withReplyMention()` で相手の名前を mentions に自動追加する（自分宛は除外）
- **ピン留め**: `chat_pins` テーブル。`room_id` が入っていれば個別チャットのピン、
  無ければ企業カードのピン（`_chatPinnedCompanyIds` / `_chatPinnedRoomIds`）。
  個別ピンは企業内リストで先頭に並び、`_prependPinnedRoomSection()` が一覧最上部にも表示する
- **添付**: `chat_messages.attachments`（`[{url,name,mimeType,size}]`）。実体は GAS 経由で Google Drive にアップロード。
  画像（`isChatImageAttachment()` が true）は `driveImageUrl(url, 800)` でサムネイル化してチャット内に直接表示、
  それ以外はファイルリンク。表示に失敗したら `_chatImgFallback()` がリンクに戻す
- 入力欄での画像貼り付けは `initChatPasteImage()`（document の paste を拾って `addChatFiles()` に渡す）

### 大房行政書士法人 案件システムとの連携

KMT → 大房の「📥 受信トレイ」に案件依頼を直接入れる仕組み。

- 大房は**別のSupabaseプロジェクト**（`ehwlgbwpycglmopiqyty`）。受信トレイの実体は
  **`intake_requests` テーブル**で、`status='pending'` が未確認として並ぶ
- 従来は KMT → 依頼スプレッドシート → 大房のGAS巡回 → `intake_requests` だったので反映が遅かった。
  いまは `ofusaFetch()` で同じ行を直接 INSERT している（スプレッドシート送信も従来どおり残してある）
- 接続先は `OFUSA_SB_URL` / `OFUSA_SB_KEY`（大房の公開ページに埋まっているのと同じ publishable キー）。
  `OFUSA_AGENCY_NAME` は大房の `agency_master` に登録されている KMT の機関名
- **書き込むテーブルは `intake_requests` だけ。** 大房の他のテーブルには触らない
- 行を作るのは **`ofusaIntakeRow(f)` の1箇所だけ**。決定報告も内定後の発注書もここを通す。
  列の一覧は `OFUSA_INTAKE_MAP` に日本語の対応表がある
- **送信できるのは決定報告の「他社支援（紹介）」と「企業単独（人材紹介）」だけ。**
  KMT案件は既存の「スプレッドシートに送信」の経路があるので、大房への直接送信は付けていない
  （重複対策は未設定。KMT案件も直送に寄せるときはここを見直す）
- 2つのフォームの追加項目は `ofusaSendSectionHtml(prefix, opt)` / `ofusaSectionValues(prefix)` で共通化。
  行の組み立ては `buildOfusaRowsFromOther()` / `buildOfusaRowsFromSolo()`
- 他社支援は人材名が複数行のことがあるので、**1名につき1行**を作って配列でINSERTする（大房は1案件＝1名）
- `delivery_method` は画面で選んだ交付方法をそのまま渡す。認定を電子交付に倒すのは大房側が案件登録時にやる

> ⚠️ **`ofusaIntakeRow` に列を足すときは、必ず大房DBの `information_schema.columns` で実在を確かめること。**
> 存在しない列が1つ混ざるだけで INSERT がまるごと400で落ちる（chat_rooms のときと同じ事故になる）。
>
> ```sql
> -- project_id は ehwlgbwpycglmopiqyty
> select column_name from information_schema.columns
> where table_name='intake_requests' and column_name in ('足したい列名');
> ```

送信済みの記録は3か所に残す（どれも二重送信のチェックと「送信済」表示に使う）:

| 送信元 | 記録先 |
|---|---|
| 発注書（内定後） | `candidate_job_progress.ofusa_intake_id` / `ofusa_sent_at` |
| 求人案件 | `job_progress.ofusa_intake_id` / `ofusa_sent_at` / `ofusa_flag` |

- **「大房側の案件」の判定は `isOfusaJob(r)` の1箇所だけ。** 会社名の文字列一致では絶対に判定しない
  （大房側の実データに `株式会社KMT` / `株式会社ＫＭＴ` / `KMT(82)` のような表記ゆれがある）。
  `ofusa_flag` が唯一の正で、発注書を送った案件は自動で立つ
- 求人詳細の「🏢 大房側の案件」チェックと「📱 求人LINEグループ名」は
  **区分が【紹介案件】のときだけ出す**。開閉するidは `JP_INTRO_ONLY_BLOCKS` の1箇所にまとめてあり、
  `toggleJpOrgPicker()` が区分の変更で連動させる（登録支援機関検索もこの配列に入っている）
- 発注書の対象ステータスは `OFUSA_ORDER_STATUSES`（内定・決定）。増やすときはここだけ直す

### 求人案件のモーダル（3タブ）

| タブ | 中身 | 保存 |
|---|---|---|
| 📋 求人進捗 情報 | 進捗・区分・コード・所属機関・担当・優先度・**手数料管理**・KMT社内・コメント | `saveJobProgress` |
| 📄 求人票＋よくある質問 | 求人ノート → 求人票の項目編集（📥テキストから読み込む付き）→ よくある質問 | `saveJobSheet` / `saveJobFaq` |
| 👥 候補者リスト | `candidate_job_progress` の紐付け | 行ごとに即保存 |

- **タブごとに保存する範囲を分けてある。** 開いていないタブの欄を空で上書きしないため、
  `saveJobProgress` は求人票の列に触らない／`saveJobSheet` は `JOB_SHEET_FIELDS` の列だけを書く
- `openJobProgressDetail(src_row, focusId)` の第2引数で、開いたあと特定の項目まで飛ばせる。
  **どのタブの項目かは `_jpTabOfField()` の1箇所だけで判定する**（`jsf_` 始まり＝求人票タブ、
  `cands`＝候補者リスト、それ以外＝情報タブ）。`_jpFocusField` がタブを開いてから黄色く光らせる。
  タブの中身が描かれる前に呼ばれても数回だけやり直すので、呼ぶ側で待たなくてよい
- 求人票の項目は **`JOB_SHEET_GROUPS` の1箇所だけ**で定義する。画面・保存・テキスト読み込みが同時に追従する。
  **列名は求人票（`buildJobOrderSheet`）が読む名前とそろえてある**ので、足すときも同じ名前にすること
  - `hint` は**単位**（円・日・分・名 …）。項目名の横に「（円）」と出し、
    読み込み時に値の末尾から落とすのに使う。**placeholder には使わない**
    （欄の中に薄く「円」と出て、値が入っているように見えてしまうため。実際に直した）
  - 入力例を出したいときは `ph`。1行いっぱいに広げたいときは `full:true`
  - 2列に流れるので、**対になる項目は隣り合う順に並べる**（始業／終業、手当① 名称／金額 など）
- 「📥 他社の求人票から読み込む」は3段構え。**必ず一覧で見せて確認してから反映する**
  1. **表の貼り付けが本命**。`initJobSheetPasteTable()` が paste を拾い、
     `text/html` に `<table>` があれば `_jsHtmlTableToLines()` で「ラベル\t値」に直す。
     Excel/Word/スプレッドシートは**文字だけコピーするとラベルの塊と値の塊が別行になり対応が壊れる**ので、
     表のまま貼ってもらうのが唯一の確実な方法。ここを崩さないこと
  2. `_jsSplitLine` … タブ区切り → 全角コロン等 → 半角コロン（ラベルに空白なし）→ 空白区切り の順。
     「始業 8:00」の時刻と区切りのコロンを取り違えないための順番
  3. `_jsMatchField` … 完全一致 > ラベルがキーワードを含む(3字以上) > その逆(4字以上) で score を付ける。
     **双方向の部分一致で拾うと無関係な行が別項目に入る**（実際に事故った）。空白区切りの行は score 100 のときだけ採用
- 当てはめは一覧の `<select>`（`jsImportRetarget`）で選び直せる。拾えなかった行も手で入れ先を指定できる

### 求人票の写真（作業場・寮等）

- 実体は `job_progress.photos`（`[{url,name,at,by}]`）。実ファイルは
  **GAS経由で Google Drive の共有フォルダ**に入れる（チャットの添付と同じ経路）。
  置き場は `JOB_PHOTO_DRIVE_URL` の1箇所だけ
- 最大 `JOB_PHOTO_MAX`(8) 枚。求人票では `JOB_PHOTO_PER_PAGE`(4) 枚ずつ並べるので**写真ページは最大2ページ**
- 貼る欄は `jobPhotoBoxHtml()` / `renderJobPhotos()` / `addJobPhotoFiles()`。
  **社内の求人票タブと外部の記入フォームで同じ部品を使う**（`_jobPhotos` が共用の入れ物）
- 求人票のページは `_jobPhotoPagesHtml(photos)` が作り、`buildJobOrderSheet` の
  page1 + page2 の後ろに足す。**ページ1に (11)(12) の空欄は置かない**
  （よくある質問はページ2の一覧、写真は3ページ目以降の専用ページ）
- (10) 確認事項・追加事項は `job_progress.order_memo`。未記入なら
  `_jpToFlyerJob` が `progress_note`（案件担当のコメント）にフォールバックする

### 求人票の外部記入リンク

企業（先方）に求人票の項目を直接書いてもらう仕組み。

- 実体は **`job_sheet_links`**。1リンク＝1レコードで、`token` で開く
- **`#jobform=<token>` でログインなしに開ける**（`initLogin` の先頭で分岐）
- 一時保存は `draft` / `draft_at`、送信は `answers` / `answered_at` / `answered_by`。
  送信すると `job_progress` の該当列にそのまま反映する
- **どの項目が企業回答で、どれを社内で直したかは `job_progress.sheet_meta` の1箇所だけに残す**

  ```js
  sheet_meta[col] = { ext:{v, at}, int:{v, at, by} }   // ext=企業回答 / int=社内編集
  ```

  表示は `_sheetFieldNote(r, col)` が作る（入力欄の下の1行）。
  `ext` は社内で直しても消さない。「企業回答 → 内部編集あり」と並べて出すため
- `saveJobSheet` は**値が変わった項目だけ** `int` を書く（開いただけでは記録しない）
- 外部フォームの入力欄idは社内タブと同じ `jsf_*`（写真の部品も共用しているので**ずらさないこと**）
- **スマホ対応は `@media(max-width:640px)` の1箇所**（`#candFormOverlay` / `#jobFormOverlay`）。
  中の欄は inline style の grid なので `[style*="1fr 1fr"]` を1列に倒す（`!important` が要る）。
  写真のサムネイル（`minmax(...)`）は対象外にしてある。
  入力欄は 16px にする（iOSで勝手に拡大しないため）。外枠の余白は `.extform-head` / `.extform-card`
- **「読み込む」機能は社内タブだけ。** 外部フォームには付けていない（そのまま欄に記入してもらう）
- リンクは `open` → `closed`（🔒 無効に）→ 削除の順。**削除できるのは `closed` のものだけ**
  （`deleteJobSheetLink`）。`closed` になったリンクはコピーボタンも出さない
- **有効（`open`）なリンクは1案件に1本まで。** 何本も出すと企業がどれに書いたか分からなくなり、
  回答が上書きし合うため。判定は `_openJobSheetLink()` の1箇所。
  `createJobSheetLink` は毎回 `loadJobSheetLinks` で取り直してから見る
  （別の人が先に発行しているかもしれないので、ボタンのdisabledだけに頼らない）
- アラートは2つ。`extAnswered`（回答が届いて `seen_at` が無い）と
  `linkStale`（`JOB_LINK_STALE_DAYS`=30日以上 未回答）。
  「✅ 確認済みにする」で `seen_at` が入り、アラートから消える

### 労働局指定項目・求人管理簿

- 定義は **`JP_LABOR_INFO_FIELDS`（情報タブぶん）＋ `JOB_SHEET_GROUPS` の `req:true`（求人票タブぶん）**。
  この2つを合わせたものが `JP_LABOR_FIELDS`。項目を足すときはどちらか片方だけを直す
- 欄の下の赤い注記は `_laborNote()`。**＊マークと注記はセットで付ける**
- 未記入の欄が薄赤（`.field-empty`）になるのは `JP_MARK_FIELDS()` に入っているものだけ。
  情報タブ・求人票タブの両方の労働局指定項目を見るので、**`req:true` を付ければ薄赤も付いてくる**
  （求人票タブは `switchJpTab('sheet')` の中で `initJpEmptyMarks()` を呼んで反映している）
- **未記入でも保存できる**。`saveJobProgress` / `saveJobSheet` が confirm で知らせるだけ
- アラートは `missingJpLaborFields(r)` の1箇所。候補者リストが空の案件は
  「求職者（候補者リスト）」も未入力として数える（`allCjpLinks` を見るので、
  `loadJobProgress` で必ず取っておくこと）

#### 国籍のまとまりの並び順

- 求人管理リストのグループ（`_jpBuildGroups`）と絞り込みチップ（`_jpRenderFilterPanel`）は
  **どちらも `_jpNatRank()` の1箇所**で並び順を決める。`JP_NAT_NONE`（（国籍指定なし））は
  国を問わず出せる案件なので必ず先頭、以降は `JP_NAT_ORDER` の順、それ以外は末尾に五十音順
- 「国籍の指定なし」の文字列は `JP_NAT_NONE` を使う（ベタ書きしない）

#### 労働局 求人管理簿（`_jpView === 'ledger'`）

- 求人管理の右上「表示」で `KMT管理リスト` ⇄ `労働局 求人管理簿` を切り替える（`setJobView`）
- 様式そのままの並び（①〜⑪＋備考）。**列順は様式なので勝手に変えない**
- ⑪ 職業紹介の取扱状況は `candidate_job_progress` から作る（`_ledgerCandRows`）。
  候補者が複数いれば行が分かれ、左側は rowspan でまとめる
- **③ 連絡担当者は `client_contact_name`（企業側の担当者）。**
  所属機関の `main_staff`（社内のメイン担当）ではないので、そこから取らないこと
- 各行の「✏️ 編集」で `openJobProgressDetail(src_row)` を開き、その場で直せる
- **セルをクリックするとその項目の編集画面が直接開く。** 対応表は **`JP_LEDGER_EDIT` の1箇所だけ**
  （管理簿の列 → 入力欄id）。クリックの入口は `ledgerEditField(src_row, col)`
  - 当たり判定には `class="jp-led-hit" data-led="<col>"` を付ける。ホバー色はこのクラスのCSS1箇所
  - **文字を選んでいる間は開かない**（`window.getSelection()` を見ている）。
    管理簿はコピーして使うことがあるので、選択を潰さないこと
  - ①②③ と ⑦ は1セルに複数項目が入っているので、**セルではなく中の `<div>` ごと**に当たり判定を付ける
  - ⑪ の欄は `cands` を渡して候補者リストのタブを開くだけ（個別の入力欄には飛ばさない）
  - ✏️ 編集ボタンは `event.stopPropagation()` を入れてある（セルのクリックと二重に発火しないように）
- 日付は令和表記（`_ledgerYmd`）、賃金は `_ledgerWage`。未記入は赤字の「未記入」
- 印刷（A4横）と Excel出力あり（`printJobLedger` / `exportJobLedgerExcel`）。
  **いま絞り込んでいる行だけ**を出す
- アラートの算出元は **`missingJpLaborFields(r)` の1箇所だけ**。
  求人・候補者アラートの「労働局指定項目（求人受理票）が未入力の求人案件」に出る
- 求人票の表示は `openJobSheetPreview(srcRow)`（求人票タブのフッター「📄 求人票を表示」）。
  **保存前の入力も反映する**ため、保存済みの行に画面の `jsf_*` と FAQ を重ねてから組み立てる
- 印刷・PDF・Excel は求人票作成の画面と同じ関数を使い回す。対象は `_flyerPreviewEl()` が決める。
  **モーダルが表示中かどうかで切り替える**こと（閉じてもDOMに残るので、
  id の有無だけで判定すると求人票作成の画面で古い内容が印刷されてしまう）
- 求人オーダー票（`buildJobOrderSheet`）の **(9) 求人受理票** が同じ項目を印刷する。
  (8) の下に入るので、以降は (10)確認事項／(11)よくある質問／(12)写真。
  **番号を足すときは (9) の位置と以降の採番をまとめて直すこと**

### 候補者の労働局指定項目（求職票）

- 定義は **`CAND_LABOR_FIELDS` の1箇所だけ**（氏名・生年月日・本国住所・希望職種・受付年月日・有効期間）。
  `or:'cf_jp_address'` は「日本住所が入っていればOK」の意味
- **見出しでまとめない。** ＊マーク（`_starMark()`）と欄の下の赤い注記（`_laborNote()`）を
  **項目ごとにセットで**付ける
- 未記入は薄赤（`updateCandEmptyMarks` / `initCandEmptyMarks`）。
  **未記入でも保存できる**＝ `saveCandidate` は confirm で知らせるだけ（止めない）
- ステータスは空（`－（未設定）`）にもできる。新規登録のときだけ `新規登録` を初期選択する
- **現在地（`domestic_overseas`）で日本住所の必須が変わる。** 選択肢は `CAND_LOCATIONS` の1箇所
  （日本国内／海外。古い `国内` `国外` は `_candLocationNorm()` が読み替える）
  - 日本国内 → 本国住所＋日本住所（在留者）の両方が必須
  - 海外 → 本国住所だけ必須。「いまいる国」（`overseas_country`）の欄が出る
  - 連動は `onCandLocationChange()` の1箇所（国の欄の出し入れ・日本住所の＊と注記・薄赤）。
    必須の判定は `CAND_LABOR_FIELDS` の `when` に書く

### 候補者のステータス

- **選択肢・並び・色・説明は `CANDIDATE_STATUSES` の1箇所だけ。**
  新規登録 → 提案待ち → 企業面接待ち → 結果待ち → 内定 ／ 見送り・辞退
- 一覧のバッジ（`candStatusBadge`）・絞り込み・モーダルの選択欄・タイトルは全部ここから作る
- **以前のステータス（選考中・入国待ち・配属済 など）が入っている候補者は、
  その値も選択肢に足して残す**（`candidateStatusOptionsHtml`）。保存で勝手に書き換えない
- ステータス欄は候補者情報タブの**いちばん上**（氏名の上）。変えると
  `_cndSyncTitle()` がモーダルのタイトル（`📋 候補者詳細 — <バッジ>`）と説明文も更新する
- 人材担当（`cf_staff`）は**チームが `GLT` のユーザーから選ぶ**（`gltStaffOptionsHtml`）。
  名簿が未読込なら `_cndFillStaffSelect()` が取り直して作り直す。
  いまの値が一覧に無いときは、その値も選択肢に残す

### 候補者の外部記入リンク（履歴書作成リンク）

- 実体は **`candidate_form_links`**。
  **`#candform=<token>` でログインなしに開ける**（`initLogin` の先頭で分岐）
- **本人へ渡すリンクは1種類だけ＝「履歴書作成リンク」（`CAND_FORM_KIND`）。**
  発行できるのは **【📝 履歴書作成】タブの1箇所だけ**（候補者情報タブには置かない）。
  候補者情報の項目は重複ぶん（`CAND_SYNC_FIELDS`）が自動反映されるので、リンクを分ける必要がない
- `kind:'info'`（旧・応募者登録リンク）は **すでに渡したぶんを開く／無効にするためだけに残してある**。
  `CAND_FORM_KINDS.info.retired = true` が目印で、パネルの一覧では「応募者登録リンク（旧）」と出る。
  **新しく発行することはない**（`createCandFormLink` は kind を無視して `CAND_FORM_KIND` にする）
- **ルールは求人票の外部記入リンクと同じ。** 有効なリンクは**候補者につき1本まで**
  （種類は問わない。判定は `_openCandFormLink()`）、発行時に確認、`closed` にしたものだけ削除できる
- 文言は **`CAND_FORM_KINDS` の1箇所**（パネルのid・案内文もここ）。パネルは1枚だけ
  （`renderCandLinkPanels()` → `cndResumeLinkPanel`。旧リンクも同じ一覧に並べる）
- 外部フォームの中身は**社内と同じ部品を使い回す**（項目の定義は1箇所のまま）
  - 履歴書作成＝`resumeFormFieldsHtml({external:true})`（1・2ページ目）。
    **社内用の欄は `ext` で出さない**＝ 履歴書の見た目（KMTロゴ・靴サイズの「履歴書に出す」）／
    🗣️ 一次面接／📝 担当者記入欄（担当者の感想・企業面接メモ）
  - **ご記入者と氏名は別物**なので、それぞれ下に説明を出す。
    ご本人が書いているときは「記入者名と同じ」（`rfNameSameAsBy`）にチェックすると
    `cndFormBy` の値が `rf_name` に入り、チェック中は氏名を直接いじれない
    （`onCandFormNameSame` / `onCandFormByInput` / `_cndFormCopyBy` の1組）
  - **職歴は空の1欄目を最初から出す**（`_ensureOneWorkHistoryRow()`）。
    2件目からは本人が［＋ 追加］で増やす。**何も書かれなかった行は送信時に落とす**
    （社内側は今までどおり0欄から。増やすのは外部フォームだけ）
  - 旧・応募者登録＝`_cndSecBasic` + `_cndSecNat` + `_cndSecJp` + `_cndSecWish`
    （ステータス・企業面接・担当・求人への紐付けは社内用なので出さない）
- 送信時、履歴書のほうは `interview1` / `resume_opts` / `staff_impression` / `interview_memo` を**送らない**
  （外部フォームに出していない項目を空で上書きしないため）。
  **`ext` で欄を消したら、`_cndFormValues()` からも必ず外すこと**
- **外部フォームには「＊自動反映」の青い注記を出さない**（本人には関係がないため）
- **登録前の候補者にもリンクだけ先に渡せる**（新規登録モーダルの【📝 履歴書作成】タブにある
  「＋ 新規リンクのみ発行」＝ `createCandFormLinkNew()`）。`candidates.name` は必須なので
  `CAND_LINK_PLACEHOLDER` の行を作り、外部フォームでは氏名を空欄で出す（本人が書く）。
  **`CAND_LINK_PLACEHOLDER` の文字列は変えない**（既存の行の判定に使っているため）
- 発行の履歴は候補者画面の「🔗 リンク発行履歴」タブ（`loadCandLinkLog` / `renderCandLinkLog`）。
  いつ・だれが・どこへ・回答の有無が並ぶ。「記入待ち（未登録）だけ」で絞れる

### 🗑 削除（求人案件・候補者）

- **消す前に必ず2回たずねる**（`_confirmDeleteTwice()` の1箇所）。
  1回目＝何を消すか、2回目＝取り消せない確認
- 実体は残す**論理削除**（`is_deleted` / `deleted_at` / `deleted_by`）。
  取得は `is_deleted=not.eq.true` を付ける。**新しく取得口を足すときもこれを付けること**
- 入口は `deleteCandidate()` / `deleteJobProgress(srcRow)`。
  ボタンはフッターの右端（フッター配置の統一ルールどおり）

### 候補者の「🔗 求人への紐付け」

- **求人は2系統あるので、どちらも出す**（片方だけだと登録済みの求人が出てこない）

  | 出どころ | 紐付けの実体 | 選択肢の値 |
  |---|---|---|
  | 求人案件（求人管理 `job_progress`） | `candidate_job_progress`（求人詳細の👥候補者リストと同じ） | `jp:<src_row>` |
  | 求人登録（`job_openings`） | `candidate_job_openings`（従来ぶん） | `jo:<id>` |

- **`jp:` / `jo:` の接頭辞が唯一の判別**（`addJobToCandidate` / `removeJobFromCandidate`）。
  `jp:` で足したものは求人側の候補者リストにもそのまま出る（同じテーブルなので二重管理にならない）
- **一覧から求人を選んだ時点で紐付ける**（`chooseCjPick` → `addJobToCandidate`）。
  別の［追加］ボタンは置かない（選んだのに反映されない、が起きるため）。取り消しは一覧の［削除］
- 逆に求人側の【👥 候補者リスト】（`jpAddCandidate`）で足したものも、同じ行なので
  候補者側の「求人への紐付け」にそのまま出る。**どちらの画面も開くたびに取り直す**
  （`loadCandidateJobLinks` / `loadJpCandidates`）ので、片方で足した分は必ず反映される
- 絞り込みは **`CJ_PICK_KINDS` の1箇所**（全求人／KMT案件／紹介案件）。
  **KMT案件と紹介案件の判定は `kmt_flag` だけ**（求人管理の分け方と同じ。会社名では判定しない）。
  求人登録には区分が無いので、（全求人）のときだけ出す
- 検索は `renderCjPickOptions()` の1箇所（コード・企業名・分野・職種・就業場所・進捗・営業担当）
- **求人を選ぶ欄は `<select>` ではなく自前のパネル**（`toggleCjPickPanel` / `chooseCjPick`）。
  検索欄と絞り込みチップを**一覧の一番上（開いたパネルの中）**に置くため
  （`<select>` の中には入れられない）。選んだ値は隠しの `#cjAddSelect` に入れるので、
  追加・削除の処理は今までどおりそこを見る。
  モーダルの下のほうにある欄なので、下に入りきらないときは上に開く

### 候補者モーダル（2タブ）と履歴書

| タブ | 中身 | 保存 |
|---|---|---|
| 📋 候補者情報 | これまでの項目（労働局指定項目・基本情報・希望条件・求人への紐付け） | `saveCandidate` |
| 📝 履歴書作成 | メモ（`candidate_notes`）→ 履歴書の項目 → 印刷／PDF／Excel | `saveResume` |

- 2つのタブは**両方DOMに置いたまま** `display` で切り替える（写すために両方必要）

#### 🔄 2つのタブで重複している項目（自動反映）

- **どちらのタブも同じ `candidates` の行を見る。** 同じ列を出している欄（`cf_xxx` と `rf_xxx`）は
  **打った先からもう片方へ写る**。どちらのタブで直しても、本人が外部リンクから送ってきても、両方そろう
- **重複している項目の一覧は `CAND_SYNC_FIELDS` の1箇所だけ**（いま24項目）。
  欄の下に出る「＊履歴書作成から自動反映」の注記（`_candSyncNote()` / `_applyCandSyncNotes()`）も
  ここから作るので、**足せば注記まで付いてくる**
- **注記を出すのは【📋 候補者情報】タブ（`cf_`）だけ。** 履歴書作成タブと履歴書作成の画面には出さない
  （入れる本人が元なので不要。画面がうるさくなる）。
  **項目名より目立たせない**＝ 8px・細字・薄い水色（`#a8d8ef`）
- 写す本体は `_cndCopyField(id)`。**候補者モーダルの中だけ**を見る
  （履歴書作成の画面を開いていると `rf_*` が二重になるため、`getElementById` では引かない）。
  選択肢の並びが違う select は、同じ選択肢があるときだけ写す
- 入力のたびの反映は `_bindCandSyncFields()`（document に1回だけ付ける委任リスナー）。
  タブを移るときは `_syncCandidateResumeFields(from)` がまとめて写す
- **食い違ったときは履歴書作成が残る**＝ `saveCandidateAll()` が候補者情報 → 履歴書の順に書く。
  履歴書タブの読み込み（`_cndLoadResumeTab`）もDBの最新を取り直してから `rf` → `cf` に写す
- 似ているが**別の列**なので写らないもの（このままにする）:
  実務経験 `experience` ↔ 職歴 `work_history`・退職理由 `resignation_note` ／
  メモ `memo` ↔ 企業面接メモ `interview_memo` ／
  評価コメント `jp_eval_note` ↔ 本人からの質問 `candidate_questions`
- **外部フォームにも出さない**（本人には関係がないため）
- **証明写真（顔写真）は `resumePhotoBoxHtml()` / `addResumePhotoFile()` の1箇所**。
  URLを貼るのではなく、その場でファイルを選ぶ。実体は人材資料と同じ GAS → Google Drive
  （置き場は `_resumeDocDriveUrl()`）。値は今までどおり `candidates.photo_url`＝隠しの `rf_photo_url` に入るので、
  保存・読み込みは触らなくてよい。**履歴書に出すときは必ず `driveImageUrl()` を通す**
  （Driveの共有URLはそのままでは画像として表示できない）
- 履歴書の入力項目は **`resumeFormFieldsHtml()` の1箇所**。
  履歴書作成の画面（`buildResumeScreen`）とこのタブで共用する。idは必ず `rf_` 始まり
- **履歴書作成の画面を開いている間は、このタブは案内文だけにする**（`rf_*` のidが二重になるため）
- 新規候補者（未保存）のときも案内文だけ。メモも履歴書も保存後から
- 出力は `_resumeExportHtml()` が元になる。画面ではプレビューをそのまま、
  モーダルでは入力中の値から `buildResumeHTML()` で組み立てる（モーダルにプレビュー欄が無いため）。
  PDFはブラウザの印刷ダイアログ任せ（求人票と同じ）
- 履歴書に「出す／出さない」を選べる項目は `candidates.resume_opts`（jsonb）。
  いまは KMTロゴ（`_resumeShowLogo()`）と靴サイズ（`_resumeShowShoe()`）。
  **既定はどちらも出す**＝ `false` がはっきり入っているときだけ隠す。
  チェック欄は `resumeFormFieldsHtml()` の中（ロゴはいちばん上の「🏢 履歴書の見た目」）
- 履歴書のロゴはヘッダーの `<img class="logo-mark">` を `_kmtLogoSrc()` で流用する
  （画像を二重に持たない。求人票の透かしと同じやり方）。
  置き場所は `.resume-head`（左＝ロゴ／右＝日付）の1行
- **印刷・PDF・Excel に出すページは `RESUME_PAGES` の1箇所**で定義する
  （`main` / `docs`＝`.resume-doc-page` / `iv1`＝`.resume-iv1-page`）。
  選択は `_resumePageSel`、出力時のふるい落としは `_resumeFilterPages()`。
  **先頭に残ったページの改ページは外す**（白紙が1枚できるため）。
  画面のプレビューは `_applyResumePageSel()` が display で隠す
  （`previewResume()` は毎回描き直すので、最後に必ず呼ぶこと）

#### 📎 人材資料保存（履歴書の2ページ目）

- 在留カード・資格証などの写真を貼る台紙。**枠の定義は `RESUME_DOC_SLOTS` ＋ `RESUME_DOC_OTHER` の1箇所だけ**
- 実体は `candidates.resume_docs`（jsonb）＝ `{ キー: [{url,name,at,by}] }`。
  読み替えは **`_resumeDocList()` の1箇所**（配列でない古い形・多すぎる分の切り捨てもここ）
- 枠は様式どおり。上＝在留カード（表/裏）、真ん中＝評価調書／技能試験、下＝その他。
  上と真ん中は1枠 `RESUME_DOC_SLOT_MAX`(4) 枚まで（在留カードは更新のたびに増えるため）、
  その他は `RESUME_DOC_OTHER_MAX`(6) 枠。**写真が無い枠も空の枠のまま出す**（あとから貼れるように）
- 実ファイルは求人票の写真と同じ GAS → Google Drive。置き場は `_resumeDocDriveUrl()` の1箇所
  （`JOB_PHOTO_DRIVE_URL` はこの下で定義されるので、const ではなく関数にしてある）
- ページを組み立てるのは `_resumeDocPageHtml()`。`buildResumeHTML` の最後に足すので、
  画面・印刷・PDF・Excel のどれにも同じものが出る

#### 🗣️ 一次面接（履歴書の3ページ目）

- 質問（項目名）は**全社共通のマスタ `interview_questions`**（`label` / `sort_order` / `is_active`）。
  初回アクセス時に `INTERVIEW1_SEED` の14問を自動投入する
- 回答は候補者ごとに `candidates.interview1`（jsonb）＝ `{ company_name, answers:{ 質問id: 回答 } }`
- **回答は誰でも入れられるが、質問の書き替え・追加・削除は管理者だけ。**
  判定は `isAdminUser()` / `requireAdmin()` の1箇所を使う（UIも管理者のときだけ出す）
- **質問の削除は行を消さず `is_active=false`**（入力済みの回答を残すため）
- ページは `_interview1PageHtml()`。左＝前半／右＝後半の2列で、番号は表示時に振る
  （マスタの `label` に番号は入れない）
- 候補者モーダルの【履歴書作成】タブのフッターは **「📄 履歴書を表示」だけ**。
  印刷・PDF・Excel はその画面（`openResumePreview`）から出す。
  出力の向き先は `_resumeExportHtml()` が決める（表示中のモーダル → 画面のプレビュー → 入力値から組み立て）

### 📝 フリーノート（求人案件・候補者で共通）

- 接続先は **`NOTE_SCOPES` の1箇所だけ**（`job` → `job_notes` / `candidate` → `candidate_notes`）。
  `loadNotes(scope)` / `renderNotes(scope)` / `addNote(scope)` / `saveNote(scope,id)` / `deleteNote(scope,id)`
- `loadJobNotes()` などの旧い名前は `'job'` を渡すだけの包みとして残してある
- 書式ツール（`jnExec` / `jnInsertTable` / `markJobNoteDirty`）はノートidで動くので scope 不要
- **添付ファイル**: `job_notes.attachments` / `candidate_notes.attachments`（jsonb の
  `[{url,name,size,mimeType,at,by}]`）。実体はチャットの添付と同じ Drive フォルダー
  （`CHAT_DRIVE_URL_DEFAULT`／設定の `kmt_chat_only_drive_url`）に GAS 経由で入れる。
  表示は**リンクだけ**（`_noteAttachHtml`）。追加・削除したら `saveNote` まで自動で走らせる
  （アップロードしたのに保存し忘れる事故を防ぐため）

### 📅 支援スケジュール

- 種別の定義は **`SCHEDULE_EVENT_TYPES` の1箇所だけ**（キーは既存データと合わせるので変えない）。
  **色は種別ごとに必ず別の色にする**（同じ色が2つあると一覧で見分けがつかない）
- カレンダーの上の凡例は `renderScheduleLegend()`。**表示中の期間（週／月）の件数を種別ごとに出す**。
  数え方は `_schedPassCommon()`（企業・人材・担当・フリーワード）＋ `_schedInRange()` で、
  **種別の絞り込みだけは外して数える**（何件あるか分からなくなるため）。
  凡例を押すと `setScheduleTypeFilter()` でその種別だけに絞れ、
  下に `renderScheduleTypeList()` が「その種別の人材・企業」の一覧を出す（行を押すとその予定が開く）
- マイページの支援カレンダーも同じ数え方だが、**自分がメイン／サブ担当のぶんだけ**
- 「📍 今日」（`schedCalThis`）は期間を今日に戻すだけでなく、`_schedScrollToToday()` が
  **今日の列（月表示は `.sched-today-cell`）まで横スクロールして光らせる**。
  すでに今日の週を見ていると押しても変化が無く「効いていない」ように見えるため、合図は必ず出す
- **サイドバーの赤いバッジは出さない**（何の件数か分かりにくかったため。
  `updateSupportScheduleBadge` / `refreshSupportScheduleBadge` は空の関数として残してある）

### 求人ヒアリングフォーム

- `#hearing` のリンクで**ログインなしで開ける**（`initLogin` の先頭で分岐）。外に渡せる
- 送ると `job_progress` に `progress='詳細確認中'` / `source='hearing'` で登録される。
  **別テーブルを作らない**（＝「直でKMTシステムに反映」がそのまま成立する）
- 精査もれは求人・候補者アラートの「ヒアリングフォームから届いた未精査の案件」に出る
- **求人管理のツールバーからはボタンを外した**（企業への記入リンクは案件ごとに発行する運用にしたため）。
  いまフォームを開けるのは `#hearing` のリンクだけ。`openHearingForm()` / `copyHearingLink()` は残してある
- **LINEグループはKMT側では作れない**（LINE基盤は大房側にしかない）。
  `requestJobLineGroup()` が求人チャットに作成依頼を投稿し、作った人が
  求人案件の `line_group_name` に書き戻す運用。募集中なのに空の**紹介案件**はアラートに出る
  （LINEグループは紹介案件だけの話なので、KMT案件はアラートにも入れない）

### 外国人材アラート（サイドバーと画面上部バナー）

**`computeGlobalWorkerAlerts(list)` が唯一の算出元。判定ロジックをここ以外に書かない。**

- 対象は 🚨申請忘れ警告 / 📋更新リマインド / 🪪在留期限の更新待ち / 🏗️建設業就労管理システム報告 の4種
- 🪪在留期限の更新待ち: apply_status が APPLY_DONE_STATUSES（審査完了以降）になってから
  VISA_UPDATE_WAIT_DAYS(30)日以上たっても、visa_expiry が空欄 or VISA_UPDATE_SOON_DAYS(120)日以内のまま。
  経過日数の起点は workers.apply_status_changed_at（saveWorker がステータス変更時に記録）→ apply_date → request_date
- サイドバーのバッジ（`updateSidebarAlerts`）と外国人材ページのバナー（`renderWorkerAlerts`）は
  同じ関数・同じデータソース（`_alertSourceWorkers()`）を使うので、必ず同じ数字になる
- これらは **一覧フィルターに追従しない**（全人材で数える）。
  退職報告アラートのように、一覧では非表示になる人材が対象のものがあるため
- 期限カード（特例期間中／1・2・3ヶ月以内）、脱退一時金、退職予定日超過、在留情報未入力は
  従来どおり一覧フィルターに追従する
- `allWorkers` は画面によって軽量 select で上書きされることがある。
  判定に必要な列が無い場合は `_alertSourceWorkers()` が専用取得分（`_sidebarAlertData`）にフォールバックする
- 在職ステータスが **辞退/キャンセル** と **支援機関変更済み** の人材は全アラートの対象外。判定は `isAlertExcludedWorker(w)` の1箇所だけ
  （`computeGlobalWorkerAlerts` / `computeMyWorkerAlerts` / 期限カードの一覧 から呼んでいる）。
  表記ゆれを拾うため「辞退」「キャンセル」の部分一致で見ている
- 10番「通算期間と過去の就労先」の**自動計算した5年満了日は、8番の `wf_tg2_expiry` に自動で入る**
  （`syncTokuteiExpiryToForm()` の1箇所。`renderTokuteiCalc` から毎回呼ぶので2つの欄がズレない）。
  モーダルを開いたときの反映はスナップショットより前に走るので、開いただけでは未保存あつかいにならない
- 🧾 通算期間の精査状況は `computeHistoryVerifyStats()` が算出元（`done` / `todo` の件数と両方の一覧）。
  外国人材ページは数字を押すとその一覧、マイページは `setMyHvTab()` で切り替える。
  **全員精査済みでも表示は残す**（精査済みの一覧を見られるようにするため）
- ⚠️【要確認】通算期間不明（`workers.history_unknown`）は `isHistoryUnknown(w)` の1箇所で判定し、
  一覧の「5年満了日」に `historyUnknownBadge()` を出す（人材一覧・更新状況リストの2か所）。
  アラート欄にも一覧が出る（`toggleHistoryUnknownList`）。
  **軽いSELECTでも読めるよう `WORKER_ALERT_SELECT_COLS` に列を入れてある**
- 必須項目の未入力アラートは `missingRequiredFields(w)` が唯一の算出元（パスポート期限・出入国の情報）。
  パスポート期限の必須判定は `isPassportExpiryRequired(w)`（申請種別が「認定」以外で必須）
- **判定条件を変えるときは `computeGlobalWorkerAlerts` だけを直せばよい**

### フッターのボタン配置（統一ルール）

`最終更新 → 💾保存 → （チャット開始）→ 閉じる │ 余白 │ 🗑削除`

## 8. 機能とコードの場所

| 機能 | 主な関数 |
|---|---|
| 所属機関一覧・詳細 | `renderTable` / `openCompanyModal` / `fillCoForm` / `getCoFormData` / `saveCompany` |
| 業種（複数選択） | `getCompanyIndustries` / `industryOptionList` / `renderIndustryChips` / `commitNewIndustry` |
| 外国人材詳細 | `openWorkerModal` / `buildWorkerModalBody` / `saveWorker` |
| 申請記録タブ | `renderApplyRecords` / `loadApplyRecords` / `saveApplyRecord` / `sortApplyRecords` |
| 求人進捗 | `buildJobsScreen` / `loadJobProgress` / `renderJobProgress` / `saveJobProgress` |
| 決定報告（KMT案件） | `buildKmtReportText` / `getDrOrgChange` / `copyKmtReport` / `submitDecisionReport` |
| 売上管理 | `renderSaleMonthTable` / `openSaleModal` / `saveSale` |
| チャット | `openChatRoom` / `loadChatMessages` / `sendChatMessage` / `_ensureWorkerRoomId` |

### データ構造のメモ

- `companies.industries`（jsonb）が業種の配列。旧 `industry`（単一）との互換は
  `getCompanyIndustries()` が吸収する。選択肢マスタは `industry_options` テーブル（全社共有）
- 申請記録は Googleシート直読み（gviz CSV）＋編集値を `apply_record_edits` に upsert
- 履歴書の保有資格は `candidates.licenses`（jsonb）。いまの形は
  `{ items:[{name, has}], other:'…' }` で、**項目名も候補者ごとに変えられる**（最大 `RESUME_LIC_MAX`=10・2列）。
  旧データ（`{career_up_card:true,…}` の決め打ち6項目）との互換は **`_licItems()` の1箇所**が吸収する。
  入力欄は `renderResumeLicInputs` / `addResumeLic` / `removeResumeLic` / `updateResumeLic`。
  **入力中に一覧を作り直さない**（`updateResumeLic` は値だけ更新してプレビューを描き直す。
  作り直すとフォーカスが飛んで文字が打てなくなる）。
  **名前が空の項目も消さずに残す**（履歴書に空の枠として出て、手書きで足せるようにするため）
- 他画面から人材チャットへ飛ぶときは `_pendingChatRoomId` に入れて `showScreen('chat')`。
  チャット画面の初期化完了後に自動で `openChatRoom()` される
  （`showScreen` → `openChatRoom` を直接続けて呼ぶと初期化に上書きされるので注意）

## 9. やってはいけないこと

- **アプリのファイルは `kmt.html` だけ**（単一ファイル構成を維持）。
  例外は開発ツール用の設定ファイルのみ = `CLAUDE.md` / `.gitignore` / `.claude/settings.json`。
  アプリのコード・アセットを別ファイルに切り出さない
- `.gitignore` は `README.md` と `CLAUDE.md` 以外の `*.md` を除外している。
  作業メモや引継ぎファイルをリポジトリに置いても**コミットされない**。この状態を維持する
- トークン等の秘密情報をリポジトリ内のファイルに書かない

### `.claude/settings.json` について（毎回の許可クリックを消す設定）

- 新しいチャットで毎回「許可しますか？」が出るのを防ぐための**権限の事前許可リスト**。
  git 管理下なので、clone した時点でどのチャットにも効く
- `.claude/settings.local.json` は**git管理外**（グローバル gitignore で除外）なので、
  そこに溜まった許可は新チャットに引き継がれない。**恒久的な許可はこの `settings.json` に書く**
- 中身: git 系コマンド・`node`/`python3`（構文チェック用）・ファイル閲覧系・`add_repo` を allow。
  `defaultMode: acceptEdits` で kmt.html の編集も毎回確認しない
- 危険な操作（`git push --force` / `git reset --hard` / `rm`）は `ask` に入れてあるので**必ず確認が出る**。
  ここは外さないこと
- 許可を追加したいときは `allow` に足して**コミット＆プッシュ**する（プッシュしないと新チャットに効かない）
