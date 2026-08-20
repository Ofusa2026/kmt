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
- **未記入でも保存できる**。`saveJobProgress` / `saveJobSheet` が confirm で知らせるだけ
- アラートは `missingJpLaborFields(r)` の1箇所。候補者リストが空の案件は
  「求職者（候補者リスト）」も未入力として数える（`allCjpLinks` を見るので、
  `loadJobProgress` で必ず取っておくこと）

#### 労働局 求人管理簿（`_jpView === 'ledger'`）

- 求人管理の右上「表示」で `KMT管理リスト` ⇄ `労働局 求人管理簿` を切り替える（`setJobView`）
- 様式そのままの並び（①〜⑪＋備考）。**列順は様式なので勝手に変えない**
- ⑪ 職業紹介の取扱状況は `candidate_job_progress` から作る（`_ledgerCandRows`）。
  候補者が複数いれば行が分かれ、左側は rowspan でまとめる
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

### 求人ヒアリングフォーム

- `#hearing` のリンクで**ログインなしで開ける**（`initLogin` の先頭で分岐）。外に渡せる
- 送ると `job_progress` に `progress='詳細確認中'` / `source='hearing'` で登録される。
  **別テーブルを作らない**（＝「直でKMTシステムに反映」がそのまま成立する）
- 精査もれは求人・候補者アラートの「ヒアリングフォームから届いた未精査の案件」に出る
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
