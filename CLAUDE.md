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
   { version:'ver.20260805.29', date:'2026-08-05', notify:true, screens:['companies'], items:[
     'ユーザー向けの変更点を日本語で。何がどう変わるかを具体的に',
   ]},
   ```

`WHATS_NEW` はアプリ内「アップデート履歴」に出る＝**社内ユーザーが読む文章**。技術用語ではなく操作ベースで書く。

### 🆕 画面ごとのアップデート表示（`screens`）

エントリに **`screens` を書くと、その画面の一番上のバナーにも出る**（＝アップデート履歴とは別に、
その画面を開いた人にだけ見せる）。書き方は3つで、**新しいエントリには必ずどれかを付ける**。

| 書き方 | 意味 |
|---|---|
| `screens:['companies','jobs']` | その画面に出す（キーは `SCREENS` のキー） |
| `screens:'all'` | 全画面に出す（画面をまたぐ変更） |
| `screens:[]` | どの画面にも出さない（アップデート履歴だけ） |
| 省略 | 本文のキーワードから推測する（**`UPDATE_SCREEN_KEYS` の1箇所**）。過去のエントリ用 |

- 項目ごとに画面を変えたいときは `items` に `{ t:'文章', s:['jobs'] }` と書ける（文字列のままでもよい）
- **どの画面の話かを決めるのは `_updScreensOf()` / `_updHitsScreen()` の1箇所だけ**
- キーが別のキーの一部になっているとき（`チャット` ⊂ `求人チャット`）は**長いほうだけ**を採る
- 未読は `localStorage`（`kmt_screen_upd_seen_<user_id>`）に画面ごとのバージョンで持つ。
  その画面をまだ開いたことがない人の基準は **`_updBaseVersion()` の1箇所**で、
  はじめてこの機能に触れたときの `kmt_last_seen_version` を `_base` として**1回だけ覚えて固定する**
  （これが無いと、はじめて開いた人のメニューが全部いきなり赤くなる）
  - ⚠️ **`kmt_last_seen_version` を毎回そのまま見てはいけない。**
    あのキーは**ログイン時のお知らせ（トースト）を閉じた瞬間に最新へ進む**ので、
    画面を1つも開いていなくても全部が既読になり、**サイドバーの赤が一斉に消える**（実際に消えた）
- 表示は3つ＝ 画面上部のバナー（`renderScreenUpdateBar`）／サイドバーの赤い項目＋●
  （`updateScreenUpdateBadges`。**画面キーは `nav-item` の onclick から取る**ので項目を二重に持たない）／
  既読にするのは `markScreenUpdatesSeen()`
- **`showScreen` では「バナーを描く → 既読にする → バッジを更新」の順を変えないこと**
  （先に既読にすると新着の件数が0になる）
- 新着があるときは**新着ぶんだけ**を開いて出す。過去ぶんは［これまでのアップデート］で開く
  （全部開くと画面が押し下げられる）

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
- **保存してもモーダルは閉じない。閉じるのは［閉じる］／✕／オーバーレイを押したときだけ**。
  保存の最後は `afterManualSave(modalId)` を呼ぶ（未保存の印を消して最終更新をその場に出す）。
  ⚠️ **新規で作れるフォームは、作った id を覚えてから開いたままにする**
  （`_schedSavedId` / `_myTaskSavedId` / `_introDecSavedId` / `_editingChatTeamId` など）。
  覚えないと、続けて保存したときに**二重に登録される**
  - 例外は「送って終わり」のもの＝新規依頼の送信・新規求人の登録（作った案件をそのまま開き直す）・
    チャットグループの小さなダイアログ・口座振替スケジュールの一括入力
- 求人進捗（`jobProgModal`）は【情報】(`jpf_`) と【求人票】(`jsf_`) の2タブぶんを見る。
  **どちらを保存するかは `_jpSaveOpenTab()` の1箇所**（開いていないタブを空で上書きしないため）
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

#### 💬 求人チャットの絞り込み・凡例（💬 チャットとは別の入れ物）

- メンションの状態と合計は **`_jobChatMentionOnly` / `_jobChatMentionKind` /
  `_jobMentionDirectTotal` ほかの専用の変数**で持つ。
  **💬 チャット（人材・企業）の `_chatMentionKind` / `_mentionDirectTotal` には絶対にさわらない**
  （共用にすると、片方で絞ったときにもう片方まで絞られる）
- 合計を作るのは未読集計の中の `_sumJobOnly()`（求人ぶんだけを足す）。
  凡例は `updateJobMentionLegend()`、絞り込みは `setJobChatMentionKind()` / `setJobChatMentionFilter()`
- 並びは **📌ピン → 未読の多い順 → 最終更新の新しい順**
- ピンのボタンは `toggleChatRoomPin()` を共用する。**そのあと描き直すのは開いている一覧だけ**
  （`#chatRoomList` があれば `renderChatRooms`、`#jobChatList` があれば `renderJobChatList`）

- **KMTドライブのリンク**: そのチャットの「元の欄」に書き戻す。人材＝`workers.drive_url_kmt`／
  求人＝`job_progress.drive_url_kmt`（求人進捗 情報タブ）／候補者＝`candidates.drive_url_kmt`（候補者情報タブ）。
  **どこに書くかは `_chatDriveTarget()` の1箇所だけ。** こうしておくと、画面の欄とチャットの
  「📁 KMT」ボタンが必ず同じものを指す（片方だけズレない）。
  大房側ドライブは人材チャット・企業チャットだけ（求人・候補者はKMT側のみ）
- **メンション**: `chat_messages.mentions`（jsonb の名前配列）。`isMentionForMe()` が部分一致で判定し、
  一覧の @バッジ・「@メンション」絞り込み・ブラウザ通知に使われる。
  返信（↩インライン／🧵スレッド）は `_withReplyMention()` で相手の名前を mentions に自動追加する（自分宛は除外）
  - **@全員 / @all / @everyone の判定は `isMentionToAll(content)` の1箇所だけ。**
    送信時は全員の名前に展開して `mentions` に入れるが、**大房側から直接入るメッセージは
    `mentions` が空のまま届く**ので、本文からも見る（これが無いと @全員 が拾えない）
  - 🔴自分あて（`mentionDirect`）／🔵全体あて（`mentionAll`）／🟡所属チームあて（`mentionGroup`）
    の分け方は **未読集計の1箇所だけ**。本文に @自分の名前 or 自分の投稿への返信＝自分あて、
    @全員＝全体あて、それ以外＝所属チームあて。合計は
    `_mentionDirectTotal` / `_mentionAllTotal` / `_mentionGroupTotal`
  - 一覧の上の凡例（`updateMentionLegend`）の**件数を押すとその内訳だけに絞れる**
    （`setChatMentionKind`。もう一度押すと解除）。絞り方は `_chatMentionKind`
    （`''` 両方／`'direct'`／`'group'`）で、**見るのは `renderChatRooms` の中の1箇所だけ**。
    ［@メンション］のチェックは今までどおり両方まとめて（`_chatMentionKind` は空に戻す）
- **🔕 チャットの離脱（管理者の承認つき）**: `chat_leave_requests` テーブル。
  全社員が全チャットに入っているので、関係のないチャットは通知を止められる
  - 申請の単位は2つ（**`CHAT_LEAVE_SCOPES` の1箇所**）＝ `room`（チャット1件）／`company`（企業まるごと）。
    **どちらも同じ1テーブルで持つ**（`room_id` か `company_id` のどちらかが入る）
  - **承認が要るのは「自分がメイン担当・サブ担当になっている案件」だけ。
    判定は `_chatLeaveNeedsApproval(scope, id)` の1箇所**
    （企業の `main_staff` / `sub_staff`。人材チャットは人材の担当 `staffStr` も見る）
    - 担当のとき … **理由の記入が必須**（空だと出せない）→ `status='pending'` で管理者の承認へ
    - 担当外のとき … 理由は任意で、**その場で `status='approved'`**（承認欄には出ない）。
      `decided_by` に「（担当外のため承認不要）」と残す
    - 部屋が見つからないときは**安全側（承認を通す）**にする
  - **承認されると通知だけ止まる。一覧には残す**（あとから読めるようにするため）＝
    未読・メンションの数を0にし、ブラウザ通知の対象からも外す
  - **離脱しているかの判定は `_chatIsLeftRoom(r)` の1箇所だけ**
    （チャット単位で承認されたぶん＋その企業ごと承認されたぶん）。
    未読集計では `_leftRoomIds` を作って合計からも外す（バッジと一覧で必ず同じ数になる）
  - 入口は2つ。企業カードの🔔（`_leaveBtn`）と、チャットのヘッダーの🔔（`_chatLeaveHeadBtn`）。
    **同じボタンで取り消し・解除もできる**（`requestChatLeave` が状態を見て切り替える）
  - **onclick に名前を埋め込まない**。表示用の名前は `_chatLeaveName()` で引く
    （企業名に `"` が入ると属性が壊れるため）
  - **管理者ではないが「承認だけ」できる人は `APPROVE_ONLY_USERS` の1箇所だけ**（いまは アヒュ）。
  できるのは **🔕 チャット離脱の承認**（`canApproveChatLeave()` の1箇所で判定）と
  **🔁 更新の承認・依頼**（`REN_APPROVERS`）だけ。`isAdminUser()` / `requireAdmin()` は変えていないので、
  **ほかの管理者向けの画面（設定・アナウンス作成・電子印など）は使えないまま**
  - 離脱の承認は4か所で見る＝`loadChatLeavePending` / `renderChatLeaveApprovals` /
    `updateApproveAlerts` の件数 / `decideChatLeave`。**全部 `canApproveChatLeave()` を通す**
  - 💼 求人チャットぶんは今までどおり `CHAT_LEAVE_JOB_APPROVERS`（ニサ）だけ。
    `_chatLeavePendingForMe()` がふるうので、ほかの承認者の一覧には出ない
- 承認は**管理者のマイページ【✅ 承認】**（`renderChatLeaveApprovals`）。
    承認待ちの件数はサイドバーとマイページのアラート欄にも出す（`updateApproveAlerts`）。
    ⚠️ アラートから開くときの画面キーは **`my_account`**（`myaccount` では開かない。実際に開かなかった）
  - **💼 求人チャット・🧑 候補者チャットは別あつかい**（`kind='job'`）
    - **担当かどうかに関係なく必ず承認が要る**（`_chatLeaveNeedsApproval` の中で
      `is_job_chat` / `is_candidate_chat` を見て先に true を返す）。理由の記入も必須
    - **承認できるのは `CHAT_LEAVE_JOB_APPROVERS` の人だけ**（いまは ニサ）。
      ほかの管理者の【✅ 承認】には出さない。ふるいは **`_chatLeavePendingForMe()` の1箇所**で、
      一覧の件数もアラートの件数もこれを見る（必ず同じ数になる）
    - 求人ぶんかどうかは**申請したときに入れる `kind` の1つ**で見分ける
      （`_chatLeaveKind()`。あとから部屋を引き直さなくていいように列に持たせている）
    - **人材・企業のチャットは今までどおり**（`kind='general'`。担当外はその場で承認なし）
- **ピン留め**: `chat_pins` テーブル。`room_id` が入っていれば個別チャットのピン、
  無ければ企業カードのピン（`_chatPinnedCompanyIds` / `_chatPinnedRoomIds`）。
  個別ピンは企業内リストで先頭に並び、`_prependPinnedRoomSection()` が一覧最上部にも表示する。
  **求人チャットのピンも同じテーブル**（💬 チャットの一覧は求人ぶんを最初から外しているので混ざらない）
- **添付**: `chat_messages.attachments`（`[{url,name,mimeType,size}]`）。実体は GAS 経由で Google Drive にアップロード。
  画像（`isChatImageAttachment()` が true）は `driveImageUrl(url, 800)` でサムネイル化してチャット内に直接表示、
  それ以外はファイルリンク。表示に失敗したら `_chatImgFallback()` がリンクに戻す
- 入力欄での画像貼り付けは `initChatPasteImage()`（document の paste を拾って `addChatFiles()` に渡す）
- **本文のURLのリンク化**: 見つけ方は **`CHAT_URL_RE` の1箇所だけ**（`renderMessageContent`）。
  - **URLに使う文字は半角だけ**にしてある。全角・日本語まで拾うと
    「（https://…/x）を見てください」の後ろの文章までリンクに入る（実際に入った）
  - 末尾の `.` `,` `)` などは `CHAT_URL_TAIL_RE` で落とす（「…/view です。」対策）
  - **メンションの色付けとは分けて処理する**＝ URL のところで文字列を切り、
    URL 以外の断片にだけ `_mentionHtml()` を掛ける。まとめて replace すると
    `.../@user` の @ をメンションと間違えたり、作った `<a>` の中をメンションが壊す
  - リンクの色は自分の吹き出し（濃い青）だけ明るくする（`renderMessageContent(text, isMine)`）

#### 🙋 大房側ドライブURLの記入依頼（外国人材の【4. ドライブリンク】）

「大房側ドライブURL」の下の［🙋 担当者に依頼］→ 担当者を選ぶ → **その人材のチャットに
依頼を投稿し、選んだ人を @メンション**する（`askOfusaDriveRequest` / `sendOfusaDriveRequest`）。

- **依頼先の一覧と並び順は `OFUSA_REQ_TEAM`（`大房チーム`）／`OFUSA_REQ_TOP` の1箇所だけ**。
  `OFUSA_REQ_TOP` に書いた人がその順で先頭（いまは 神出　紘陽 → サンデー）、残りは名簿の順。
  `is_active=false` の人は出さない
- **文面は `OFUSA_REQ_TEXT` の1箇所だけ**
- 氏名の**全角／半角スペースの違いは同じものとして扱う**（`_ofusaNm()`）。
  名簿に `team` が入っていないときは `users` を取り直してから並べる
- 投稿は `_ensureWorkerRoomId()` で人材チャットを用意してから `chat_messages` に1件入れる。
  **`mentions` には選んだ人の名前をそのまま入れる**（本文の `@名前` だけだと、
  `@姓　名` は空白で切れて拾えない。通知の判定は `isMentionForMe()` ＝ `mentions` を見る）
- **人材を保存する前は使えない**（`editingWorkerId` が無いときはトーストで知らせる）。
  依頼したあとは欄の下に「✅ ○○さんに依頼しました ／ 💬 チャットを開く」を出す（画面だけ・DBに列は持たせていない）

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

- 求人管理の上のタブは3つ（**`JP_BUCKETS` の1箇所**）＝ 💼 現在の求人 ／ 🏢 大房側の求人 ／ 📦 過去の求人。
  枠に入るかどうかは **`_jpInBucket()` の1箇所**。大房側の求人は `isOfusaJob(r)` が true で
  過去に移っていないもの（過去に移ったものは 📦 のほうだけに出す）
- **「大房側の案件」の判定は `isOfusaJob(r)` の1箇所だけ。** 会社名の文字列一致では絶対に判定しない
  （大房側の実データに `株式会社KMT` / `株式会社ＫＭＴ` / `KMT(82)` のような表記ゆれがある）。
  `ofusa_flag` が唯一の正で、発注書を送った案件は自動で立つ
- 求人詳細の「🏢 大房側の案件」チェックと「📱 求人LINEグループ名」は
  **区分が【紹介案件】のときだけ出す**。開閉するidは `JP_INTRO_ONLY_BLOCKS` の1箇所にまとめてあり、
  `toggleJpOrgPicker()` が区分の変更で連動させる（登録支援機関検索もこの配列に入っている）
- 発注書の対象ステータスは `OFUSA_ORDER_STATUSES`（内定・決定）。増やすときはここだけ直す

### 📖 求人管理・候補者管理の使い方（ガイド）

- 入口は2つ＝求人管理と候補者管理の**画面右上の［📖 使い方］**。
  どちらから開いても中身は同じで、タブで切り替える（`openJobCandGuide('job'|'cand')`）
- **説明の文章と図は `JC_GUIDE` / `_jcGuideSvg()` の1箇所だけ。**
  2つの画面は「求人と候補者の紐付け」でつながっているので、**ガイドを分けない**
  （分けると片方だけ直して食い違うため）
- 各タブの中身は `purpose` / `steps` / `tabs` / `links` / `tips` / `cautions` の6つ。
  仕組みを変えたら該当するところだけ直す

### 📄 求人票の3分類（特定技能 ／ 一般 ／ 高度人材）

同じ求人案件でも、出す求人票は3種類ある。**どれで作るかは案件ごとに選ぶ**
（【📄 求人票＋よくある質問】タブのいちばん上）。

| 分類（`sheet_kind`） | 中身 | 項目の定義 | 入力した値の保存先 |
|---|---|---|---|
| `ssw`（既定・空も同じ） | 🌏 特定技能の求人票（従来のフォーマット） | `JOB_SHEET_GROUPS` | `job_progress` の列そのもの |
| `general` | 📋 一般の求人票（87項目） | `JOB_SHEET_GEN_GROUPS` | `job_progress.sheet_ext.general` |
| `hsp` | 🎓 高度人材の求人票（108項目・日英併記） | `JOB_SHEET_HSP_GROUPS` | `job_progress.sheet_ext.hsp` |

- **分類は `JOB_SHEET_KINDS` の1箇所だけ。** 選択欄・入力欄・書類・労働局指定項目のチェックが
  ここから同時に追従する。**案件がどれかを決めるのは `jobSheetKindOf(r)` の1箇所**
- 一般・高度人材は**列を増やさない**（項目が195もあるため）。`sheet_ext`（jsonb）に分類ごとに入れる。
  読み書きは **`_jsExt()` / `_jsExtSet()` の2つだけ**を通す
- ⚠️ **項目の `no` は様式（Excel）の No. とそのまま同じで、キー（`k`）も `g<no>` / `h<no>`。
  番号を振り直さないこと**（入力済みの内容が迷子になる）。足すときは使っていない番号を末尾に足す
- 印は様式の凡例どおり＝ `req:true` ◎ 労働局指定（職安法5条の3・規則4条の2の明示義務。空欄だと受理できない）／
  `agent:true` △ 紹介事業者が記入／`opt:true` ○ 任意／どれも無ければ ● 必須
- `ph` は記入例（欄の中に薄く「例）…」と出す）、`note` は法令根拠・留意点（欄の下）、
  `en` は英語の項目名（高度人材だけ。日英併記の様式のため）
  - ⚠️ **記入例には必ず「例）」を付ける**（`_jsExtFieldHtml` の1箇所）。付けないと値が入っているように見える。
    色は `.jsx-f::placeholder` の1箇所で薄くしてある
- **すでに入っている内容を自動で入れる対応表は `JOB_SHEET_EXT_FILL` の1箇所だけ**
  （会社名・就業場所・職種・仕事の内容・勤務時間・賃金など）。**入れるのは空いている欄だけ**
- **書類を組み立てる入口は `buildJobSheetDoc(j)` の1箇所**（分類を見て振り分ける）。
  求人票作成の画面（`renderFlyerByMode`）も、モーダルの［📄 求人票を表示］（`openJobSheetPreview`）も必ずここを通る
  - 特定技能＝`buildJobOrderSheet`（従来どおり）／一般・高度人材＝`buildJobSheetKindDoc`
  - ⚠️ **紙のページは自分で分ける**（`JOB_SHEET_DOC_ROWS`＝1ページの行数のめやす）。
    表を1枚に流し込むと、途中のページに余白が付かない（`@page{margin:0}` のため）。
    末尾の署名欄が入りきらないときは `JOB_SHEET_DOC_SIGN` ぶんのページを足す
  - ⚠️ **透かしの上に載せる `position:relative` の箱は必ず閉じること。**
    閉じ忘れると次のページが前のページの中に入れ子になる（実際になった）
  - ◎の未記入は**黄色**（`JOB_SHEET_BLANK_BG`）。様式の黄色いセルと同じ意味
- **労働局指定項目は分類で変わる。判定は `_jpLaborFieldsOf(r)` の1箇所だけ**
  （特定技能＝`JP_LABOR_FIELDS` そのまま／一般・高度人材＝情報タブぶん＋その様式の ◎）。
  `missingJpLaborFields()` も欄の薄赤（`JP_MARK_FIELDS`）もここを見るので、必ず同じ内容になる
- 入力欄の id は **`jsx_<k>`**（特定技能は今までどおり `jsf_<col>`）。
  `_jpTabOfField()` は `jsf_` と `jsx_` の両方を求人票タブとして見る
- **「📥 他社の求人票から読み込む」と「❓ よくある質問」は特定技能のときだけ出す**
  （読み込みの対応表が `JOB_SHEET_FIELDS` のため／FAQは寮・国籍・宗教など特定技能向けの質問のため）
- 種類を切り替えても**打ちかけは種類ごとに残る**（`_jsExtDraft`。モーダルを開き直すと忘れる）。
  ⚠️ **書類は打ちかけを見ない**（`_jsExtDocValues()`）。見ると別の案件の求人票に前の案件の内容が出る
- 一覧のコードの右に出る印は `_jpSheetKindBadge()`。**特定技能のときは出さない**（今までの一覧を変えないため）
- **種類を選ぶ場所は2つ。役割が違うので混ぜないこと**
  | 場所 | できること | 書き込み |
  |---|---|---|
  | 求人案件の【📄 求人票】タブ（`_jsKindPickerHtml`） | 種類を決めて**項目を入力**する | ［💾 保存］で `sheet_kind` と `sheet_ext` を書く |
  | 求人票作成の画面（`renderFlyerKindBar`） | **表示の切り替えだけ**（印刷・PDFの前に見る） | **書き込まない**（`_flyerSheetKind` に持つだけ） |
  - 求人票作成で見ている種類は **`_flyerKindNow()` の1箇所**。求人を選び直すと `null` に戻る
    ＝その案件に保存されている種類に戻る
  - ［✏️ 項目を入力］（`flyerEditSheet`）は、**いま見ている種類のまま**求人案件の【📄 求人票】タブを開く
    （開いてから `_jpSheetKind` に入れる。保存は今までどおり 💾 のとき）

### 求人案件のモーダル（3タブ）

| タブ | 中身 | 保存 |
|---|---|---|
| 📋 求人進捗 情報 | 進捗・区分・コード・所属機関・担当・優先度・**手数料管理**・KMT社内・コメント | `saveJobProgress` |
| 📄 求人票＋よくある質問 | **求人票の種類**（上の節）→ 求人ノート → 求人票の項目編集（📥テキストから読み込む付き）→ よくある質問 | `saveJobSheet` / `saveJobFaq` |
| 👥 候補者リスト | `candidate_job_progress` の紐付け | 行ごとに即保存 |

- **タブごとに保存する範囲を分けてある。** 開いていないタブの欄を空で上書きしないため、
  `saveJobProgress` は求人票の列に触らない／`saveJobSheet` は `JOB_SHEET_FIELDS` の列だけを書く
- `openJobProgressDetail(src_row, focusId)` の第2引数で、開いたあと特定の項目まで飛ばせる。
  **どのタブの項目かは `_jpTabOfField()` の1箇所だけで判定する**（`jsf_` / `jsx_` 始まり＝求人票タブ、
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

### 求人票のページの組み方（印刷・PDF）

- **ページ1の2列は `<table>` で組む。grid にしない。**
  Chrome の印刷は **grid コンテナをページで分割できない**ので、1mm でもはみ出すと
  2列がまるごと次のページへ送られ、**1ページ目がヘッダーだけの空白ページになる**（実際になった）。
  table なら足りないぶんだけ次ページへ続く
- **用紙の余白は `.jobsheet-page` の `padding` の2箇所**（印刷用＝`_flyerPrintHtml`／
  画面用＝`#jobFlyerPreview.sheet-mode .jobsheet-page`）。**必ず同じ値にする**
  （違うと画面と印刷で入る量がズレる）。いまは `10mm 12mm`

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
- **リンクの発行は任意。** 出さなくても求人票タブの項目を直接入力して保存できる
  （見出しに［任意］と案内を出してある。必須の手順と誤解されたことがある）
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
- ⚠️ **求人票の種類（特定技能／一般／高度人材）で見る項目が変わる。分けるのは `_jpLaborFieldsOf(r)` の1箇所**
  （一般・高度人材は `JOB_SHEET_GEN_GROUPS` / `JOB_SHEET_HSP_GROUPS` の `req:true` を見る）。
  `missingJpLaborFields()` も `JP_MARK_FIELDS()` もここを通す
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
- 見出しは「求人票」。左上に `_flyerLogoHead()` で「（KMTロゴ）登録支援機関KMT」を出す
- **ロゴを出すかどうかは `_flyerShowLogo()` の1箇所だけ**（求人票作成の左側のチェック `ff_show_logo`）。
  左上のロゴ・社名と、背景の透かし（`_flyerWatermark`）の**両方**がこれで消える。
  チェック欄が無い画面（求人案件モーダルの求人票プレビューなど）では今までどおり出す。
  チェックの onchange は `renderFlyerByMode()`（`previewFlyer()` だと候補者向けに戻ってしまう）

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
  新規登録 →（見送り）一次面接 → 提案待ち → 企業面接待ち → 結果待ち → 内定 ／
  （見送り）企業面接不合格 ・（辞退）本人が応募キャンセル
- **求人案件の【👥 候補者リスト】に出さないものは `hideJob:true` の1箇所で決める**
  （新規登録 と（見送り）一次面接＝まだ企業へ出していない段階）。
  `JP_CAND_HIDDEN_STATUSES` はここから作るので、足すときは `hideJob` を付けるだけ
- 一覧のバッジ（`candStatusBadge`）・絞り込み・モーダルの選択欄・タイトルは全部ここから作る
- **以前のステータス（選考中・入国待ち・配属済 など）が入っている候補者は、
  その値も選択肢に足して残す**（`candidateStatusOptionsHtml`）。保存で勝手に書き換えない
- ステータス欄は候補者情報タブの**いちばん上**（氏名の上）。変えると
  `_cndSyncTitle()` がモーダルのタイトルと説明文も更新する
- **モーダルの見出しは氏名**（`📋 <氏名> — <バッジ>`）。名前を決めるのは `_cndTitleName()` の1箇所で、
  **`cf_name` が空なら空のまま**（`rf_name` に戻さない。消したのに前の名前が残って見えるため）。
  氏名が空・新規・`CAND_LINK_PLACEHOLDER` のときだけ「候補者詳細／候補者登録」に戻す。
  打っている先から追いかけるのは `_bindCandSyncFields()` の中（`cf_name` / `rf_name`）
- 人材担当（`cf_staff`）は**チームが `GLT` のユーザーから選ぶ**（`gltStaffOptionsHtml`）。
  名簿が未読込なら `_cndFillStaffSelect()` が取り直して作り直す。
  いまの値が一覧に無いときは、その値も選択肢に残す

#### 🎓 学歴（最終学歴・卒業状況）

- **選択肢は `EDU_LEVELS`（中学校卒業〜大学院卒業）／`EDU_STATUSES`（卒業・中退・在学中）の1箇所だけ**。
  保存先は `candidates.education_level` / `education_status`
- 卒業年月しか無いと「2年生で中退」「まだ在学中」が書けないので、
  **どこまで行ったか（最終学歴）と、卒業したかどうか（卒業状況）を分けて持つ**
- 卒業状況で**日付の見出しが変わる**（`onEduStatusChange()` の1箇所。中退＝中退年月／在学中＝入学年月）。
  履歴書の学歴の表も同じ見出しで出す。学歴の1行を作るのは `_eduLine()` の1箇所

#### 🕌 宗教（選択式）

- **選択肢は `CAND_RELIGIONS` の1箇所だけ**（仏教／ヒンズー教／イスラム教／キリスト教）。
  欄を作るのは `religionFieldHtml(prefix, cur)` の1箇所で、候補者情報タブ（`cf_`）・
  履歴書作成（`rf_`）・外部の履歴書作成リンクが同時に追従する
- 選択肢に無いものは「その他（記入）」（`CAND_RELIGION_OTHER`）の欄に残す。
  **保存先は今までどおり `candidates.religion` の1列**（`religion_other` という列は無い）
  - 読み書きは **`_religionValue(prefix)` / `_religionSet(prefix, val)` の2つだけ**を通す
  - 旧・応募者登録リンクは `cf_*` を総なめして保存するので、`_cndFormValues()` で
    `religion` / `religion_other` を除いてから `_religionValue('cf')` で組み立てる
    （でないと存在しない列が混ざって保存がまるごと落ちる）
- 2つのタブで写すため `CAND_SYNC_FIELDS` に `religion` と `religion_other` の両方を入れてある。
  写したあとの記入欄の出し入れは `_cndCopyField` の中で `onCandReligionChange()` を呼んで合わせる
- 外部フォームの訳は `EXT_FORM_I18N` に選択肢ぶんも入れてある（`option` は「日本語 / 訳」になる）

#### 📢 冒頭の案内（はじめにお読みください）

- **文章は `app_settings.job_form_intro` の1箇所**（1行＝1項目のJSON配列）。
  まだ直していないときは `JOB_FORM_INTRO_DEFAULT` を出す
- **直せるのは管理者だけ**（`requireAdmin`）。求人票タブの「🔗 外部記入リンク」の横の
  ［✏️ 冒頭の案内を編集］（`openJobFormIntroEdit`）。プレビュー付き・［元の文章に戻す］あり
- `**…**` ではさむとその部分が太字になる（`_jobFormIntroLineHtml`）
- 外部フォームを開くときに `loadJobFormIntro()` で読む（ログイン不要のページなので、
  `app_settings` の読み取りだけで済ませる）

#### 👤 求人の外部フォームの「ご記入者」

- ① お名前 ② 記入者情報（`JSF_BY_KINDS` ＝ 受け入れ企業／登録支援機関／その他＋記入欄）。
  保存先は `job_sheet_links` の `by_kind` / `by_other` / `org_no` / `org_name`
  （**回答（`answers`）には入れない**。answers は求人案件の列にそのまま書くので、
  存在しない列を混ぜると保存がまるごと落ちる）
- 「登録支援機関」のときだけ **登録支援機関検索** を出す。
  **外部の方には一覧を全部見せない**＝打った名前に近いものだけを出す（`_jsfOrgMatch`）。
  ゆらぎ（株式会社などの有無・全半角・記号）は `_jsfNorm()` の1箇所で吸収する
- 見つからないときは「新規登録をお願いします」と案内。［＋ 新規登録］は最初から出ている
- 新規登録は社内の「＋ 新規提携先」と**同じ `intro_reg_orgs`** に入れる（別管理にしない）。
  出す項目は **`JSF_ORG_NEW_COLS` の1箇所**（社内の提携先情報から選ぶので、
  入力欄の作り方＝`_iroFieldHtml` は共用。コードは社内で後から付ける）
- 新規登録のモーダルは **z-index 99999**（外部フォームが 99998 なので、それより上に出す）
- 送信のとき、選ばれた機関にコードがあり、求人案件の登録支援機関コードが空なら入れておく

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
  - **列は `CAND_LINK_LOG_COLS` の1箇所だけ**（見出し・並べ替えのキー・検索の対象）。
    列を足せば、見出し・A→Z/Z→Aの並べ替え・検索が同時に付いてくる
  - 見出しを押すと並べ替え（`setCandLinkLogSort`。1回目 A→Z、もう一度で Z→A。
    発行日時だけ新しい順から）。検索は `setCandLinkLogQ`（全列＋日付の見た目から探す）
  - 打つたびに描き直すので、**検索欄のフォーカスとカーソル位置を戻してから返す**
    （戻さないと1文字ごとにフォーカスが外れて打てない）

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
- **表示するステータスは必ず候補者本体の値**（`_linkStatusOf()`）。古い紐付けに以前の値が
  残っていても、開いたときに `_repairLinkStatuses()` がDBごとそろえる
- **求人側の【👥 候補者リスト】に出るのは「提案待ち」以降だけ。**
  新規登録・未設定の候補者は出さない（判定は `_jpCandListVisible()` の1箇所）。
  紐付け自体は消えないので、候補者側の「求人への紐付け」には出る。
  隠れている人数は一覧の上に出て、［表示する］（`toggleJpHiddenCands`）で確認できる
- **紐付けのステータスは候補者情報のステータスと同じもの**（`CANDIDATE_STATUSES`）。
  1人の候補者につき値は1つで、**書き込みは `_syncCandidateStatus()` の1箇所だけ**。
  求人側の【👥 候補者リスト】でも、候補者側の紐付けの欄でも、候補者情報のいちばん上でも、
  どこで変えても `candidates.status` とその候補者の紐付け全部が同じ値になる。
  選択肢は `candStatusOptionsShort()`（以前のステータスが入っている行は、その値も残す）。
  新しく紐付けたときは、その候補者のいまのステータスをそのまま入れる
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
  **項目名より目立たせない**＝ 8px・細字・水色（`#3d9bd1`）。
  以前の `#a8d8ef` は白地で読めなかったので、色味とサイズはそのままに濃さだけ上げてある
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
- **表の大きさを変えるのは `_jnHitEdge()` ＋ `_jnInitResize()` の1組だけ**（`.jn-table`）
  - **罫線をドラッグ**＝列幅（縦の罫線）／行の高さ（横の罫線）。掴める距離は **`JN_EDGE` の1箇所**
  - **表の右下の［⇔］をドラッグ**＝表全体の幅（列の比率はそのまま）。目印は `_jnShowGrip()`、
    いま動かす罫線を示す青い線は `_jnGuide()`
  - 列幅は `<colgroup>` に入れる（`_jnEnsureColgroup()`）。`table-layout:fixed` が要る
  - **`pointer` イベントで書く**（マウスも指も同じ処理。タブレットでも動く）
  - ⚠️ **`#jnGrip` / `#jnGuide` は `document.body` に置くので z-index を最大にする**。
    モーダル（99999 まである）より下だと、つまみが隠れて掴めない（実際に掴めなかった）
  - 動かしたら `markJobNoteDirty()` で「未保存」にする（保存し忘れの防止）
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
- **［📖 使い方］は画面の右上**（`SCREENS` のツールバー）。中身は `openScheduleGuide()` で、
  **説明の文章と図は `SCHED_GUIDE_PURPOSE` / `SCHED_GUIDE_LINKS` / `SCHED_GUIDE_LINKS_OTHER` /
  `SCHED_GUIDE_TIPS` / `_schedGuideSvg()` の1箇所だけ**。
  `SCHED_GUIDE_LINKS` は **`SCHED_WORKER_FIELDS` と対になる説明**なので、向こうを直したらここも直す
- **サイドバーの赤いバッジは出さない**（何の件数か分かりにくかったため。
  `updateSupportScheduleBadge` / `refreshSupportScheduleBadge` は空の関数として残してある）

#### 🔁 支援スケジュール ⇄ 人材情報 の自動反映

- **対応表は `SCHED_WORKER_FIELDS` の1箇所だけ**（種別 → 人材の日付の列／担当の列）。
  **どちらで入れても、もう片方に入る**

  | 予定の種別 | 人材情報【8. 支援情報】 |
  |---|---|
  | `pre_guidance` 事前ガイダンス | 事前ガイダンス実施日／実施担当名 |
  | `life_orientation` 生活オリエンテーション | オリエンテーション実施日／実施担当 |
  | `entry` 入国（上陸日） | 入国：出迎え日（上陸日）／担当者名 |
  | `final_return` 本帰国（出国日） | 出国：見送り日（出国日）／担当者名 |
  | `temp_return` 一時帰国 | 一時帰国日（`return_home_date`。担当の欄は無い） |
  | `reentry` 再入国 | 再入国日（`return_jp_date`。担当の欄は無い） |

- 予定 → 人材は **`_syncScheduleToWorker()` の1箇所**（`saveSchedule` から呼ぶ）。
  人材 → 予定は **`_syncWorkerToSchedules()` の1箇所**（`saveWorker` から呼ぶ。
  同じ人材・同じ種別の予定があれば日付を書き替え、無ければ作る）
- **`staff` が `null` の種別（一時帰国・再入国）は日付だけ写す**（担当の列が人材情報に無いため）。
  ここを見落とすと `patch[null]` を書いて保存が落ちる
- 種別を足すときは **`SCHED_WORKER_FIELDS` と `SCHED_GUIDE_LINKS`（使い方の説明）をセットで**直す

### 📅 面談記録（所属機関と外国人材で共通 ／ `company_meetings`）

- **入口は2つ、実体は1つ**＝ 所属機関詳細の【📅 面談記録】タブと、外国人材詳細の【🗒️ 面談記録】タブ。
  **どちらから入れても `company_meetings` の同じ行**になる（別テーブルを作らない）
  - 所属機関側 … その企業の記録（`company_id`）
  - 外国人材側 … 参加外国人材にその人材が入っている記録（`worker_ids=cs.["<id>"]`）
  - 記入フォームは**ポップアップ1つを共用**（`openMeetingForm(m, {companyId, workerId})`）。
    人材から開くとその人材が最初から選ばれている
  - **1記録＝1人材**（支援カレンダーの予定が1人材なので、それと1対1にする）。
    フォームの人材は**ラジオで1名**。⚠️ ただし**以前の入れ方で複数人が入っている記録**は
    今までどおりチェックボックスで直せる（**判定は `openMeetingForm` の `_multi` の1箇所**）。
    この互換の道は残してあるが、**過去の合同記録（40件・のべ162名）は
    ver.20260831.351 で1名ずつに分割済み**（相談内容・対応結果は全員にそのままコピー）。
    元に戻せるよう `company_meetings_split_backup`（分割前の `worker_ids` / `schedule_id`）を残してある
  - **保存すると「▶ 次の人材を記録する」**が出て、共通項目（項目・年度・期間・面談日・実施方法・
    面談相手・対応者）を引き継いだ空のフォームが開く。**引き継ぐ中身は `_meetingCarry` の1箇所**。
    同じ日の記録がもう入っている人材には ✔ を出す（`_meetingDoneWorkerIds()`）
  - ⚠️ **保存したあとの一覧の取り直し（`_meetingAfterChange`）は待たない。**
    先に `showLoading(false)` を呼んでから確認のダイアログを出す
    （待つと「読み込み中」が出たままになり、［▶ 次の人材を記録する］が押せなくなる）
  - 描くのは **`renderMeetingList(side)` の1箇所**（`side` は `co`＝所属機関 / `wk`＝外国人材）。
    保存・削除のあとの取り直しは `_meetingAfterChange()`（開いているほうを両方とも取り直す）
- **項目（種類）と、その項目で出す欄は `MEETING_KINDS` の1箇所だけ**
  （📅 定期面談 ／ 💬 随時相談 ／ 🗒️ その他）。`fields` に書いた欄だけがフォームに出る
  （出し入れするのは `mfg_<名前>` の入れ物。切り替えは `renderMeetingKindFields()`）
  - **その他は項目名を書ける**（`kind_other`）。旧「やり取りの記録」の 電話・チャット等はここに入っている
  - **定期面談だけ 年度・期間・実施方法・面談相手 が出る**。随時相談・その他は
    **年度・期間を日付から自動で入れる**（`_meetingFyQ()` の1箇所。4月はじまり）
- 欄は 項目・年度・期間・面談日・**実施方法**・面談相手（企業担当者）・参加外国人材・
  **対応者の氏名**・**相談内容**・**対応結果**

  | 画面の名前 | 列 |
  |---|---|
  | 相談内容（旧・詳細メモ） | `memo`（**列名は変えない**。過去の記録がそのまま出るように） |
  | 対応結果 | `result_memo` |
  | 対応者の氏名 | `staff_name` |
  | 項目 ／ 項目名（その他） | `kind` / `kind_other` |
  | 実施方法 | `event_method`（`support_schedules` と同じ値） |
  | 支援カレンダーの予定 | `schedule_id` |

- ⚠️ 旧・外国人材の「やり取りの記録」（`workers.contact_log`）は**面談記録に移した**。
  列は消していない（保存でつぶさないよう `wf_contact_log` に元の値を持ち回るだけ）。
  Excel からの貼り付け取り込み（`importContactLogFromText`）は、
  **company_meetings に面談記録として入れる**（項目が合わなければ【その他】＋項目名）
- **実施方法の選択肢は `MEETING_METHODS` の1箇所だけ**（未設定／🖥️ オンライン／🏢 訪問）。
  **支援スケジュールのモーダルも同じものを使う**（`meetingMethodOptionsHtml()`）

#### 📆 支援カレンダー（定期面談）との連携

- **支援カレンダーで［定期面談］を選ぶと、メモの代わりに「面談記録の相談内容」「面談記録の対応結果」が出る。**
  保存先は予定ではなく **`company_meetings`**（その予定に紐づく1件）＝
  カレンダーから書いても【面談記録】タブから書いても同じ1件になる
  - 書き込むのは **`_saveMeetingFromSchedule()` の1箇所**（記録があれば更新／無ければ作る。
    中身が空のときは作らない）。予定は1人材なので `worker_ids` は1名
  - 逆方向は **`_syncScheduleNotes()` の1箇所**（面談記録を保存したら予定の `notes` もそろえる）。
    `notes` はカレンダー上の表示用の写しで、**正は `company_meetings.memo`**
  - 欄の出し入れは `onSchedTypeChange()` の中（メモに書きかけがあれば相談内容へ引き継ぐ）

- 支援カレンダーの **定期面談（`support_schedules.event_type='periodic_meeting'`）は、
  面談記録の一覧にもそのまま出る**。まだ記録が無いものは「📅 予定」の行で、
  ［📝 記録する］（`recordFromSchedule`）を押すと日付・実施方法・担当・人材が入った記入フォームが開く
  - 予定 → 1行に直すのは **`_meetingSchedRow()` の1箇所**。
    **支援カレンダーの「メモ」（`support_schedules.notes`）は面談記録の【相談内容】（`memo`）に入る**
    ＝ 一覧では「📝 支援カレンダーのメモ」として出て、［📝 記録する］でそのまま記入フォームに入る
    （予定の取得SELECTに `notes` を入れておくこと）
  - 記録が入った予定は「📅 予定」を出さず、記録のほうに 🔗 カレンダー を付ける
- **記録が入っているかの判定は `_meetingSchedDone()` の1箇所**
  ＝ その予定に紐づいた記録があり、**対応結果（`result_memo`）が入っていること**
- **予定日から `MEETING_TODO_DAYS`(14)日たっても対応結果が空＝「⏰ 記入待ち」。
  判定は `_meetingSchedTodo()` の1箇所**
- 手で入れた定期面談も、**同じ企業・`MEETING_MATCH_DAYS`(3)日以内の予定があれば自動で結び付ける**
  （`_meetingMatchSchedule()`。これが無いと記入待ちが消えない）
- **担当者へのお知らせ**。数えるのは **`_meetingTodoMine()` の1箇所だけ**
  - **担当かどうかの判定は `_meetingTodoIsMine()` の1箇所**＝ その企業のメイン／サブ担当・
    **人材の担当（`workers.staff`）**・予定の担当者（`staff_names`）のどれかに自分がいること。
    関係のない人には出さない
  - **📅 定期面談報告書に必要な日付（配属日・退職日）が空の人材**のお知らせは別もの＝
    `loadMeetDateTodos()` → `updateMeetDateAlerts()`（サイドバー `meetDateAlerts` ／
    マイページ `myMeetDateAlerts`）。押すと `openMeetDateFix('mine')` が開く
    - **出すのは GLT の担当だけ**（営業には出さない）。**判定は `_meetDateFixStaff()` の1箇所**
      ＝ その企業のメイン・サブ・チーム担当と人材の担当のうち、**チームが GLT の人**
    - 人材は**足りないものだけを引く**（退職済で退職日が空／配属日が空）ので、全人材は読まない
    - **処理の期限は `MEET_DATE_FIX_DUE` の1箇所だけ**（いまは `2026-09-04`）。
      曜日は日付から出す（`_meetDateDueText()`）ので、**日付を変えるだけでよい**。
      これが入っているとサイドバーのチップ・マイページ・画面の見出しと案内に
      「【要対応】各担当 ◯月◯日（◯曜日）までに処理してください！」が出て、色が赤系になる。
      **空にすると期限の表示だけ消える**（お知らせ自体は残る）。
      **⏰ 面談記録の記入待ちも同じ期限を使う**（`updateMeetingAlerts` / `openMeetingTodoList`）
    - **期限を過ぎたら赤く点滅する**（アナウンスの未確認と同じ `kmtAnnBlinkRed`）。
      **過ぎたかの判定は `_meetDueOver()` の1箇所だけ**で、見た目を作るのは
      `_meetDueTag()`（【要対応】／【期限超過】の印）・`_meetDueTagText()`（モーダルの見出し用の文字）・
      `_meetDueLine(inline)`（案内の1行。`inline` はお知らせの中に入れるとき）の3つ。
      サイドバーのチップは `_alertChipHtml(..., cls)` の最後の引数に `due-over` を渡す。
      **点滅のCSSは `.due-over` の1箇所**
    - サイドバー・マイページの並びは **📅 報告書の日付 → ⏰ 面談記録の記入待ち**
      （`meetDateAlerts` / `meetingAlerts` の並び順。マイページは `my` 付きの同じ2つ）
  - **`MEETING_TODO_WATCHERS` に書いた人だけは全社ぶんが出る**（いまは ニサ）。
    確認と全員への共有のためで、見出しに「（全体）」、一覧に🙋担当者名が付く（`_meetingTodoStaffText()`）。
    やめるときはこの配列を空にする
  - 出る場所は3つ＝ サイドバー `meetingAlerts` ／ マイページ `myMeetingAlerts`
    （どちらも `updateMeetingAlerts()`）／ 面談記録の一覧の上のバナー（`renderMeetingList`）
  - ブラウザ通知は `_meetingNotify()`。**同じ日に何度も出さない**（`localStorage` に日付を覚える）
  - 元データを取るのは `loadMeetingTodos()`（起動時とマイページを開いたとき、記録を直したとき）

- **対応者の選択肢を作るのは `meetingStaffOptionsHtml(cur, companyId)` の1箇所だけ**。並びは
  ① その企業の **メイン担当 → サブ担当**（`main_staff` / `sub_staff`。名簿にいない名前でも先頭に出す）
  ② 続けて **GLT と 営業** のユーザー（①に出したぶんは重ねない）。大房チーム・総務は出さない
  - いまの値が①②のどちらにも無いときは、**その値も選択肢に残す**（担当が変わっても消えないように）
  - 名前の**全角／半角スペースの違いは同じもの**としてあつかう
- 一覧では相談内容と対応結果を **見出しを付けて分けて出す**（どちらが何か分かるように）。
  対応者は面談相手の隣に 🙋 のバッジで出す
- 面談記録は **相談記録書（5-4号）の元データ**になる（下の節）

### 求人ヒアリングフォーム

- `#hearing` のリンクで**ログインなしで開ける**（`initLogin` の先頭で分岐）。外に渡せる
- 送ると `job_progress` に `progress='詳細確認中'` / `source='hearing'` で登録される。
  **別テーブルを作らない**（＝「直でKMTシステムに反映」がそのまま成立する）
- 精査もれは求人・候補者アラートの「ヒアリングフォームから届いた未精査の案件」に出る
- **求人管理のツールバーからはボタンを外した**（企業への記入リンクは案件ごとに発行する運用にしたため）。
  いまフォームを開けるのは `#hearing` のリンクだけ。`openHearingForm()` / `copyHearingLink()` は残してある
- **求人LINEグループの「✅ 作成済み」**は `job_progress.line_group_done`（jsonb `{done,at,by}`）。
  判定は **`_lineGroupDone()` の1箇所**で、アラート（`noLine`）も欄の下の表示も同じものを見る。
  グループ名がまだ空でも、押せばアラートから消える（名前は後から入れられる）。「取り消し」で戻せる
- **LINEグループはKMT側では作れない**（LINE基盤は大房側にしかない）。
  `requestJobLineGroup()` が求人チャットに作成依頼を投稿し、作った人が
  求人案件の `line_group_name` に書き戻す運用。募集中なのに空の**紹介案件**はアラートに出る
  （LINEグループは紹介案件だけの話なので、KMT案件はアラートにも入れない）

### 🔁 更新状況ステータス

- **選択肢・並び・色・説明・月の見出しの件数チップは `REN_STATUSES` の1箇所だけ**。
  人材詳細の選択欄／更新リストの絞り込み（`REN_STATUS_OPTIONS`）／バッジの色（`REN_STATUS_COLORS`）／
  月の見出しの内訳が、ここから作られるので同時に追従する
  - `old:true` … いまは選ばせない古い値（すでに入っているぶんを残すために持っている。
    絞り込みでは【旧ステータス】に出る）
  - `chip` … 月の見出しに件数を出すもの（書いたものだけ出る）
- **前の名前の読み替えは `REN_STATUS_ALIAS` / `_renStatusNorm()` の1箇所だけ**
  （`更新決定`→`更新決定（通常）` ／ `優先度高め`→`更新決定（優先度高め）`）。
  DBは移行済みだが、取りこぼしがあってもバッジ・件数・絞り込み・選択欄で新しい名前になる
- **名前を変えるときは `workers.renewal_status` の移行も一緒にやること**
  （表示だけ変えると、絞り込みの件数と一覧が食い違う）
- `decide:true` … 更新決定チェックの［決定］ボタンに出すもの（4つ）。
  申請依頼済み・完了は決定の対象ではないので付けていない

#### 📌 更新リストの上を貼り付ける

- タブの行（`#wsTabsBar`）→ 絞り込みの行（`#renStickyBar`）→ 一覧の見出し行（`.ren-sticky`）の
  3段を重ねて貼り付ける。**位置と高さを測るのは `_renStickTop()` の1箇所だけ**
  （`.topbar` と各行の高さから出すので、絞り込みが折り返しても崩れない）
- `#renTableArea` に `max-height` を入れて**一覧の中だけが縦にスクロール**する形にしてある。
  こうしないと、見出し行の `position:sticky` が画面ではなく箱に対して効いてしまう
- 「すべて開く／閉じる」と見出し行は**`.ren-sticky` の1つの箱にまとめて**貼り付ける
  （別々にすると重なる位置を2つ計算することになる）
- 描き直したあと（`renderRenewalList` の最後）とタブを開いたとき、`resize` のときに測り直す

#### 📝 更新決定のしくみ（チェック → 決定 → 承認 → チャット → 毎月のサイクル）

- **確認事項（質問・選択肢・出す条件・必須かどうか）は `REN_CHECK_ITEMS` の1箇所だけ。**
  `when` を書いた欄はその条件のときだけ出て、出ていない欄は未入力に数えない（`_renCheckVisible`）
  - 実体は **`renewal_checks`**（人材 × 対象月で1件）。対象月を決めるのは **`_renTargetYm(w)` の1箇所**
    ＝ 在留期限の `REN_TARGET_MONTHS_AHEAD`(4) か月前の月
  - **退職予定日と一時帰国日は人材情報にも書き戻す**（対応は **`REN_CHECK_TO_WORKER` の1箇所**＝
    `resign_expected_date`→`expected_leave_date` / `temp_return_date`→`return_home_date` /
    `temp_return_back_date`→`return_jp_date`）。
    欄とチェックシートが必ず同じものを指すようにするため
  - 記入できるのは**メイン／サブ担当と管理者**（`_renCheckCanEdit()` の1箇所）
  - **1つでも空いていると決定のボタンは押せない**（`_renCheckMissing()` の1箇所）
  - 収入印紙費用の負担パターンは **`VISA_FEE_PATTERNS` の1箇所**（下の節）。
    `REN_FEE_PATTERNS` はそこから作るだけ

### 📋 所属機関の【決定報告履歴】タブ（財務情報の左）

- **読むだけの画面**。どのテーブルにも書き込まないので、これまでの機能・紐づきには一切さわらない
- **元データは大房の `intake_requests` の1箇所だけ**（`ofusaFetch` で読む）。
  KMT案件（スプレッドシート送信）も大房のGASを通ってここに入るので、**過去分もこれからの分も同じ一覧**に出る
  - 読む列は **`DR_HIST_COLS` の1箇所**（存在しない列を書くと 400 で落ちる）
  - 企業名は表記ゆれがある（「第一舗道(株)」「第一舗道株式会社」）ので、
    **そろえ方は `_drCoNorm()` の1箇所だけ**（会社の種類・記号・全半角を落とす）。
    大房へは種類を落とした名前で `ilike` して、返ってきたものを `_drCoNorm` で絞り込む
  - 状態の言い換えは **`DR_HIST_STATUS` の1箇所**（pending／registered／rejected）
- 行を押すと中身を確認できる（`openCoDecisionDetail`）。上に**そのときの決定報告の文面**、
  下に受信トレイの中身（**出す項目は `DR_HIST_DETAIL` の1箇所だけ**）
- **文面は `decision_report_log` に控える**（KMT側の新テーブル）。
  **書き込むのは `_saveDecisionReportLog()` の1箇所だけ**で、
  スプレッドシート送信／大房送信が**成功したあとに1件足すだけ**。送り先も紐づけも何も変えていない
  - 文面は `buildKmtReportText()` ／ `copyOtherReport(true)` ／ `copySoloReport(true)`。
    引数 `true` を渡すと**コピーせず文面だけ返す**（コピーボタンの動きは今までどおり）
  - 受信トレイの1件と控えを結び付けるのは **`_drLogFor(r)` の1箇所**（人材名＋受付日が近いもの）
  - **控えが無い古い分は `_drRebuildReportText(r)` が受信トレイの内容から組み立てて出す**
- **請求条件・雇用条件・コメントは依頼スプレッドシートにしか無い**ので、受信トレイの
  `sheet_url` と `row_number` から**その行を読んでそのまま出す**（`_drFillSheetRow()`）
  - 読み方は申請記録と同じ **gviz の CSV**（`_drLoadSheet()`。読むだけ・シートには書かない）。
    同じシートは1回読んだら覚える（`_drSheetCache`）
  - 行は `row_number` を見て、人材名が合わなければ全体から探す（**`_drFindSheetRow()` の1箇所**）
  - ⚠️ **シートが「リンクを知っている全員が閲覧可」でないと読めない**。読めなかったときは
    その旨とシートへのリンクだけ出す（黙って空にしない）
- 読み込みは `loadCoDecisionHistory()`。**タブを開いたときだけ**引き、同じ企業なら2回目は引き直さない
  （［🔄 更新］で取り直す）

### 📑 所属機関詳細の見出しの通し番号

- 見出し（`.section-title`）に **1. 2. 3. …** の通し番号を出す。タブをまたいで通しなので、
  「◯番の項目」で1つに決まる（1〜7＝基本情報／8＝契約・委託費／9〜10＝財務情報／11〜14＝メモ）
- **番号を振るのは `numberCoSections()` の1箇所だけ**（画面に並んでいる順そのまま）。
  `openCompanyModal` から呼ぶ。**番号を手で書かないこと**（見出しを足すと自動でずれる）
- 2回目からは付け直さない（`data-co-no` を見る）

### 🔁 所属機関の担当の変更履歴（引継ぎの記録）

- 場所は **所属機関の詳細【基本情報】タブ → 🏢 KMT担当 → 社内メモの上**
- 実体は **`companies.staff_handovers`**（jsonb の配列。別テーブルを作らない＝企業の保存と一緒に入る）。
  1行＝1回の引継ぎで、`{date, kind, from, to, method, note, at, by}`
- **選択肢は `HANDOVER_KINDS`（区分）／`HANDOVER_METHODS`（引継ぎ方法）の1箇所だけ**。
  前の担当・新しい担当は `<input list="coStaffDl">` なので**名簿の候補から選べて手入力もできる**
  （候補を作るのは `_coStaffDatalistHtml()` の1箇所＝`KMT_STAFF_ROSTER` ＋ `TEAM_STAFF_DEFAULTS`）
- 読み書きは **`renderCoHandovers()` / `getCoHandovers()` の2つだけ**。並びは**引継ぎ日の新しい順**、
  **中身が何も入っていない行は保存時に落とす**。［＋ 引継ぎを追加］は先頭に今日の日付で1行足す
- **担当の最終更新（日時・ユーザー名）は `loadCoStaffMeta()` の1箇所**が
  **`company_change_log`** から引く（`CO_STAFF_LOG_FIELDS` ＝ メイン／サブ／チーム内サブ／引継ぎ予定）。
  **列を別に持たない**ので、【変更履歴】タブの内容と必ず一致する。
  出すのは チーム内サブ担当の下（`co_staff_meta`）で、モーダルを開いたときと保存したあとに描き直す

### 💴 入管申請費用（収入印紙代）の負担パターン

- **選択肢は `VISA_FEE_PATTERNS` の1箇所だけ**（A＝受け入れ企業全額負担 ／
  B＝特定技能人材の自己負担（本人へ直接請求）／C＝受け入れ企業にて全額お支払い後、本人負担分を回収）。
  **保存するのは `A` / `B` / `C` の1文字**（更新チェックシートに入っている値と同じ）
- 画面に出す文字（「パターンA：【…】」）を作るのは **`visaFeePatternText(v)` の1箇所**、
  選択欄を作るのは `visaFeePatternOptionsHtml(cur)` の1箇所
- 出る場所は3つ。**どれも同じ選択肢**
  | 画面 | 欄のid | 保存先 |
  |---|---|---|
  | 所属機関の詳細（🏢 KMT担当の上） | `co_visa_fee_pattern` / `co_visa_fee_note` | `companies.visa_fee_pattern` / `visa_fee_note` |
  | 決定報告（請求条件の下・3つのフォームすべて） | `dr_` / `ot_` / `sl_` + `_visa_fee_pattern` / `_visa_fee_note` | 報告テキストとコメント欄に出す |
  | 更新チェックシート | `fee_pattern`（`REN_FEE_PATTERNS`） | `renewal_checks` |
- ⚠️ **所属機関の欄は画面に直に書いてある `<select>`** なので、選択肢は
  `fillVisaFeeSelects()` が入れる（`class="visa-fee-sel"` を見る）。
  **`openCompanyModal` の先頭で `fillCoForm` より前に呼ぶこと**（後だと選んだ値が入らない）。
  決定報告はテンプレートで作るので `${visaFeePatternOptionsHtml('')}` を直接書いている
- 決定報告は**大房の `intake_requests` には送らない**（列が無い。存在しない列を混ぜるとINSERTが落ちる）。
  報告テキストとコメント欄にだけ入れる
- **`workers.renewal_status` を書くのは `_renWriteStatus()` の1箇所だけ。**
  入口は `applyRenewalStatus()` で、人材詳細の選択欄（`onRenewalStatusChange`）も
  チェックシートの決定（`decideRenewalFromCheck`）も必ずここを通る＝**履歴が必ず残る**
  - 履歴は **`renewal_status_log`**（いつ・だれが・何から何へ・承認者）。
    人材詳細（`renderWorkerRenLog`）とチェックシート（`renderRenewalLog`）に出る
  - ⚠️ 人材詳細の選択欄は、承認が要るときは**元の値に戻す**（DBと画面が食い違わないように）
- **更新の管理者は `REN_APPROVERS` の1箇所だけ**（いまは ニサ・白井・アヒュ）。
  **システムの管理者（`isAdminUser()`）とは別あつかい**で、ここに書いた人は管理者でなくても
  ①更新ステータスの変更を承認できる ②毎月の依頼をかけられる（`REN_CYCLE_ADMINS` は同じ配列）
  ③どの人材のチェックシートでも記入できる。**更新の件だけ**で、ほかの管理者向け画面には効かない
  - 判定は **`_renNameIn()` の1箇所**。名簿が「白井　美紗」でここが「白井」のように
    **名字だけ／フルネームのどちらでも合うよう前方一致でも見る**（完全一致だけだと外れる。実際に外れた）
  - マイページの【✅ 承認】タブは `canApproveChatLeave() || _renIsApprover()` で出す
  - サイドバーの承認待ちのお知らせ（`updateApproveAlerts`）は、中身に合わせて見出しを変える
    （更新だけ／離脱だけ／両方）
- **承認が要る組み合わせは `REN_APPROVAL_MATRIX` の1箇所だけ**（左＝いまの値／右＝変えたい値）。
  判定は **`_renNeedsApproval()` の1箇所**で、**一度も決定していない人材（初回）は承認なし**
  - 申請は **`renewal_status_requests`**（チャット離脱の承認と同じ作り）。**理由の記入が必須**
  - 承認できるのは **`REN_APPROVERS` の1箇所**（いまは ニサ・白井）。
    管理者のマイページ【✅ 承認】にチャット離脱の承認と**並べて**出す（`renderRenewalApprovals`）。
    承認待ちの件数（`updateApproveAlerts`）は両方を足したもの
  - **承認されるとその人材のチャットに `@全員` で投稿する**（文面は `_renPostStatusChat()` の1箇所）。
    `mentions` には全員の名前を展開して入れる
- **毎月のサイクル**。日付を出すのは **`_renCycleDates(ym)` の1箇所**
  ＝ その月の最終営業日（期限）と、その `REN_CYCLE_LEAD_DAYS`(14) 日前の営業日（開始）。
  土日だけを見て祝日は見ない（ほかの平日計算と同じ）
  - 段階は **`_renCycleState()` の1箇所**（before／admin＝管理者が依頼をかける番／open／over）
  - 依頼をかけられるのは **`REN_CYCLE_ADMINS` の1箇所**（ニサ・白井）。
    押した記録は `renewal_month_meta.requested_at` / `requested_by`
  - **対象の決め方は `REN_TARGET_MONTHS_AHEAD` まわりの一かたまりだけ**（`_renCycleTargets`）
    - メイン … 在留期限が「サイクル月 ＋ `REN_TARGET_MONTHS_AHEAD`(5) か月」の人材。
      **決定から在留期限まで4か月以上あける**ための5か月
    - 繰り越し … 前の `REN_CARRY_MONTHS`(1) か月ぶんのうち、**まだ申請が動いていないもの**
      （`REN_CARRY_VISA`＝特定技能1号 ／ `REN_CARRY_APPLY`＝空か「申請予定なし」）は
      次の回にも残す。判定は **`_renIsCarry()` の1箇所**で、行には `_carry` の印が付く
    - 繰り越しは**前に決めていても「残り」に入れる**（決め直しが要るため）。
      うるさければ `REN_CARRY_SKIP_REN` に更新ステータスを足せば外れる（いまは空）
  - **全員決まるまでアラートは消えない**。期限を過ぎたら `.due-over` で赤く点滅
  - アラートの置き場はサイドバー `renCycleAlerts` ／ マイページ `myRenCycleAlerts`
#### ⏰ 申請依頼まち（更新決定のまま止まっている人材）

- **在留期限まで `REN_APPLY_ALERT_DAYS`(75)日を切っているのに、更新ステータスが
  【更新決定】のまま**＝まだ【申請依頼済み】になっていない人材を知らせる（依頼もれ・変えもれの防止）
- **判定は `_renApplyLate(w)` の1箇所だけ**。対象の状態は **`REN_APPLY_LATE_FROM` の1箇所**
  （更新決定（通常）／更新決定（優先度高め）だけ。申請依頼済み・完了・更新なし・要確認は出さない）。
  旧い値（`更新決定` / `優先度高め`）も `_renStatusNorm()` を通すので拾える
- 期限を過ぎているものも出す（`_renDaysToExpiry()` がマイナスを返す）。
  **30日を切っている人がいると赤く点滅する**（`.due-over`）
- 出す相手は**自分がメイン／サブ担当（または人材の担当）のぶん**（`_renApplyLateMine()`）。
  0名のときだけ、更新の管理者（`REN_APPROVERS`）に全体を出す
- 置き場はサイドバー `renApplyAlerts` ／ マイページ `myRenApplyAlerts`（`updateRenApplyAlerts()`）。
  一覧は `openRenApplyLate()`、行を押すとその人材の詳細が開く
- ブラウザ通知は `_renApplyNotify()`。**同じ日に何度も出さない**（面談記録の記入待ちと同じ作り）
- 数え直すのは 起動時（`loadRenewalCycle`）と **ステータスを書いたとき（`_renWriteStatus`）**
- 🧪 テスト用の人材と、辞退・支援機関変更済みの人材（`isAlertExcludedWorker`）は出さない

- 🧪 **テスト用の人材は氏名が `TEST_WORKER_PREFIX`（`【テスト】`）で始まる行**。
  **判定は `_isTestWorker()` の1箇所だけ**で、本番にさわらずに流れを試すためのもの
  - **だれでも記入・決定できる**（`_renCheckCanEdit` の先頭）／**だれのリストにも出る**（`_renCycleMine`）
  - **在留期限・更新ステータス・在職ステータスが何であっても、いつでも一覧のいちばん上に出る**
    （`_renCycleTargets` / `_renCycleUndecided` で `_test` の行を素通しする）。
    決定しても月が変わっても、いつでも試せるようにするため
  - ⚠️ **そのぶん件数からは必ず外す**（`_renCycleRealCount()` の1箇所）。
    「残り」もアラートもこれを通すので、テストが残っていても本番が片づけばお知らせは消える
  - 承認後のチャットは **@全員 にしない**（`_renPostStatusChat` の中で宛先を申請者と承認者だけにする）
  - `resetRenewalTest()` でステータス・チェックシート・記録・申請を消して何度でも試せる。
    **消すのはテスト用の人材のぶんだけ**（`worker_id` で絞る）
  - 実体は本物の `workers` の行（`【テスト】更新テスト 太郎`／`【テスト】更新テスト社`）。
    アラートに出ないよう必須項目は埋めてある
- ⚠️ **更新決定のモーダルは人材詳細の上から開くので、z-index を上げてある**
  （`#renCycleModal` 1100 → `#renCheckModal` 1110 → `#renGuideModal` 1120）。
  **上に出すモーダルの z-index は `.modal-overlay` の下の1箇所にまとめて書く**
  （あちこちに散らすと、どれが上か分からなくなる）
- **手順の文章と図は `REN_GUIDE_STEPS` / `_renGuideSvg()` の1箇所だけ**（`openRenewalGuide`）。
  図は SVG を直に書く（ライブラリを増やさない）。仕組みを変えたらここも直すこと

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
  - ⚠️ **そのぶん、開いて閉じただけでは `tg2_expiry` がDBに書かれない。**
    以前これで一覧の「5年満了日」が「-」のままの人材が182名いた（実際に起きた）。対策は2つ:
    1. **一覧などで5年満了日を読むのは `worker5yExpiry(w)` の1箇所だけ。**
       `tg2_expiry` が空なら `tokutei_history` から同じ式で計算して返す
       （人材一覧・更新状況リスト・並べ替えが同じ値になる）
    2. モーダルを開いたとき `_fillBlankTg2Expiry()` が、**8番が空のときだけ**
       10番の計算結果をDBにも入れる。**すでに値が入っている人材は触らない**
       （手で直したものを勝手に書き替えないため）。`updated_at` / `updated_by` も書かない
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

### 🏷️ 人材情報の【分野】と【職種】（検索できる選択式）

- **分野と職種の一覧は `SSW_FIELDS` の1箇所だけ**（特定技能の19分野 → その分野の業務区分）。
  分野を選ぶと**その分野の職種だけ**が出る（`sswJobOptions(field)`。分野が空なら全部＋分野名を添える）
- ⚠️ 制度の見直しで分野・業務区分は増える。**足すときは `SSW_FIELDS` だけ直す**
- 欄は **`swPickHtml(id, value, ph, cfg)` の1組**（検索できる自前のパネル。
  `<select>` の中には検索欄を置けないため）。`swPickOpen` / `swPickRender` / `swPickSet` / `swPickFree`
  - **選んだ値は隠しの `<input id="wf_applicant_field">` に入れる**ので、
    保存（`saveWorker`）も未保存判定（スナップショット）も今までどおり
  - **一覧に無い値（前から入っている値）は消さない**＝ パネルの先頭に「いまの値」として残る。
    ［✏️ 一覧にない値を入れる］で手入力もできる（移行は不要）
  - ⚠️ ボタンの class は **`.sw-pick-btn`**（`.form-input` にしない）。
    `.form-input` にすると `markEmptyField()` が薄赤を勝手に外す
  - 未記入の薄赤は `swPickSet()` が `field-empty` を付け外しする（`cfg.req` のときだけ）

### 👤 外国人材【8. 支援情報】の担当を選ぶ欄

- 事前ガイダンス実施担当名／オリエンテーション実施担当／入国・出国の担当者名は**選択式**。
  **選択肢を作るのは `workerStaffOptionsHtml(cur, w)` の1箇所だけ**
  ① その人材の所属機関の **メイン → サブ → チーム担当**（名簿にいない名前でも先頭に出す）
  ② 続けて **GLT と 営業** のユーザー（①に出したぶんは重ねない）。大房チーム・総務は出さない
- **いまの値が①②のどちらにも無いときは、その値も選択肢に残す**（担当が変わっても消えないように）。
  名前の**全角／半角スペースの違いは同じもの**としてあつかう
- 面談記録の `meetingStaffOptionsHtml()` と考え方は同じだが、**チーム担当まで見るのはこちらだけ**
  （面談記録の対応者はメイン・サブだけ）

#### 🛫 出国：担当者名（KMT担当＋その他）

- 出国の対応も**登録支援機関の業務**なので、**KMT担当を選ばずに「その他」だけにはできない**。
  判定は `_departStaffOk()` の1箇所で、`saveWorker` の先頭で止める（欄の下の注記も赤くする）
- **保存先は今までどおり `workers.departure_staff` の1列**。KMT担当とその他を
  `DEPART_STAFF_SEP`（`／`）でつないで入れる。読み書きは
  **`_departStaffParts(v)` / `_departStaffValue()` の2つだけ**を通す
- 入国：担当者名は今までどおり選択だけ（その他の欄は出国だけ）

### 📅 10. 通算期間と過去の就労先

#### 🛫 母国滞在（中断）と 5年満了日

- 特定技能1号の途中で本帰国し、あとから在留資格認定で入り直した場合、
  **母国にいた期間は通算期間に入れない**。入れ方は2つあり、**どちらでも結果は同じ**
  1. **🛫 母国滞在の行を足す** … 在留資格に `TOKUTEI_ABSENT`（`（母国滞在・不在）`）を選び、
     日付に**帰国した日**を入れる。`_isTokuteiVisa()` が false なので通算に足されない
     - **使うのは【本帰国】のときだけ**（日本の住所＝住民票を抜いて帰国し、あとから認定で入り直した場合）。
       一時帰国（再入国許可）は通算に入るので、こちらではなく「一時帰国」の欄に日付を入れる
     - 取り違えないよう、**記入する前に確認を出す**。文章は **`TOKUTEI_ABSENT_CONFIRM` の1箇所だけ**で、
       ［🛫 母国滞在を追加］と在留資格の欄から選んだときの**両方**が同じものを通る
  2. **その行に終了日（`end`）を入れる** … そこで打ち切り、次の指定書発行日までが空白になる
- **計算は `calcTokuteiSummary()` の1箇所だけ**。1行の期間は「発行日 → 次の行の発行日」で、
  `end` があればそこで打ち切る。`days`＝通算に入る日数／`gapDays`＝入らない日数

#### 🍼 休業等（通算に含めないことができる期間）

- 入管庁の規定（<https://www.moj.go.jp/isa/10_00233.html>）で通算在留期間に含めないことが
  できる期間。**選択肢は `TOKUTEI_LEAVE_KINDS` の1箇所だけ**＝
  産前産後休業／育児休業／病気・怪我による休業／再入国できなかった期間
- ⚠️ **除外されるのは在留諸申請で申立書・疎明資料を出して許可されたときだけ**なので、
  **行ごとに「含む／除外」を選ぶ**（`r.count`。**既定は `'in'`＝含む**。安全側）。
  切り替えは `toggleTokuteiCount()`、判定は **`_tkCounted(r)` の1箇所**
- **含む → 除外 のときだけ `openTokuteiApv()` を開き、申立書の状況とメモを入れてもらう**
  （除外 → 含む はそのまま戻す。状況とメモは消さずに残す）
  - 状況の選択肢は **`TOKUTEI_APV` の1箇所だけ**（`done`＝承認済み／`yet`＝まだ）。
    行に `r.apv` / `r.apvNote` で持つ（`tokutei_history` の中）
  - **承認がまだのまま除外している行は `_tokuteiWarnings()` が注意を出す**
  - 入れた内容は **`_tokuteiLeaveListHtml()` の1箇所**が計算結果の下に一覧で出す
    （行のボタンの title と同じ内容）
- **休業が終わった（`end` を入れた）あとは、直前の就労（特定技能）の続きとして計算する**。
  復職では指定書が出ないので行が増えないため。休業が続いているときは
  `_isTokuteiLeave` の行を遡って、その前の就労先を見る
- 入口は2つ（［🍼 休業期間を追加］／在留資格の欄から選ぶ）。**どちらも
  `TOKUTEI_LEAVE_CONFIRM` の案内を出してから**入る（母国滞在と同じ作り）
- **入管庁のリンクは `TOKUTEI_LINKS` の1箇所だけ**（`renderTokuteiLinks()` が
  入力欄のすぐ下に並べる）

#### ⭐ 通算在留期間 6年（特定技能2号評価試験等に不合格）

- **年数は `TOKUTEI_YEARS`(5) / `TOKUTEI_YEARS_SIX`(6) の1箇所だけ**。
  `calcTokuteiSummary(rows, { sixYear })` が満了日の計算に使い、`s.years` を画面のラベルに出す
- **満了日は5年ぶんと6年ぶんを両方返す**（`s.byYear[5]` / `s.byYear[6]`。作るのは `mk()` の1箇所）。
  画面では2つ並べて、いま効いているほうに【★いまの設定】、もう一方に（参考）を付ける
  ＝ 5年と6年の違いがその場で分かる。`s.expiry5` / `s.over` / `s.remain` は**効いているほう**の値
- 保存先は **`workers.tokutei_six_year`**（boolean）。読むのは **`isTokutei6y(w)` の1箇所**で、
  `worker5yExpiry(w)` と一覧のバッジ（`tokutei6yBadge()`）が同じものを見る。
  軽いSELECTでも読めるよう `WORKER_ALERT_SELECT_COLS` に列を入れてある
- チェックを**入れるときだけ** `TOKUTEI_SIX_CONFIRM` の案内を出す（許可された人だけに付けるため）
- **入口は就労先の行の「6年」（`r.six`）だけ**。「初回」と同じで**1行だけ**選べる。
  書き込むのは **`setTokuteiSix()` の1箇所**、読むのは **`_tokutei6yOn()` の1箇所**
  （＝どれかの行に `six` があるか）。保存は `tokutei_six_year` の1列
  - ⚠️ 以前は計算結果の下にもチェック欄（`wf_tokutei_6y`）があったが、**同じものが2つあって
    分かりにくかったので外した**。増やし直さないこと
- ⚠️ 行の「6年」は **`.tk-row` の列を増やさず、「初回」と同じ列に縦2段**で置く
  （列を1つ足しただけで横スクロールしないと見えなくなった）
- 一覧の列名は「5年満了日」のままで、6年の人には日付の隣に【6年】の印を出す（列名を増やさない）

### 🔍 外国人材【基本情報】タブの項目検索

- タブのすぐ下の `#wfFindBox` に項目名を打つと候補が出て、選ぶとその欄まで動いて黄色く光る
- **探す対象は `.section-title` と `.form-label` の2つだけ**（`_wfFindCollect()` の1箇所）。
  項目名を別に持たないので、欄を足せば検索にも自動で出る
- 見るのは **`#wtab_basic_panel` の中だけ**。光らせ方は `wfFindGo()` の1箇所
- ⚠️ 検索欄の id は **`wf_` で始めない**（`wf_*` は保存対象・未保存判定の対象になるため）
- **5年満了日は「通算が5年に達する日」**＝ `今日 ＋（起点＋5年 − 通算ぶんを積んだ仮の到達日）`。
  中断があるとそのぶん後ろにずれる。**中断が1日も無ければ「起点＋5年」と同じ日**になるので、
  いままでのデータは値が変わらない（特定活動をはさむ人だけ後ろにずれる＝正しくなる）
- **入力の食い違いを知らせるのは `_tokuteiWarnings()` の1箇所**。
  2行目以降の［認定］は「いったん出て入り直した」合図なので、直前に空白が無ければ注意を出す
- **就労先の行の「申請」が未記入だと薄赤になり、必須項目のアラートにも出る。**
  数えるのは **`tokuteiApplyMissing(rows)` の1箇所だけ**（一覧のアラート `missingRequiredFields` ／
  保存時の確認 `missingRequiredFieldsFromForm` ／ 欄の薄赤 が同じものを見る）。
  🛫 母国滞在・🍼 休業の行と、指定書発行日が空の行は数えない
  - ⚠️ 薄赤は class を直に書いても `markEmptyField()` に消される。
    **id を持たない欄は `data-req="1"` を付ける**（`_wfIsRequiredField()` がこれも見る）
- 行の列の定義は **`.tk-row` のCSS 1箇所だけ**（見出しと行が必ずそろう）。
  欄が多いので狭い画面では横スクロールになる（モーダルの ⛶ 全画面で広くできる）

#### 🏢 在留歴の2つの枠（特定技能2号 ／ 特定技能以外）

- セクションの下に枠が2つ。**入力する項目は同じ**で、在留資格の欄が出るかどうかだけが違う
  | 枠 | 列 | 在留資格の欄 |
  |---|---|---|
  | 🏢 特定技能2号の履歴 | `workers.tokutei2_history` | なし（2号と分かっているため） |
  | 🏢 過去の所属又は勤務先（特定技能以外） | `workers.past_employment` | あり（`PAST_EMP_VISA_OPTS`） |
- **枠の定義は `EMP_HISTORY_KINDS` の1箇所だけ**（入れ物・見出し・placeholder・在留資格の有無）。
  描く・足す・消す・直す・詳細を開くのは `renderEmpRows(kind)` ほか **kind を受け取る関数1組だけ**
- `renderPastEmpRows()` などの旧い名前は `'past'` を渡すだけの包みとして残してある
- **どちらの枠も通算期間の計算には入らない**（10番の就労先の行だけが計算対象）

#### 並び順

- 就労先の行は**指定書発行日の古い順に上から**並ぶ。**並べ替えは `_tokuteiSortRows()` の1箇所**で、
  `renderTokuteiRows()` が毎回呼ぶのでバラバラの順で入れても自動でそろう
- **日付が空の行は末尾**（＋追加した直後の行が上に飛ばないように）。
  同じ日付なら入れた順のまま（`sort` が安定なので）
- **並べ直すのは日付を直したときだけ**（`updateTokuteiRow` の `key === 'date'`）。
  会社名は `oninput` で入ってくるので、そこで描き直すとフォーカスが飛んで打てなくなる

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

### 🌐 外部フォームの翻訳（履歴書作成リンク）

- 本人が読む画面なので、**日本語を消さずに下へ訳を並べる**（担当者も確認できるように）
- **言語の一覧は `EXT_FORM_LANGS`、文言の対応表は `EXT_FORM_I18N` の1箇所だけ**
  （英・インドネシア・ベトナム・タイ・クメール・ミャンマー・中国語）。
  **対応表のキーは画面に出ている日本語そのまま。文言を変えたらキーも直すこと**
- 通信もAPIキーも使わない**ただの置き換え**（外に出す画面なので、キーを置けない）。
  社内の求人票翻訳（`translateFlyer`）はAnthropic APIを使うが、あれは社内画面だけ
- `applyExtFormLang()` が1箇所で全部やる
  - ふつうの文字 … テキストノードの後ろに `.i18n-tr` を1行足す
  - `option` / `placeholder` … 行を足せないので「日本語 / 訳」にする
    - ⚠️ **`value` 属性が無い `option` は、見た目を変えると `value` まで変わる。**
      訳を足す前に `o.setAttribute('value', o.value)` で固定しておくこと。
      これを外すと「男 / Male」がそのまま保存される（実際に起きていた）
  - 案内文のように**1文にまとめたいところは `data-i18n`** を付ける（中の `<b>` で分断されないように）
- 職歴・資格・資料の欄を描き直したあとは `reapplyExtFormLang()` で付け直す
  （`renderWorkHistoryInputs` などの末尾から呼んでいる）
- 選んだ言語は `localStorage`（`kmt_extform_lang`）に覚える
- 紺色の見出しと青いボタンの中だけ、訳の色をCSSで白系にしている（青のままだと読めない）

### 🪪 希望在留期間（決定報告・新規依頼・発注書で共通）

- **選択肢は `STAY_PERIODS_3Y` / `STAY_PERIODS_5Y` / `STAY_PERIODS_BY_VISA` の1箇所だけ**。
  特定技能1号・2号・特定活動＝3ヶ月〜3年／技人国・家族滞在＝3ヶ月〜5年。
  在留資格が未選択のときは広いほう（5年まで）を出す（`stayPeriodsFor()`）
- 欄を作るのは **`stayPeriodFieldHtml(id, visaId, num)` の1箇所**。置き場所は
  **「5.【A区分】希望在留資格」のすぐ下**（KMT案件・新規依頼は番号つきで `6.`。
  以降の番号はそのぶん繰り下げてある）
- **選択肢を作り直すのは欄を開いたとき（`onfocus` → `syncStayPeriod()`）**。
  在留資格の `<select>` のほうに onchange を足して回ると、直す場所が増えてズレるため。
  いま入っている値がその資格でも選べるならそのまま残す
- 保存先は大房の **`intake_requests.hoped_stay_period`**（「1年」のように文字のまま）。
  KMT案件はスプレッドシート送信なので `getDecisionData().stay_period` に入れ、
  **コメント欄にも「希望在留期間: 〇〇」を足す**（シート側に列が無くても届くように）

### 📥 大房の「＋新規案件依頼」に必要な項目

他社支援・企業単独の送信セクション（**`ofusaSendSectionHtml` / `ofusaSectionValues` の1箇所**）に、
大房のモーダルの必要項目をひととおり入れてある。

| 大房の項目 | こちらの欄 | 列 |
|---|---|---|
| 希望在留資格 | 希望在留資格 | `desired_status` |
| 希望在留期間 | 希望在留期間 | `hoped_stay_period` |
| 現在の在留資格 | 現在の在留資格 | `residence_status` |
| 案件担当者① | 案件担当者①（企業側） | `client_staff` |
| 案件担当者② | 案件担当者②（登録支援機関側） | `support_staff` |
| 人材紹介手数料 | 紹介料／初期費用＋申請費用 | `intro_fee` |

- `agency_office` / `aidem_branch` / `toritsugisha` は**アイデム等の他社ぶんの列なので使わない**
- 他社支援は人材名が複数行になるが、**国籍・在留期限・現在の在留資格・希望在留期間は
  フォーム全体で1つ**（1名ずつ変えたいときは行を分けて送る）

### 👤 候補者 → 決定報告

- 決定報告の3フォーム（KMT案件／他社支援／企業単独）に「👤 候補者から自動入力」を出す。
  **出る候補者は `DR_CAND_PICK_STATUSES`（結果待ち・内定）だけ**
- **どの欄に何を入れるかは `DR_CAND_FILL` の1箇所だけ**（フォームが増えたらここに足す）。
  すでに値が入っている欄は上書きしない（氏名だけは必ず入れる）
- 企業名・分野・登録支援機関は、その候補者に紐付いた求人案件（`_drJobOfCandidate()`）から取る。
  登録支援機関は `job_progress.org_code` → `allIntroRegOrgs` で名前を引く（名前の列は無い）
- **選択肢の全角／半角の数字違いは同じものとして扱う**（`_drMatchOption()`）。
  候補者は「技能実習２号」、決定報告は「技能実習2号」で表記がそろっていないため
- **決定報告の行き先は `_drProceed()` の1箇所**（①KMT案件／②紹介案件の並びは `DR_ROUTES`）。
  内定の案内ダイアログからも、候補者のステータス欄に残るボタンからも同じものを通す
- **「あとで」を押しても、①②のボタンは候補者情報タブのステータスの下に残る**
  （`_cndRenderDrPanel()`。ステータスが内定のあいだはずっと出る）。
  ボタンの隣の（済）は `candidates.dr_done`（jsonb `{kmt:{done,at,by}, intro:{...}}`）に
  **押したその場で** `setDrDone()` が書く（モーダルの保存は要らない）。
  軽いSELECTで開いたときは `dr_done` が無いので、その1行だけ取り直してから描き直す
- ステータスが**内定になった瞬間**に `offerDecisionReport()` が案内を出す
  （①KMT案件の決定報告へ ②（紹）決定者リストに入れてから紹介案件の決定報告へ）。
  入口は `onCandidateStatusChange` / `cjSetLinkStatus` / `jpUpdateLinkStatus` の3つで、
  判定は `_maybeOfferDecisionReport()`（前の値が内定なら出さない）
- （紹）決定者リストには「📋 決定報告に進む」がある。**保存だけ押されたときは
  `saveIntroDecision` が1回たずねる**（進む／進まない）。進む先は他社支援か企業単独を選ぶ
- 画面を移ってから流し込むので、`goDecisionReport(type, candId)` が `_drPendingType/_drPendingCand`
  に入れ、`loadDecisionReport()` の最後で `pickDrCandidate()` する
- 2〜3択のダイアログは **`askChoice()` の1箇所**（Promiseで押したキーが返る）。
  ⚠️ **出す前に必ず `showLoading(false)` を呼ぶ**（`askChoice` の先頭でやっている）。
  さらに **`#choiceModal` の z-index は「読み込み中」の覆い（9999）より上**にしてある
  （下だと、覆いが残っているあいだボタンが押せず、リロードするしかなくなる。実際になった）
- **「読み込み中」の覆いは `LOADING_MAX_MS`(20秒) で自動的に消える**（`showLoading` の1箇所）。
  どこかで消し忘れても画面が固まったままにならないための保険。
  ほかのモーダルの上に出すため `#choiceModal` だけ z-index を上げてある

### ⚖️ 求職管理簿（候補者一覧の中の表示切り替え）

- 候補者一覧の**件数チップの右側**にある「表示」で `📋 候補者一覧` ⇄ `⚖️ 労働局 求職管理簿`
  を切り替える（`setCandView`）。求人管理の管理簿と同じ置き方。上のタブには出さない
- **上の検索・ステータス・国籍の絞り込みにそのまま追従する**（`_candLedgerRows()` は
  `filteredCandidates` を見る）。管理簿だけの検索欄は持たない。
  印刷・Excel出力も**いま絞り込んでいるぶん**だけ出す
- 労働局に出す様式そのままの一覧。**中身は候補者一覧と同じ `candidates` の行**なので、
  どちらで直しても両方に出る（別テーブルを作らない）
- 列順は様式なので**勝手に変えない**（①②③氏名・住所・生年月日／④希望職種／⑤受付年月日／
  ⑥有効期間／⑦職業紹介の取扱状況／備考）
- **セルをクリックするとその項目の編集画面が開く。** 対応表は **`CAND_LEDGER_EDIT` の1箇所だけ**。
  入口は `candLedgerEdit(candId, col)` →`openCandidateModal(id, focusId)` →`_cndFocusField()`
  （`rf_` 始まりなら履歴書タブを開く）。文字を選んでいる間は開かない
- ⑦は `candidate_job_progress` から作る（`_candLedgerJobRows`）。紐付けが複数あれば行が分かれ、
  左側は rowspan でまとめる。**採否結果は候補者のステータスから決める**
  （内定＝採用／見送り・辞退＝不採用。`_candLedgerResult()` の1箇所）
- **求人管理の「⚖️ 労働局 求人管理簿」と行き来できる。**
  求職管理簿の求人受理番号・企業名 → その求人案件（`candLedgerOpenJob`）／
  求人管理簿の求職者氏名 → その候補者（`_ledgerCandLink`）
- 転職勧奨禁止期間・6か月以内の離職状況は**手書き用の空欄**（DBに列を持たせていない）
- 印刷（A4横）と Excel出力あり（`printCandLedger` / `exportCandLedgerExcel`）

### 💰 手数料管理簿（求人管理の「表示」の3つ目）

- 求人管理の右上「表示」で `KMT管理リスト` / `労働局 求人管理簿` / `手数料管理簿` を切り替える（`setJobView`）
- **元データは売上リスト（`sales_forecast`）**。1行＝1件の手数料で、行を押すと
  その売上の編集画面（`openSaleModal`）が開く＝売上リストとどちらで直しても同じ
- **② 徴収年月日は売上リストの「請求」チェックの日（`billed_date`）。**
  チェックが無い行は「まだ徴収していない」ので既定では出さない（チップで出せる）
- **既定は「請求チェックが入っている売上を全部」出す。** 実際の売上は
  支援委託費・初期費用などの名前で入っていて、紹介手数料の名前（`FEE_LEDGER_ITEMS`
  ＝人材紹介料・紹介手数料・求人受付手数料）だけに絞ると**1件も出なくなる**（実際に0件になった）。
  紹介手数料だけ見たいときは「手数料の項目だけに絞る」（`_feeLedgerFeeOnly`）
- 請求月の絞り込みは `_feeLedgerMonth`。選択肢は取得したデータから作る（`_feeLedgerMonths()`）
- ⑤ 算出根拠は上から **「賃金総額 × 料率 ＝ 金額」**（`sales_forecast.wage_total` / `fee_rate`）→
  「単価 × 数量」→ 対象者・登録支援機関。さらに同じ企業の求人案件の
  手数料管理（単価・見込人数・金額）を `_feeLedgerJobBasis()` が1行足す
- **賃金総額・料率を入れる欄は売上の編集画面**（`saf_wage_total` / `saf_fee_rate`）。
  賃金総額を入れると `calcSaleFeeRate()` が「金額（税抜）÷ 賃金総額」のめやすを下に出し、
  **料率が空のときだけ**自動で入れる（手で直した値は上書きしない）
- 「第二種特別加入保険料」と「備考」の空欄は**手書き用**（DBに列を持たせていない）
- 列順は様式なので**勝手に変えない**。印刷（A4横）と Excel出力あり

### 📅 タスクカード → Googleカレンダー

マイページ【✅ タスク・担当リスト】のカードを開いたモーダルの
［📅 Googleカレンダーに追加］。**Googleの「予定を作る画面」を、値を入れた状態で開くだけ**。

- **URLを作るのは `_gcalTaskUrl()` の1箇所**（`GCAL_RENDER_URL` ＋ `action=TEMPLATE`）。
  **認証もAPIも使わない**（外に出るのはこのURLだけ。トークンを持たなくて済む）
- 期限の日の**終日予定**にする。Googleは終了日を「翌日」で渡す決まりなので `_gcalYmd(ymd, 1)`
  （月またぎ・うるう年も `Date` に任せる）
- **期限が空のときは開かない**（終日予定の日付が決められないため。トーストで知らせる）
- 値は**開いているフォーム（`mtf_*`）を優先**する＝保存前に押してもそのまま出る
- **1件ずつ入れる方式なので、あとから期限を直してもカレンダー側は変わらない**。
  自動で追従させるなら別途 ICS の購読リンク（GAS で配信）が要る

### ✅ 権限のお知らせ（その人だけに1回出る）

- **中身は `ROLE_GRANT_NOTICES` の1箇所だけ**（`key` / `users` / `title` / `lead` / `items` / `note`）。
  ログインの少しあとに `showRoleGrantNotice()` がモーダルで出す
- **全員が見るアップデート履歴（`WHATS_NEW`）には、だれに何を許可したかを書かない。**
  「⚙️ 承認まわりの権限を更新しました」程度にとどめ、中身は本人だけに出す
- **できないことは書かない**（できるようになったことだけを並べる）
- あて先は `_renNameIn()` で判定するので、名字だけ／フルネームのどちらでも合う
- 「見た」は `localStorage`（`kmt_role_notice_seen_<user_id>`）に `key` ごとに覚える。
  **新しいお知らせを出したいときは `key` を変えて足すだけ**

### 🗒️ MTG議事録（社内共有事項の【🗒️ MTG議事録】タブ）

- 画面は **📋 共有事項 → 🗒️ MTG議事録 → 📢 アナウンス** の3タブ（`switchNoticeTab`）。
  実体は **`meeting_minutes`** テーブル（1件＝1回のMTG）
- **選択肢は2つの1箇所だけ**
  - `MTG_KINDS` … 週次MTG／随時MTG／緊急MTG／全体MTG／チームMTG／その他（アイコンと色もここ）
  - `MTG_TARGETS` … KMT全体／GLT／総務／各チーム／グループ全体。
    **`team:true` を付けたものを選んだときだけ**下に `MTG_TEAMS`（国名）の欄が出る
    （出し入れは `_mtgNeedsTeam()` の1箇所。共有事項の「グループ → カテゴリー」と同じ考え方）
- **メモと添付は共有事項と同じ**（textarea ＋ ドラッグ＆ドロップ／貼り付け／クリックの入れ物）。
  実ファイルはチャット・ノートと同じ Drive フォルダーへ GAS 経由で入れる
- **ToDo は `todos`（jsonb）**＝ `[{text, who, due, done}]`。
  残っている件数を数えるのは **`_mtgTodoLeft()` の1箇所**で、一覧のバッジと
  「☑️ ToDoが残っているものだけ」の絞り込みが同じ数を見る
- 絞り込みは5つ＝ 検索／MTG種類／対象（＋チーム）／開催日の期間／ToDoが残っているものだけ。
  **ふるいは `renderMinutes()` の1箇所**
- 「各チーム」以外を選んだときは **保存で `target_team` を null にする**（前の値が残らないように）
- 削除は**論理削除**（`is_deleted`）で、消す前に `_confirmDeleteTwice()` で2回たずねる

### 📢 アナウンスの出る場所

- **ログイン時のお知らせ**（`showAnnouncementLoginNotice`）は、［✅ 確認しました］を押した
  アナウンスを `localStorage`（`kmt_ann_notice_seen_<user_id>`）に覚えて、**次からは出さない**。
  新しいアナウンスが来たときだけまた出る（覚えるのは `_annNoticeMarkSeen()` の1箇所）
- **バッジは完了にするまで出続ける**（`renderAnnouncementAlerts` の4か所）＝
  サイドバー `announcementAlerts` ／ 💬 チャットの上 `announcementBanner` ／
  💬 求人チャットの上 `announcementBannerJob` ／ マイページ `myAnnouncementAlerts`

### 📢 アナウンスの「自分の確認状況」

- 確認 → 対応中 → 完了 は **押した時点では画面だけ**変わり、［💾 保存］で書き込む
  （未保存ぶんは `_annPending`。表示は保存済みと重ねた `_annMyConfirm()` を見る）
- 保存先は `announcement_confirmations`（`announcement_id` ＋ `user_id` で1行）。
  **必ず upsert（`resolution=merge-duplicates`）で書く。**
  以前は PATCH だけだったので、行が無い人（あとから対象に足した人・
  SQLで作ったアナウンス）は押しても保存されず、開き直すと未確認に戻っていた
- 未保存のまま**画面を移る／詳細を閉じる／リロードする**とリマインドが出る
  （`annUnsavedPrompt()` ／ `showScreen` の中の `annHasUnsaved()` ／ `beforeunload`）
- 詳細を開き直す `_annRefresh()` は **await して返す**（閉じる処理と重なると、
  閉じたあとに開き直しが走ってモーダルが残るため）

### 📢 アナウンスから自分のテスト候補者へ飛ぶ

- アナウンス本文（`announcements.content_html`）は `innerHTML` で挿し込むので、**中に `onclick` を書ける**
- テスト用のページは2つ。**探し方は同じ**（`【テスト】<ログイン中の名前>用`）
  | ボタン | 関数 | 実体 | 担当の列 |
  |---|---|---|---|
  | 👉 自分のテスト候補者を開く | `openMyTestCandidate()` | `candidates.name` | `staff`（人材担当） |
  | 👉 自分のテスト求人案件を開く | `openMyTestJob()` | `job_progress.company_name` | `sales_staff` / `glt_staff` |
  見つからないときは、その一覧を開いて「【テスト】」で検索した状態にする
- 「👉 自分のテスト候補者を開く」は `openMyTestCandidate()` の1箇所。
  ログイン中のユーザー名から「`【テスト】<名前>用`」（接頭辞は `TEST_CAND_PREFIX`）を探して
  `openCandidateModal()` を開く。見つからなければ候補者一覧を開いて「【テスト】」で検索した状態にする
- アナウンス詳細（`#annDetailModal`）は生DOMなので、閉じるのは `remove()`（`closeGenModal` ではない）

### 📛 ユーザー名とフルネーム

- `users.name` ＝ **ユーザー名**（チャット・各一覧・入力欄はすべてこれ。今までどおり）
- `users.full_name` ＝ **フルネーム**（**書類を作るときだけ**使う。空ならユーザー名のまま出る）
- **引くのは `fullNameOf(name)` の1箇所だけ**。`allUsers` に `full_name` が無い画面でも引けるよう、
  `loadFullNames()` が `name → full_name` の対応表を別に持つ（1回だけ読む）
- ⚠️ **所属機関の担当欄には「蛭田」「白井」のように名字だけが入っている**。
  名前からユーザーを引くのは **`_userByName()` の1箇所だけ**で、ぴったり合わなければ
  **1人だけに絞れるときに限って**前方一致・部分一致でも拾う（2人以上に当たるときは拾わない）。
  `fullNameOf()` / `_meetTeamOf()`（GLT・営業の判定）/ `_meetStaffRole()` は全部これを通す
- 書類でフルネームにする欄は **`REG_DOC_FULLNAME_KEYS` の1箇所**（担当者名が入る欄）。
  差し替えるのは `regRunDocExport` の中の `_regApplyFullNames()` の1回だけ
- ⚠️ **`openRegDocExport` は記録の `data` をコピーしてから使う**（そのまま使うと、
  書類用の書き替え（フルネーム・署名画像）が記録に残ってしまう）
- ⚠️ **フルネームはリポジトリに書かない**（個人情報）。DBの `users.full_name` にだけ持つ

#### 🟨 書類の未記入は黄色で出す

- **どの様式でも同じ**（`buildRegTaskDocHtml` の中の `blank()` / `val()` の1箇所で塗る）。
  色は **`REG_DOC_BLANK_BG` の1箇所**
- **class ではなく style を直に書く**。Word は外部CSSを落とすことがあるので、
  こうしないと Word だけ色が出ない（Word・Excel・PDF・印刷で同じ見た目にするため）
- 値のある欄はそのまま。**「あれば書く」欄は黄色くしない**
  （5-9号の実施日時は1行目だけ／別紙や記録簿の空行はそのまま）

### 📘 登録支援機関業務の記録簿 ← 人材情報の反映

- 様式の定義は **`REG_FORM_DEFS` の1箇所**（項目の一覧は `REG_SUPPORT_TASKS`）
- ⚠️ **`no` は記録（`reg_support_records.task_no`）と結びついた内部の番号。振り直さないこと**
  （過去の記録が迷子になる）。項目を足すときは**使っていない番号を足して、並びだけ変える**
- **参考様式（`form`）も記録簿と同じ「1つの項目」として並べる**（入れ子にしない）。
  並びは**入管庁の様式一覧と同じ 1〜14**＋KMTで足したぶん（15＝支援経費収支記録簿）。
  `kind` はアイコン（📄 / 📘）とサイドバーの色分けにだけ使う
- **画面に出す通し番号は `regTaskDispNo()` の1箇所**＝ `REG_SUPPORT_TASKS` の並び順そのもの
  （参考様式も数える）。見出しは `regTaskLabel()`（サイドバー・画面の見出し・
  `SCREENS` のタイトルが同時に追従する）
- `regSupportFormsOf()` は**常に空を返す**（以前は直前の記録簿にぶら下げていた名残。
  ページ下の「この帳票で使う参考様式」の欄はこれで出ない）
- **自動で入る欄は、フィールド定義の3つのキーの1箇所だけ**で決める

  | キー | 入る値 | 例 |
  |---|---|---|
  | `from` | `workers` の列 | 出入国送迎記録簿（no.5）の `entry_date` など |
  | `fromCo` | `companies` の列（`from` と両方なら人材が空のときの控え） | 5-10号の委託料＝`fee_per_person` |
  | `fromFn(w, co, d)` | 計算して入れる | 5-10号の契約期間（終了）＝就労開始日の1年後 |

  `fromL` / `fromCoL` / `fromNote` は欄の下の注記に出す文字。
  `syncTo` を書くと、その欄を直したとき相手の欄（1年後）も付いてくる（`onRegPeriodFromChange`）
- **1年後の計算と裁判所の既定値は `_regPlusOneYear()` / `REG_DEFAULT_COURT` の1箇所**
- 読む列は `REG_WORKER_SELECT_COLS`（`from`）と `REG_COMPANY_SELECT_COLS`（`fromCo`）が
  定義から自動で作る。**キーを足すだけで取得も追従する**
- `from` を書くと3つが同時に付いてくる
  1. 記入フォームで人材を選んだとき自動で入る（`_regApplyWorkerFill()`。
     **すでに入っている欄は上書きしない**。［📥 人材情報から取り込む］＝`regFillFromWorker()` のときだけ入れ直す）
  2. 一覧の**未記入の行にもその値が出る**（緑文字＋「人材情報あり」の印）
  3. 画面上の［📥 人材情報から一括反映］（`regBulkFillFromWorkers()`）でまとめて記録を作れる。
     **すでにある記録には触らない**／いま絞り込んでいる範囲だけが対象
- **「未記入」かどうかは在職ステータスで変わる。** 判定は様式定義の `needFields(人材)` の1箇所で、
  中身は人材情報の必須判定（`_reqEntryInfo` / `_reqDepartureInfo`）をそのまま使う
  - 在職中・在職中(一時帰国中) … 出迎え日＋入国の担当者（**見送り日は空でも未記入にしない**）
  - 退職済(帰国) … 見送り日＋出国の担当者
  - どちらでもないステータス（配属前・退職済(転職)など）＝「対象外」（数に入れない）
- **一覧の「所属機関」の列には企業番号を頭に付ける**（`_regCoNoHtml()` の1箇所。
  企業ごとの様式は `x.ent` が企業／人材ごとの様式はその人材の所属機関を引く）
- **一覧の名前を押すとその画面が開く**（`entLink(x, txt, 列のキー)` の1箇所）。
  **どちらが開くかは「行の単位」ではなく「列」で決める**＝ `_worker` の列は人材の【基本情報】、
  `_company` の列は所属機関。人材ごとの様式でも `_company` の列があるので
  （5-10号の `listCols` は 所属機関／外国人／月額）、行の単位で決めると両方とも人材が開いてしまう
- **一覧の検索でひっかける文字は `_regSearchText(ent, rec)` の1箇所だけ**
  （人材ごと＝氏名・国籍・管理番号・所属機関名／企業ごと＝企業名）。
  一覧の絞り込みも［📥 人材情報から一括反映］の対象も同じものを見る
- **人材ごとの様式で「所属機関」の列に出すのは、その人材の所属機関名**（`_regCoName(w)` の1箇所。
  記入フォームの `company_name` も同じものを通す）。
  記録がまだ無い行の `x.ent` は**人材**なので、`x.ent.name` をそのまま出すと人材名が2列に並ぶ（実際に並んだ）
- 一覧に出す値は
  **記入フォームの欄に入るのと同じ値**（`_regFillValue`。担当者は人材情報が空なら
  所属機関のメイン担当）＝一覧と［＋ 記入］で違う名前が出ないようにする
- 行の状態は `stateOf()` の1箇所（`done` 記録あり／`ok` 人材情報でそろい／`none` 対象外／`blank` 未記入）。
  記録があっても必要な欄が抜けていれば「記入もれ」を出す（`missOf()`）
- 一覧・フォームで読む人材の列は **`REG_WORKER_SELECT_COLS`**（`from` の列を自動で含める）。
  1つでも欠けていたら `loadRegTaskRecords()` が取り直すので、`from` を足すだけでよい

#### 📄 参考様式第５－１０号（支援委託契約書）の書類

- **条文は `REG_5_10_TASKS`（第１条の１〜11号）＋ `REG_5_10_SUB4`（４の（１）〜（６））の1箇所だけ**。
  入管庁の様式そのままの文言なので、勝手に要約しない
- 第１条11号の「●●分野」は入力欄 `field_name`（人材の `applicant_field` から自動反映）。
  **未記入のときは様式どおり `●●` のまま出す**
- 別紙の費用内訳は **`REG_5_10_ROWS`(10) 項＋合計**（様式どおり。1項につき「金額」「徴収時期」の2行）。
  徴収時期は `REG_FEE_TIMINGS`（随時／定期）で ☑ を出す
- **別紙に最初から入れる内容は `REG_5_10_BREAKDOWN_SEED` ＋ `REG_5_10_INIT_FEE` の1箇所だけ**
  - 項1（随時）＝全企業とも同じ金額（`REG_5_10_INIT_FEE`）／項2（定期）＝**その企業の委託料**（`fee_month`）
  - 入れるのは **空の欄だけ**（`reg510FillBreakdown(false)`）。手で直したものは開き直しても上書きしない。
    元に戻すのは［📥 既定の内容を入れ直す］＝ `reg510FillBreakdown(true)`
  - 呼ぶのは2か所（フォームを開いた最後／`onRegWorkerPick` で委託料が入ったあと）
- `rows` 型の列に **`multi:true`** を書くと textarea になり、改行がそのまま書類に出る（別紙の「名目」）
- 書類の合計欄は **定期／随時／合計（税別）の3行**（様式の記入例と同じ）
- **署名欄は「住所 → 会社名 → （甲/乙）代表者　㊞」の3段**を左右に並べる
  - 甲＝所属機関の `pref+city+address` と `rep_title+rep_name`（**引くのは `_regCoAddr()` / `_regCoRep()` の1箇所**）。
    この列は **`REG_COMPANY_SELECT_COLS` に明示で入れてある**（`fromFn` で組み立てるので自動では入らない）
  - 乙＝**`REG_ORG_INFO` の1箇所**（機関名 → 住所・代表者）。`reg510FillOrg()` が入れる。
    **手で書いたものは残す**（空のとき／ほかの機関の既定値のままのときだけ入れ替える）
- 🖋️ **乙の電子印は管理者だけ**。保存先は `app_settings.reg_org_seals`（機関名 → `{img, at, by}`、img は data URI）
  - `loadRegOrgSeals()` は**管理者以外では読み込まない**ので、画像が手元に届かない＝画面にも書類にも出ない
  - 出すのは `_regSealImgHtml()` の1箇所（管理者以外には空を返す）。設定は `openRegSealModal()`（`requireAdmin`）
  - ⚠️ これはアプリの中での制限。DBは公開キーで読めるつくりなので、
    **絶対に他の人の手に渡ってはいけない画像はここに置かない**

#### 📄 事前ガイダンスの確認書（5-9号 ／ no.4 ／ 表示も「4.」）

- 参考様式だが、**企業と人材の一覧＋書類生成を持つ独立した項目**（ほかの様式と同じ扱い）
- 本文は **`REG_5_9_ITEMS` の1箇所**（入管庁の様式そのまま。要約しない）
- 書類の見た目も様式どおり。部品は **`_reg59Cell` / `_reg59SessionsHtml` / `_reg59Label` の3つだけ**
  - 日付・時刻は「　年　月　日　時　分から　時　分まで」の**マス目**（空欄はそのまま空ける）
  - 実施日時は様式どおり **`REG_59_SESSION_ROWS`(3) 行**出す（入力が1件でも空行を2行足す）
  - 機関名・説明者は**表にしない**（見出し → 下線の上に中央ぞろえ）
- 自動で入るもの
  - 実施日時 … 年月日＝人材の `pre_guidance_date`／時刻＝**全員一律**（`REG_59_TIME_FROM`〜`REG_59_TIME_TO`）。
    入れるのは `reg59FillSession()` で、**空の欄だけ**（［📥 既定の内容を入れ直す］で入れ直せる）
  - 説明者 … その企業のメイン→サブ担当のうち **GLTのユーザー**（**`_regGltStaffOf()` の1箇所**。営業は入れない）
  - 署名日 … `from:'pre_guidance_date'`（事前ガイダンス実施日と同じ）

#### 📄 生活オリエンテーションの確認書（5-8号 ／ no.7 ／ 表示も「7.」）

- **5-9号とまったく同じつくり**（企業と人材の一覧＋書類生成＋本人の電子署名）。
  書類を組み立てるのも `buildRegTaskDocHtml` の **`no === 4 || no === 7` の1箇所**で、
  部品（`_reg59Cell` / `_reg59SessionsHtml` / `_reg59Label`）も共用する。
  **見た目を直すときは両方に効くので、片方だけ直したいときは分岐を足すこと**
- 本文は **`REG_5_8_ITEMS` の1箇所**（入管庁の様式そのまま。要約しない）。
  タイトルは「生 活 オ リ エ ン テ ー シ ョ ン の 確 認 書」。
  **「また、４について〜」の一文は 5-9号だけ**（5-8号には出さない）
- 自動で入るもの
  - 実施日時 … 年月日＝人材の `orientation_date`／時刻＝**全員一律**
    （`REG_58_TIME_FROM`〜`REG_58_TIME_TO` ＝ 09:00〜18:00）。入れるのは `reg58FillSession()` で、
    **空の欄だけ**（［📥 既定の内容を入れ直す］で入れ直せる）。呼ぶのは
    `openRegTaskForm`（開いた最後）と `onRegWorkerPick`（人材を選んだあと）の2か所
  - 機関名・説明者・署名日は 5-9号と同じ（説明者＝`_regGltStaffOf()`、署名日＝`orientation_date`）

#### 📄 相談記録書（5-4号 ／ no.10 ／ 表示も「10.」）

- **企業と人材の一覧＋書類生成**（`type:'worker'` の欄があるので一覧は人材ごと）
- **相談記録の行は面談記録（`company_meetings`）から作る**＝ `reg54FillRows(force)` の1箇所
  （相談受理日＝面談日／相談内容＝`memo`／対応結果＝`result_memo`／対応者＝`staff_name`）。
  氏名・性別・国籍・地域・生年月日・在留カード番号は**人材の基本情報**から入れる
  - 呼ぶのは2か所（`onRegWorkerPick` で人材を選んだとき／［📥 面談記録から入れ直す］）。
    **人材を選んだときは空のときだけ**入れる（手で直したものを消さない）
- **見出しの期間は `_reg54Period()` の1箇所**＝ 相談受理日の**いちばん早い月の1日 〜
  いちばん遅い月の末日**（様式の記入例が「1/30・2/1・3/5 → 1月1日から3月31日まで」なので月まるごと）。
  入れるのは `reg54FillPeriod()` で、あとから手で直せる
- 書類は様式どおり。**1ページ目が `REG_5_4_ROWS_P1`(4) 行、2ページ目が5〜`REG_5_4_ROWS_P2`(9) 行**
  （足りない行は空の枠のまま出す）。性別は「男・女」を出して入っているほうに○を付ける
- **「記載上の留意点」のページは `REG_5_4_NOTES` の1箇所**（★1＝相談受理日／★2＝対応結果）。
  ★は書類の見出しにも黄色で出す
- **用紙の向きは「横」が既定**（表が横に長いため）。決めるのは **`REG_DOC_ORIENT` の1箇所**
  （様式番号 → `landscape`）。Word・Excel・PDF・印刷のどれも同じ向きで出る
  - 出力設定の画面でその回だけ縦に変えられる。
    **向きが決まっている様式は、選んだ向きを `localStorage` に覚えない**
    （覚えると、ほかの様式まで横向きで開いてしまう）

#### 📄 定期面談報告書（5-5号・5-6号 ／ no.13）

- ⚠️ **記録は「所属機関 × 四半期」で1件**（人材ごとではない）。届出が四半期ごとで、
  1社ぶんをまとめて出す運用のため。`def.period:true` の様式はこれだけ
- **届出の四半期は `MEET_REPORT_QUARTERS` の1箇所**（暦の3か月区切り。
  面談記録の「年度（4月はじまり）」とは呼び方が違うだけで、区切る3か月は同じ）。
  保存してある文字（「第3四半期（7〜9月）」）→番号は **`_meetQOf()` の1箇所**
- **その四半期に在籍していた人材と区分（在職中／新規配属／退職）は
  `_meetReportWorkers()` の1箇所**。はじまり＝`joined_date`（無ければ `work_start_date`）、
  おわり＝`left_date`（無ければ `expected_leave_date` / `departure_date`）
  - ⚠️ **退職済みなのに退職日が空の人材は、対象に入れない**（いつ辞めたか分からないため。
    入れると過去の四半期すべてに出てしまう）。**足りない人材は `_meetReportNeedFix()` の1箇所**が出し、
    一覧の「対象人材」に ⚠️日付なし◯名 と付く。人材情報に退職日（または配属日）を入れれば出るようになる
- **面談事項（18項目）は `REG_MEET_GROUPS` の1箇所だけ**（入管庁の様式そのまま）。
  `w`＝5-5号の言い方（本人に聞く）／`s`＝5-6号の言い方（監督者に聞く）。
  **並びと数は同じ**なので、書類を作るときに `_regMeetItemText()` の1箇所で入れ替える
- **原則はすべて「問題（無）」＝ `MEET_NO_PROBLEM` の1箇所**。
  「有」にするのはその場で選び直したときだけ（有りのときは個別に手で対応する運用）
- 見出し・面談対象者の欄・記載要領のリンクの違いは **`REG_MEET_FORMS` の1箇所**。
  書類の下に出すリンクは `_regMeetLinksHtml()`（両方の様式ぶんを出す）
- **書類は面談日ごとに分けて出す**（`buildRegTaskDocHtml` の no===13 の1箇所）。
  その日の対象者が **1名なら氏名をそのまま／2名以上なら「別紙参照」＋別紙**（様式に1つしか
  氏名欄が無いため）。第５－６号（監督者用）も同じ日付でセットにする
- **別紙は「１号特定技能外国人支援対象者名簿」**（参考様式第４－３号 別紙と同じ形）＝
  氏名（ローマ字）・性別・生年月日・国籍地域・在留カード番号・住居地・支援実施状況
  - **記録に入っていない項目は人材情報から引く**（**`_meetMemberInfo(m)` の1箇所だけ**）。
    氏名と在留カード番号しか入っていない古い記録でも、開き直さずに正しく出る
  - 住居地は `workers.jp_address` ＋ 電話（`phone` → `cel_phone`）。
    読む列は `REG_WORKER_SELECT_COLS` に入れてある
  - **支援実施状況の選択肢は `MEET_SUPPORT_STATUSES` の1箇所だけ**で、
    **既定は `MEET_SUPPORT_DEFAULT`（問題なし）**。「未実施の支援項目がある」は別のチェック
    （`MEET_UNIMPL_OPTS`）。どちらも記入フォームの【対象の外国人材】の行で変えられる
  - **別紙のページだけ横向きにする**（表が横に長いため）。囲むのは
    **`REG_DOC_LS_START` / `REG_DOC_LS_END` の印の1箇所だけ**で、出力ごとに置き換える
    | 出力 | やり方 |
    |---|---|
    | Word | `buildRegWordHtml()` が `WordSection2`（もう一方の向き）に切り替える |
    | 印刷・PDF | **名前つきページ**＝ `_regDocPrintHtml()` が `.reg-ls` の入れ物にし、`_regLsPrintCss()` が `@page ls{size:…}` ＋ `.reg-ls{page:ls;}` を出す |
    - ⚠️ **Excel だけは、ページごとに向きを変えられない**ので書類全体が
      `REG_DOC_ORIENT` / 出力設定で選んだ向きのまま出る。
      **no.13 の既定は Excel（`REG_DOC_FMT`）なので、別紙を横で出したいときは PDF か Word にする**
    - 印（`<!--REG_LS_...-->`）はコメントなので、置き換えない出力では**そのまま消える**
      （画面・Excel に文字として出ない）
- 自動で入るもの（**きまりは `REG_MEET_ROLES` の下の1かたまり**にまとめてある）
  - 対象の人材 … `reg13FillMembers()`（`_meetReportWorkers` の結果）
  - **面談日** … 面談記録があればその日。無ければ **`_meetAssignDates()` の1箇所**が
    「**四半期の最後の月の平日**」から割り当てる（足りなければ真ん中の月も使う）。
    **同じ担当者が同じ日に別の都道府県へ行かない**ように選ぶ（同じ都道府県なら同じ日に何社でも可）。
    並べ替えは年・四半期を種にしているので、**何度やっても同じ結果**になる
    - **`MEET_RANDOM_UNTIL`（2026年 第3四半期）までの四半期だけ**。それ以降は実際の記録の日を使う
    - 空いているぶんだけ入れ直すのは `autoMeetDateBulk()`（［🎲 空いている面談日を自動で入れる］）
    - **1社ぶんを引くときは `_meetAutoDateFor()` の1箇所**（その四半期の対象企業ぜんぶで計算してから
      1社を取り出す。1社だけで計算すると、まとめて作成したときと食い違う）。
      記入フォームを開いたときと書類を作るときの両方がこれを見るので、**空のままの記録でも同じ日が出る**
  - **①監督者の氏名及び役職**（5-6号）… **`_meetSvOf()` の1箇所**が氏名と役職を決める。上から順に
    ①`companies.supervisor_name` ／②`rep_name` ／③`contact_name`・`contact_persons[0]`。
    **役職はどれも無ければ `MEET_SV_TITLE_DEF`（代表取締役）**（③は `dept` があればそれ）。
    書類は「氏名　（役職）」に組み立てる（古い記録は氏名の欄に（役職）まで入っているので、そのときは足さない）。
    **代表者名が入っている企業が少ない**ので、担当者まで見る（223社中203社が埋まる）。所属部署は空でよい。
    **記録が空でも書類を作るときに所属機関から引き直す**（きまりを入れる前に作った記録のため）
  - **②① 対応者の氏名は様式ごとに違う。出し分けは `_meetStaff55()` / `_meetStaff56()` の2つだけ**
    - 第５－５号 … サブ担当（GLT）。サブが営業ならメイン（GLT）
    - 第５－６号 … メイン担当（営業）。メインがGLTならサブ（営業）。どちらもGLTならメインのまま
    - **書類にはフルネーム**で出す（`buildRegTaskDocHtml` の中で `fullNameOf()` を通すので、
      `REG_DOC_FULLNAME_KEYS` には入れない＝二重に変換しない）
  - 役職＝`MEET_STAFF_ROLE_DEF`（支援担当者）／役職名＝`MEET_STAFF_TITLE_DEF`（役職なし）／
    方式＝`MEET_METHOD_DEF`（対面。面談記録があればその実施方法）
  - **作成年月日＝面談日の `MEET_MADE_AFTER_DAYS`(14)日後の平日**（`_meetMadeDate()` の1箇所）。
    **面談実施者の氏名は、その様式の対応者と同じ**（欄は持たせていない）
  - 原則の値を入れるのは `reg13FillDefaults()`。**空の欄だけ**に入れる
- **📅 面談日をまとめて入れる画面は `openMeetDateBulk(no)` の1箇所**
  （その四半期の記録を企業一覧の形で並べ、面談日・実施方法・対応者を入れて `saveMeetDateBulk()` が
  **直した企業ぶんだけ**書く）。**面談日を入れると、人ごとの面談日が空の人にも同じ日を入れる**
  （別紙にそのまま出るように）。まだ作られていない企業があるときは件数と［まとめて作成する］を出す
- **⚠️ 日付が足りない人材を直す画面は `openMeetDateFix(companyId)` の1箇所**
  （企業ごとに並べて配属日・退職日を入れ、`saveMeetDateFix()` が**直した欄だけ**を workers に書く）。
  入口は2つ＝画面の［⚠️ 日付が足りない人材］（全体）と、一覧の「⚠️日付なし◯名」（その企業だけ）
- **［📥 この四半期をまとめて作成］は `reg13BulkCreate()` の1箇所**。
  その四半期に在籍していた人材がいる企業ぶんを一度に作る。**すでにある記録には触らない**
  （過去の四半期をそろえるための入口）
- **出力の既定は Excel**（様式が Excel のため）。決めるのは **`REG_DOC_FMT` の1箇所**
  （`REG_DOC_ORIENT` と同じつくり。**選んだ形式は `localStorage` に覚えない**＝ほかの様式に移らない）
- ４ 基準不適合等への対応は「あったときだけ書く」欄なので、
  **⑥が「なし」のときは黄色くしない**
- 選択欄に `on:'関数()'` を書くと、選び直したときにその関数を呼べる

#### 📄 作成済みの書類（生成履歴 ／ `reg_doc_history`）

- 書類を作れる様式（`def.doc` があるもの）の画面は **📝 記録の一覧 ／ 📄 作成済みの書類** の2タブ。
  切り替えは `setRegView(no, 'list'|'docs')`、いま見ているほうは `_regViewOf(no)`
- 出力設定のフッターのボタンは2つ＝ **［📄 書類生成のみ］（履歴に残さない）／
  ［💾 書類生成＋履歴保存］**。どちらも `regRunDocExport(saveHistory)` の1箇所を通り、
  最後に `_regSaveDocHistory()` を呼ぶかどうかだけが違う
- **履歴を残せる様式かどうかは `_regDocHistoryOn(no)` の1箇所**
  （`REG_DOC_HISTORY_TASKS` が `null` のあいだは `def.doc` があるものすべて）
- **「何年何期分か」を作るのは `_regDocPeriodLabel(no, d)` の1箇所**＝
  年＋四半期（no.13）→ 対象年月（no.15）→ 記録の主な日付、の順に見る
- 残すのは `reg_doc_history`（様式番号・記録id・企業／人材・対象期間・形式／用紙／向き・
  ファイル名・**Driveのリンク（`file_url`）**・生成日時・生成した人）
- **実ファイルは Google ドライブに入れる**（チャットの添付・領収書と同じ GAS 経由）。
  **置き場は `REG_DOC_DRIVE_URL` の1箇所だけ**、入れるのは `_regUploadDocToDrive()` の1箇所だけ
  - **どの形で残すかは `REG_DOC_SAVE_KIND` の1箇所**（拡張子・MIME・中身の作り方）＝
    Excel は `.xls`、**それ以外は `.doc`（Word）**
    - ⚠️ PDF・印刷はブラウザの印刷ダイアログが作るので、こちらでファイルを取り出せない。
      **`.html` で入れてはいけない**（ドライブのプレビューがHTMLのソースをそのまま出す。実際に出た）。
      Word なら中身がそのまま見え、開いてPDFにもできる
  - 印刷・PDF用のHTMLは **`buildRegPrintHtml()` の1箇所**（画面の印刷に使う）
  - **Driveに入らなくても履歴は残す**（`file_url` が空になるだけ）。黙って消えないようトーストで知らせる
  - 一覧・記入フォームの**ファイル名を押すと開く**。作るのは `_regDocFileLink(r)` の1箇所
  - ⚠️ **ドライブのURLは `driveFileViewUrl()` を必ず通す（直すのはこの1箇所だけ）。**
    `.doc` / `.xls` を入れると `docs.google.com/document/d/<id>/edit?…rtpof=true` が返ることがあり、
    これは**Googleドキュメントの編集画面**のURLなので、中身が変換されていない生のファイルは
    「現在、ファイルを開くことができません。」になる（実際になった）。
    `drive.google.com/file/d/<id>/view` に直せば開ける。入れたときと出すときの両方で通すので、
    すでに入っている古い記録もそのまま開ける
  - **作り直したほうがよい行かは `_regDocNeedsRemake(r)` の1箇所**
    （ドライブに入っていない／`.html` で入っている古いぶん）。⚠️ を付けて［📄 作り直す］を出す
- 一覧は `renderRegDocHistory(no)`（検索は `regDocQ_<no>`）。行を押すと元の記録が開き
  （`regDocHistoryOpen`）、🗑 は**論理削除**（`is_deleted`）
- **記入フォームの中にも「📄 この記録で作成済みの書類」を出す**（`renderRegDocRecPanel()`。
  いま開いているフォームは `_regDocRecFor` の1箇所に持つ）。**見るのは同じ `_regDocHist`** なので
  一覧と必ず同じ内容になる。**保存前（新規）のときは出さない**（記録に結び付けられないため）
- 読み込みは `loadRegDocHistory(no)`。`loadRegTaskRecords(no)` の先頭から呼ぶので、
  画面を開けばタブの件数までそろう

#### 👁 書類のプレビュー（ダウンロードしない）

- **書類に流し込む中身を作るのは `_regDocData(no, id)` の1箇所だけ**。
  出力（`openRegDocExport` → `regRunDocExport`）もプレビュー（`regDocPreview`）も同じものを見るので、
  **画面で見たものと出したものが必ず同じ**になる
  - ⚠️ 記録の `data` は**必ずコピーしてから使う**（書類用の書き替え＝フルネーム・署名画像が
    記録に残ってしまうため）。`sign:true` の様式は `_regSignOf()` から署名画像を `_sign_img` に載せる
- **見せるのは `regDocPreview(no, id)` の1箇所**。`buildRegTaskDocHtml(no, d)` を
  `#regDocPreviewBody` に描いて、A4風の白い紙面（縦780px／横1000px）に出す。
  **z-index は 10070**（出力設定のモーダルより上）。フッターから［🖨️ 書類を出力］でそのまま出力へ進める
- 入口は3か所＝ **【📝 記録の一覧】の各行**（🖨️書類の左）／
  **【📄 作成済みの書類】の各行**（`r.record_id` があるとき）／
  **記入フォームの中の「この記録で作成済みの書類」**
- ⚠️ **これが要る理由**: ドライブに入れている `.doc` は**中身がHTMLのWordファイル**なので、
  Wordでは開けるが**ドライブのプレビューでは開けない**（「ファイルをプレビューできませんでした」）。
  GASは変えられないので、**同じ中身をアプリの中で組み立てて見せる**ことにした

#### 📜 委任状（オンライン申請の代理 ／ no.16）

- **所属機関ごとに1件**。委任者＝その企業（機関名・所在地・代表者の氏名・連絡先を
  `reg16FillDefaults()` が所属機関から入れる）、受任者＝**`POA_AGENT` の1箇所**
- **本文は `POA_GROUPS` / `POA_LEAD` の1箇所**（これまで Excel で作っていたひな形と同じ文言）
- 一覧は **`def.poaOnly` が付いているので「委任状が必要な企業だけ」に絞れる**（既定はON）。
  必要かどうかは **`companies.needs_poa`**（過去の資料の「委任状がいるところ」から取り込み済み）

#### ✍️ 本人の電子署名（`sign:true` の様式）

- 実体は **`reg_sign_links`**。1記録＝1リンクで、`record_id` で結び付ける
- **`#regsign=<token>` でログインなしに開ける**（`initLogin` の先頭で分岐）
- 署名の画像（data URI）は**リンクの行だけ**に持つ。`reg_support_records` は書き替えない
  （外に出るページが触るのは自分の1行だけになる）
- 書類に載せるのは `openRegDocExport` で `d._sign_img` に入れるところの1箇所。
  一覧の ✍️署名済み／⏳署名待ち は `_regSignOf(record.id)` を見る
- **署名ページに出す確認事項は `REG_SIGN_ITEMS` の1箇所**（`{4:REG_5_9_ITEMS, 7:REG_5_8_ITEMS}`）。
  `link.task_no` で選ぶ。**`sign:true` の様式を足したらここにも足すこと**
  （足さないと、別の様式の確認事項が出たまま署名させてしまう。実際に出た）
- 署名の入力は `regSignInitPad()`（pointer イベントなのでマウスも指も同じ処理）。
  キャンバスは `devicePixelRatio` を掛けてから `scale` する（スマホでぼやけないように）
- **`sign:true` の様式には3つめのタブ【✍️ 署名リンク】が出る**
  （`regPanel_signs_<no>` ／ `renderRegSignList(no)`）。1行で
  **① リンクの発行（いつ・だれが）→ ② ご本人の回答（署名の画像と日時）→ ③ 回答のあとに作った書類**
  まで追える。行を押すとその記録が開く
  - **状態を決めるのは `_regSignState(l)` の1箇所**（`REG_SIGN_STATES` ＝ 署名済み／署名待ち／無効）。
    タブの数字は**署名待ちの件数**
  - **「回答後の書類」は `_regSignDocs(no, l)` の1箇所**＝ その記録の `reg_doc_history` のうち、
    **`signed_at` 以降に作ったものだけ**（署名前に作ったぶんは数えない）。
    ［📄 書類］を押すと出力設定が開くので、署名入りで作り直して履歴に残せる
  - 元データは `reg_sign_links` と `reg_doc_history` の2つだけ。
    リンクを出す・止めるたび（`createRegSignLink` / `closeRegSignLink`）と
    書類の履歴を読み直すたびに描き直す

#### 🧾 支援経費収支記録簿（no.15 ／ 表示も「15.」）

- **所属機関 × 対象年月で1枚**（`type:'company'` の欄があるので一覧は企業ごと）
- **収支区分の選択肢は `REG_IE_KINDS` の1箇所だけ**（1つめが収入・2つめが支出）。
  `rows` 型の列に `opts` を書くと選択式になる（`regRenderRows` の1箇所）
- **合計は入力させない＝行から計算する。数えるのは `_regIeTotals()` の1箇所**
  （収入計・支出計・差引。一覧の「差引（円）」も書類の「合計」も同じものを見る）。
  **合計＝収入−支出**で、内訳（収入計／支出計）は書類の注記に出す
- 対象年月の見た目（`2026-08` →「2026年8月」）は **`_regYmText()` の1箇所**
- 書類は様式そのまま。行が少なくても `REG_IE_MIN_ROWS`(15) 行ぶんの空の枠を出す（手書きで足せるように）

#### 🤝 外国人材の【その他支援】タブ（基本情報のとなり）

- 出すのは **`WORKER_OTHER_TASKS` の1箇所**（住居確保・生活契約支援＝no.6／関係機関同行等支援＝no.8／
  日本語学習支援＝no.9／日本人との交流促進支援＝no.11／転職支援＝no.12。
  `label` が見出し、`no` が記録簿の番号）。**タブの上の案内文もここから作る**（ベタ書きしない）
- **入力する項目は記録簿と同じ**＝ `REG_FORM_DEFS` が唯一の元。項目を足すときは記録簿の定義だけを直す
- **記録の実体も同じ `reg_support_records`**（別テーブルを作らない）。
  人材から入れても記録簿から入れても同じ行なので、どちらで直しても両方に出る
- 記入・編集は記録簿と同じ `openRegTaskForm()` を開く（入力欄を二重に持たない）。
  保存・削除のあとの読み直しは `_regAfterChange()` の1箇所（開いているほうを取り直す）
- 記録を2か所から開くので、`_regFindRec()` が記録簿の読み込み分と人材タブの読み込み分の両方から引く

#### 担当者名の既定値（所属機関のメイン担当）

- フィールド定義に `fromCo` を書くと、**人材情報が空のときだけ**その人材の所属機関の欄を使う
  （出入国送迎記録簿の「実施担当者の氏名・役職（入国）」＝ `companies.main_staff`）。
  値を決めるのは `_regFillValue()` の1箇所、注記は `_regFillNote()` の1箇所
- **一覧の「人材情報あり」の印と［📥 一括反映］の対象は、人材自身に値があるときだけ**
  （メイン担当まで数えると、担当者がいる企業の人材が全員そうなってしまう）

### 📍 住所（所属機関の本社所在地・就業場所）

- **入力欄は「郵便番号」と「住所」の2つだけ**（都道府県・市区町村・番地には分けない）。
  分けても正しく入っていることがほとんど無く、出張の行き先を調べるときに何度もコピーが要ったため
- **DBは今までどおり `pref` / `city` / `address` の3列**。ただし `pref` と `city` は
  **住所の文字列から自動で切り出した「おまけ」**で、手では入れない
  （一覧の都道府県の絞り込み・並べ替え・面談日の都道府県判定が今までどおり効くようにするため）
  - 切り出すのは **`splitAddress(addr)` の1箇所だけ**（`_addrPref()` と既存の `_prefCity()` を使う）。
    `address` には**都道府県から建物名まで全文**を入れる
  - 読むのは **`joinAddress(o)` の1箇所だけ**。`address` が都道府県で始まっていればそれが全文、
    始まっていなければ旧データなので `pref + city + address` をつなぐ。
    **`[co.pref, co.city, co.address].join('')` と書かない**（新形式で都道府県が二重になる）
  - 移行は不要。開いて保存すればその企業だけ1本にそろう
- **Googleマップのリンクは `_addrMapHtml(v)` の1箇所**。出すのは
  `isMappableAddress()` が true のとき（都道府県＋市区町村があり、そのあとに番地が続く）だけで、
  足りないときは「都道府県から市区町村・番地まで入れると地図のリンクが出ます」と出す
  - リンクは2つ＝ 🗺️ Googleマップで開く（`mapsUrl`）／🚗 KMTからの経路（`mapsRouteUrl`）。
    **経路の出発地は `KMT_OFFICE_ADDRESS` の1箇所**
  - 描き直すのは `updateAddrMap(id)`（本社・就業場所）と `updateAddrMapEl(el)`（追加の就業場所。
    id を振れないので入力欄の隣の `.addr-map` を見る）

### 🏢 提携先登録支援機関の詳細（2タブ）

| タブ | 中身 | 保存 |
|---|---|---|
| 🏢 提携先情報 | 登録した項目をその場で直せる入力欄 | `saveIntroRegOrg`（フッターの💾保存） |
| 🔖 求人・コード管理 | コード管理（最後に使ったコード・次の候補・重複）＋そのコードの求人一覧 | 読むだけ |

- **項目の一覧は `IRO_FIELD_SEED` の1箇所**（ラベル・並び・入力の種類 `ftype`）。
  ラベルと並びは `intro_reg_org_fields` テーブルにも入れて、管理者が変えられるようにしてある
  （初回アクセス時に自動投入）。`col` が保存先
- `ftype` は text / textarea / url / select / multi / contacts / fstaff / request
  - `multi`（区分・主な職種）… 複数選択。値は「、」でつないで既存のテキスト列に入れる。
    もともと入っている値で選択肢に無いものは「その他（記入）」の欄に残す（`_iroSplit` / `_iroMultiValue`）
  - `contacts`（担当者）… 1名ずつ（担当者名・メール・電話・その他連絡ツール）を
    `intro_reg_orgs.contacts`（jsonb）へ。**contacts が空のときは `contact_name` を1名ずつに分けて出す**。
    保存時は一覧・検索用に `contact_name` にも名前を並べて書く
  - `fstaff`（外国籍スタッフ情報）… `foreign_staff`（jsonb `{count, memo, list:[{name,nationality,contact}]}`）。
    **「外国籍スタッフ在籍＝あり」のときだけ出す**（`onIroForeignChange`）。
    前のデータの「有／無／確認中」は `_iroForeignNorm()` が あり／なし／要確認 に読み替える
  - `request`（依頼内容）… 選択（`request_type`）＋費用の記入欄（`request_fee`）
- **担当者・外国籍スタッフの行を足すときは、その欄だけ描き直す**
  （`_iroRenderContacts` / `_iroRenderFstaff`）。タブ全体を描き直すと、
  ほかの欄の入力中の値が消える（実際に消えた）
- **既存の列（`IRO_BASE_COLS`）はそのまま、あとから足した項目は `intro_reg_orgs.extra`（jsonb）** に入る。
  値を読むのは `_iroValue()` の1箇所
- **項目名の変更・追加・削除は管理者だけ**（`requireAdmin()`）。「⚙️ 項目を編集」を押すと
  各項目に［✏️ 名前］［🗑］が出る。**削除は行を消さず `is_active=false`**（入力済みの内容を残すため）
- 保存すると `updated_at` / `updated_by` を書き、`renderSaveMeta('sm_iro', …)` で
  「✓ 最終更新 日時（ユーザー名）」を出す

### 🗓️ 社内の年間カレンダー（社内の一番下）

- 実体は **`company_calendars`**（**1年度＝1行**。年度は4月はじまり）。
  `fy` / `target_hours`（目標労働時間/日）/ `std_hours`（所定労働時間/日）/
  `days`（日付ごとの区分）/ `holidays`（◇祝日）/ `birthdays`（🎂 誕生月）
- **日の区分は `CAL_DAY_KINDS` の1箇所だけ**（勤務日／休日／祝日／特別休暇）。
  **休みとして数えるのは `CAL_OFF_KINDS` の1箇所**（所定労働日数から引くもの）
- **集計は `_calMonthStats()` の1箇所だけ**＝
  所定労働日数＝その月の日数 − 休みの日数／月間最低労働時間＝×目標時間／月間総労働時間＝×所定時間。
  年間休日（上期4〜9月／下期10〜3月／合計）は `_calHalfStats()`
- **直せるのは管理者だけ**（`_calCanEdit()` ＝ `isAdminUser()`／保存は `requireAdmin()`）。
  管理者以外は見るだけ（日付を押しても変わらない・保存ボタンも出ない）
  - 日付を押すと **勤務日 → 休日 → 祝日 → 特別休暇** の順に変わる（`calCycleDay`）
  - ［📅 土日を休日にする］は**まだ何も入っていない土日だけ**を休日にする（祝日・特別休暇は触らない）
  - ◇祝日を足すと**その日は自動で「祝日」の色になる**（2か所で直さなくていいように）
- **保存は upsert（`on_conflict=fy`）**。画面の操作はその場では書かず、［💾 保存］でまとめて入れる
- 印刷は A4縦（`printCompanyCalendar`）。ボタン・入力欄は落としてから出す

### 💸 立替管理（立替登録）

- **選択肢は5つの const の1箇所だけ**。増やす・並べ替えるときはここを直せば
  登録画面・一覧・絞り込みが同時に追従する

  | const | 中身 |
  |---|---|
  | `EXPENSE_STATUSES` | 未請求／請求済／入金済 |
  | `EXPENSE_BILL_TO` | 人材／企業／登録支援機関 |
  | `EXPENSE_ITEMS` | 名目（航空券代・宿泊代・試験代（2号試験）・試験代（ビジネスキャリア検定）・JITCO保険・セントラルインシュランス保険） |
  | `EXPENSE_PAY_METHODS` | 支払（決済）方法＝AMEX／ANA／振込／現金 |
  | `EXPENSE_PAID_METHODS` | 入金方法＝振込／現金 |

- 名目は選択肢に無いものを「その他（記入）」の欄に書く。**保存先は今までどおり `item_name` の1列**
  （古いデータも `EXPENSE_ITEMS` に無ければ自動で「その他」の欄に出る）
- **ステータスで出る欄が変わる**（`renderExpenseStatusFields()` の1箇所）。
  請求済＝請求者・請求日／入金済＝＋入金確認者・入金日・入金方法。
  **切り替えたときだけ**空欄に「自分＋今日」を入れる（`onExpenseStatusChange()`。開いただけでは入れない）
- **請求先で出る欄が変わる**（`renderExpenseBillToFields()` の1箇所）
  - 人材 … 人材を選ぶ → **企業はその人材の所属で決まる**（選ばせない・読み取り専用で出す）
  - 企業 … 企業を選ぶ → 人材は**その企業の人材から選ぶ**か手入力（`worker_name_custom`）
  - 登録支援機関 … `org_name` / `company_name_custom` / `worker_name_custom` の3つとも手入力
- **人材名・企業名は打つと候補が出る欄**（`_expPickHtml` / `expPickOpen` / `expPickChoose`）。
  選んだ結果は隠しの `ef_worker_id` / `ef_company_id` に入り、**この2つが唯一の正**
  （表示の文字は毎回 id から引き直すので、名前だけ書き換わることがない）
- **使わない列は保存時に必ず `null` で書く**（`saveExpense`）。
  請求先を切り替えたときに前の内容が残らないようにするため
- 候補の人材・企業は `_expEnsureMasters()` が専用に取っておく。
  **`allWorkers` は画面によって軽いSELECTで上書きされるので当てにしない**
- 一覧に出す名前は **`expWorkerNameOf()` / `expCompanyNameOf()` の2つだけ**
  （選んだ人材・企業か、手入力ぶんか）
- モーダルは**手動保存**（`_manualSave.expenseModal` ／ 接頭辞 `ef_` ／ `sm_expense`）。
  ［💾 保存］を押すまで書かない。保存しても**モーダルは閉じない**（最終更新をその場で見せる）。
  ピッカーや領収書の追加・削除は input イベントが出ないので `_expTouch()` で未保存の印を付ける
- 🗑 削除は**論理削除**（`is_deleted` / `deleted_at` / `deleted_by`）。
  入口は一覧の右端（`deleteExpenseRow`）とフッター右端（`deleteExpense`）の2つで、
  中身は **`_expDeleteRow()` の1箇所**。消す前に `_confirmDeleteTwice()` で2回たずねる。
  取得には `is_deleted=not.eq.true` を付ける

### 💳 カード決済報告

- 立替管理と同じ作り（一覧＋手動保存のモーダル＋論理削除）。実体は **`card_payments`** テーブル
- **選択肢は3つの1箇所だけ**

  | const | 中身 |
  |---|---|
  | `CARD_ITEMS` | 名目＝航空券代・宿泊代・レンタカー代・印鑑代・広告料（＋その他は記入） |
  | `CARD_LIST` | 利用カード＝AMEX（KOTA TAKAMURA／KEN KOMORI／TUNNISA FAUZIA／AKIRA OFUSA）・ANA（KEN KOMORI） |
  | `CARD_PURPOSES` | 目的＝配属・定期面談・本帰国（＋その他は記入） |

- 名目・目的の「その他」は**選択肢に無い文字をそのまま列に入れる**（`item_name` / `uses[].purpose_other`）
- **利用目的は「対象企業 × 目的」の行を何社ぶんでも足せる**＝ `card_payments.uses`（jsonb の
  `[{company_id, company_name, purpose, purpose_other}]`）。読み書きは **`_cardUseList()` /
  `cardUsesValue()` の2つだけ**を通し、描くのは `renderCardUses()` の1箇所
  - 対象企業は **`<datalist>`（`cardCoList`）** なので、打つと候補が絞られ、**名簿に無い企業も手入力できる**。
    候補から選べたときだけ `company_id` も入れる（あとで集計できるように）
  - 一覧に出す文字を作るのは **`_cardUseText()` の1箇所**（「企業名：目的」）
- モーダルは**手動保存**（`_manualSave.cardModal` ／ 接頭辞 `kp_` ／ `sm_card`）。
  保存してもモーダルは閉じない。行の追加・削除は input が出ないので `_cardTouch()` で未保存の印を付ける
  - ⚠️ 接頭辞に `cf_`（候補者）や `ef_`（立替）を使い回さないこと
- 🗑 削除は**論理削除**（`is_deleted` / `deleted_at` / `deleted_by`）。消す前に `_confirmDeleteTwice()` で
  2回たずねる。取得には `is_deleted=not.eq.true` を付ける

#### 📎 領収書（立替管理・カード決済報告で共通）

- 実体は `expenses.receipts` / `card_payments.receipts`（jsonb の `[{url,name,mimeType,size,at,by}]`）。
  **Drive に入れるのは `_uploadReceiptToDrive()` の1箇所だけ**（GAS 経由。両方ここを通る）
  - ⚠️ **GASへ送る中身は、うまく動いているチャットの添付・求人票の写真とまったく同じ形にする**
    （`action` / `driveUrl` / `fileName` / `fileData` / `mimeType` の5つだけ）。
    以前はカード決済だけ `subFolder` にカード名を足していたが、カードごとのフォルダを直に指しているので要らない。
    ほかの経路が送っていないものを1つ足すと、うまくいかないときに切り分けられなくなる
  - ⚠️ **ファイルは入っているのに `success:false` で返ることがある。**
    ドライブ側は「ファイルを作る」→「リンクで見られるようにする」の順で動くので、
    後半だけ失敗すると実物は入っているのにエラーが返る（実際に5個ぶん増えた）。
    **`fileUrl` が返ってきていれば成功としてあつかう**（同じファイルを何度も入れないため）。
    URLも返らなかったときは `err.mayExist` を立てて「入っていることがあります」と知らせる
  - ⚠️ **「アクセスが拒否されました: DriveApp。」の正体はフォルダの共有のしかた。**
    ドライブ連携は入れたファイルを1つずつ「リンクを知っている全員（**閲覧者**）」にするが、
    **フォルダ自体が「リンクを知っている全員（編集者）」だと、ファイルがその共有を受け継いでしまい、
    閲覧者に下げようとして断られる**＝ ファイルは入るのにエラーが返る（実際にそうなった）。
    直し方は2つ。どちらでもよいが、**「編集者」の全体共有のままにしないこと**が要点
      ① フォルダを「リンクを知っている全員（**閲覧者**）」にする（連携が付けるのと同じ権限なので通る）
      ② フォルダを［制限付き］にして、必要な人を個別に足す
    どちらでも、**書き込みにいく `RECEIPT_GAS_ACCOUNT` がオーナーか編集者**である必要がある。
    うまく動いているフォルダ（求人票の写真・チャットの添付）はどれも編集者の全体共有になっていない
  - 文言は **`RECEIPT_SHARE_HINT` の1箇所**。言い換えるのは `_receiptErrText()`、
    画面に出すのは `_receiptSayError()`（フォルダを開くリンク付き）
  - ドライブに書きにいくのは `kmt@k-m-t.jp`（`RECEIPT_GAS_ACCOUNT`）
- 置き場は **`EXPENSE_RECEIPT_DRIVE_URL`（立替）／`CARD_RECEIPT_DRIVE_URL`（カード決済）の1箇所ずつ**
  - カード決済は**カード名のフォルダ**に入れる。フォルダのURLが分かったら
    **`CARD_DRIVE_FOLDERS`（カード名 → フォルダURL）に足すだけ**。空なら親フォルダに入る
    （GAS には `subFolder` にカード名を渡している。対応していないときは親に入る）
- ファイル名はどちらも **「利用日(yyyymmdd) 名目 金額」**（`_expReceiptBaseName` / `_cardReceiptBaseName`）。
  **3つそろっていないと添付させない**（カード決済は利用カードも要る＝保存先が決まらないため）
- 一覧の「領収書」欄と「経理チェック」欄を作るのは
  **`_rcptCellHtml()` / `_acctCellHtml()` の2つだけ**（立替もカード決済も同じ見た目）
- 経理チェックは**押したその場で保存する**（`toggleAcctChecked()` の1箇所。
  `acct_checked` / `acct_checked_at` / `acct_checked_by`。だれがいつ付けたかは ✔ の隣とツールチップに出る）

#### 💵 現金小口のボタン

- 立替管理の［🔄 更新］の左。**リンクは `PETTY_CASH_SHEET_URL` の1箇所だけ**で、
  開くのは `openPettyCashSheet()`（`requireAdmin()` を通す）
- **ボタンは誰にも見えるが、押せるのは管理者だけ**（管理者以外は 🔒 を付けて薄くし、
  押すとトーストで知らせる）。シート自体の閲覧権限は Google 側で決まる

#### 📎 領収書の添付

- 実体は `expenses.receipts`（jsonb の `[{url,name,mimeType,size,at,by}]`）。
  実ファイルは**GAS → Google Drive**（チャットの添付・求人票の写真と同じ経路）。
  **置き場は `EXPENSE_RECEIPT_DRIVE_URL` の1箇所だけ**。最大 `EXPENSE_RECEIPT_MAX`(6) 件
- **Drive でのファイル名は「利用日(yyyymmdd) 名目 金額」**（例 `20260825 航空券代 52000.pdf`）。
  名前を作るのは **`_expReceiptBaseName()` の1箇所**。2件目からは末尾に `(2)` `(3)` を付ける
- **利用日・名目・金額のどれかが空だと添付させない**（ファイル名が作れないため。
  赤字で「先に入れてください」と出す）
- 名前は**添付した時点**で決まる。あとから利用日や金額を直しても Drive のファイル名は変わらない
- 添付・削除しただけでは確定しない（`_expTouch()` で未保存の印 →［💾 保存］で `receipts` を書く）

### 💰 売上管理 月別シートの合計行

- 明細表の一番下（`<tfoot class="sale-total-row">`）に **いま絞り込んでいる `rows` の合計**を出す。
  金額（税抜・税込）・数量・作成／請求／入金の件数と、効いている絞り込みの名前
  （`activeFilterLabels`）が並ぶ。**数え方は `renderSaleMonthTable` の1箇所**
- **上のサマリ（総合計カード・📊 売上内訳）は今までどおり `allSales`＝その月の全体**。
  絞り込んでも変わらない。ここを `rows` に変えると「その月の売上」が分からなくなるので変えないこと
- 絞り込みは4つ（項目 `saleItemFilter` ／ 支払方法 ／ 請求方法 ／ 請求法人）。
  絞り込みが1つでも効いていると見出しが「絞り込みの合計」になる
- **0件のときは合計行を出さない**（「データがありません」の行だけにする）
- 数量は**未入力の行を1件**として数える（新規決定数の数え方とそろえてある）

### 💴 支援委託費の表（人材 × 月）

- セルの操作は2つ。**クリック＝確定の付け外し／ダブルクリック＝編集**。
  クリックはダブルクリックと重なるので `_fmClickTimer` で少し待ってから実行する
  （待っている間にダブルクリックが来たら取り消す）
- **確定＝濃字だけ**（`.fmcell.fixed`）／**未確定＝淡字＋薄い赤の背景**（`.fmcell.unfixed`）。
  CSSはこの2つの1箇所。**確定に線や背景色は付けない**（表がうるさくなって見づらかった）
- 実体は `worker_fee_overrides`（`management_no` ＋ `billing_month` で1行）。
  1セル＝ `{ amount, text_value, confirmed }`
  - `amount` … 金額（数字を入れたとき）。`null` なら計算値のまま
  - `text_value` … **数字以外を入れたとき**（例「2号申請」）。金額は0であつかう
  - 数字かどうかの判定は `fmSaveOverride` の中の1箇所（全角数字・カンマ・円・万に対応）
- 画面での読み方は **`_fmOvOf()` / `_fmOvHasValue()` / `_fmOvAmount()` の3つ**を通す
  （古い「数値だけ」の形も読めるようにしてある）
- 確定を外したときに金額も文字も無い行は消す（`fmToggleConfirm`）
- **記録が無い月は `FEE_CONFIRMED_UNTIL`（2026-07）までを確定ずみ**とみなす。
  判定は **`fmIsFixed()` の1箇所**。既定と同じ状態に戻すだけの操作では行を作らない
- **当月でまだ確定していないセルは赤字＋濃いめの薄赤**（`.fmcell.todo`）。今月の列は薄い青（`.thism`）
  - **背景は `.fmx td.fmcell.unfixed` / `.fmx td.fmcell.todo` と書く。**
    `.fmx td.thism`（今月の列の薄い青）のほうが強いので、`.fmcell.xxx` だけだと
    今月の列で薄赤が消える（実際に消えた）
- **月のセルは幅を固定して折り返す**（`.fmx th.mo,.fmx td.fmcell` の1箇所＝74px・`word-break`）。
  「2号申請中のため停止」のような文字を入れても横に広がらず、全部見える
- **年では切り替えない。** 期間は `fmMonthList()`（24か月前〜12か月先）を作って**横スクロール**。
  開いたときは `fmScrollToDefault()` が**今月の列を一番右**に合わせる（＝1年前あたり〜今月が見える。
  「1年前を左端」にすると今月が見切れるので、右端そろえにしてある）。［📍 今月へ］で戻せる。
  位置合わせは**画面上の位置から測る**（`offsetLeft` は入れ子や余白でずれる）
- **氏名〜在職ステータスの5列は固定**（`.fz1`〜`.fz5`）。年計の列は無し
  - ⚠️ **スマホ（`@media(max-width:640px)`）では氏名・企業名の2列だけ固定にする。**
    5列ぶん（586px）が画面幅を超えると、月の列が固定列の下に隠れて
    **横スクロールしても出てこない**（実際に出てこなかった）
  - この上書きは **`.fmx .fz` の定義より後ろ**に書くこと（同じ強さなので前に書くと負ける）
  - 上の「確定処理依頼」の説明（`#feeMgmtNote`）もスマホではたたむ（タップで開閉）。
    開いたままだと表が画面の下に押し出される
- **氏名を押すと人材の詳細／企業名を押すと所属機関の詳細**が開く。
  リンクを作るのは **`fmOpenLink()` の1箇所**。表全体（`#feeMatrixTable`）に
  クリック＝確定・ダブルクリック＝編集が付いているので、**`event.stopPropagation()` は必須**
- 画面の上の案内文（「確定処理依頼」）は **`renderFeeMgmt` の中の1箇所**にまとめてある
- 絞り込みは メイン担当／サブ担当／**国籍**／フリーワード。選択肢を作るのは
  **`fmFillStaffFilters()` の1箇所**（いま表に出しうる人材から作るので、いない国籍は出ない）。
  国籍の `<option>` は**必ず `value` に生の国籍名を入れる**（表示は旗付きなので、
  value が無いと「🇻🇳 ベトナム」で比較してしまい一致しない）
#### 🔒 月次確定（月のロック）と 🔎 未確定の絞り込み

- **ロックされているかは `_fmIsLocked(m)` の1箇所だけ**（実体は `fee_month_locks`。1行＝1か月）。
  ロックすると**その月は編集できない**（確定の付け外しも金額の書き替えも止める）＝
  `fmToggleConfirm` / `fmSaveOverride` の先頭で見る。セルからは `.ed` を外して `.mlock` にする
- **ロックできるのは `FEE_LOCK_USERS`（経理担当＝白井）＋ 管理者**（`_fmCanLock()` の1箇所。
  名字だけでもフルネームでも合う＝`_renNameIn()` を通す）。
  入口は月の見出しの 🔓/🔒（`fmToggleMonthLock()`）。**未確定が残っていると件数を出して確認する**
- **未確定かどうかは `_fmUnfixedOf(w, m)` の1箇所だけ**。
  絞り込み（「未確定」の選択欄）も、見出しに出る件数（`_fmUnfixedCount`）も同じものを見る
- 表に出す人材を作るのは **`_fmVisibleWorkers()` の1箇所**（描画も件数もここを通る）

#### 表に出さない人材＝「除外者」

- 除外者は2種類あるが、**まとめて `_fmIsExcluded()` の1箇所で判定する**。
  既定ではどちらも出さず、「**除外者を表示**」にチェックを入れると両方まとめて出る

  | 種類 | 実体 | 外し方 |
  |---|---|---|
  | ✕で手動で外した人 | `fee_excluded_workers` | 氏名の ✕ |
  | 退職済で直近1年に記録がない人 | 自動判定 | 自動 |

- **どちらも ↩ で1人ずつ表に戻せる。戻す処理は `fmRestoreWorker()` の1箇所**
  - ✕で外したぶん … `fee_excluded_workers` から消す
  - 自動除外ぶん … **`fee_kept_workers` に入れて残す**（`FEE_KEPT`）。
    自動判定だけだと開き直すたびにまた消えてしまうため
  - 戻した人をもう一度外すのは今までどおり ✕（`fee_excluded_workers` に入る）
- **退職済で「直近1年（今月を含む12か月）」に何も入っていない人材**の判定は
  **`fmHasRecentRecord()` / `_fmRetiredNoRecord()` の1箇所だけ**
  - 「入っている」＝手入力（`_fmOvHasValue`。金額でも文字でも）があるか、実績（`FEE_ACTUALS`）の
    金額があること。**予定（自動計算）は入力ではないので数えない**
  - `FEE_KEPT` に入っている人は自動除外しない（`_fmRetiredNoRecord` の先頭で見る）
  - セルに何か入力しても自動で表に戻る（`fmHasRecentRecord` が true になるため）
- 「除外者を表示」の横に人数の内訳は出さない（ラベルは文字だけ）。
  表から居なくなった人は、チェックを入れれば全部出るので、そちらで確認する

#### 📄 請求書CSV（マネーフォワード クラウド請求書）

支援委託費の表の金額から、そのまま取り込めるCSVを書き出す。
入口は画面右上の［📄 請求書CSV］（`openMfInvoiceCsv`）。月を選ぶ → 一覧で確かめる → 法人ごとに落とす。

- **列の並び・件名・支払期限・敬称・タグ・単価のきまりは `MF_*` の一かたまりだけ**。
  請求内容を組み立てるのは **`mfBuildInvoices(ym)` の1箇所**で、
  **プレビューもCSVも必ずここを見る**（画面と出力が食い違わない）
- 1企業＝請求書1件（「請求書」行1つ＋人数ぶんの「品目」行）。
  金額が0の月は出さない（**マイナス＝返金は出す**）。
  文字が入っているセル（「2号申請」など）は金額0なので出ない＝⚠️で人数を知らせる
- **枝番（`folder_no` に `-`）の企業は親にまとめる**＝工場別に請求書を分けない
  （株式会社YSK（佐賀）／モスニック株式会社茨城工場 など）。
  **判定は `feeParentNo()` / `feeGroupRep()` の1箇所だけ**で、
  **売上管理への同期（`computeCompanyFeeAggregates`）とまったく同じきまり**を使う（必ず同じ内容になる）。
  親になる行は「枝番でない → 請求法人が入っている → 請求タイミングが入っている」の順で選ぶ
  - ⚠️ **会社名では絶対に判定しない。** 以前は売上側だけ `'株式会社YSK'` をベタ書きしていたので、
    企業名を直すと統合が黙って壊れる状態だった
- 🏬 **店舗ごとに請求書を分ける企業は `MF_SPLIT_BY_WORKPLACE` の1箇所だけ**（いま `'203'`＝株式会社ラポール）。
  その企業だけ、**人材の就業場所（`workers.workplace_name`）ごとに1通**にし、取引先名称も店舗名にする
  （「株式会社ラポール（蒲田店）」）。**判定は `folder_no`**（企業名を直しても壊れない）。
  就業場所が空の人材は企業名のまま1通にまとまるので、⚠️で知らせる。
  売上管理への同期（`computeCompanyFeeAggregates`）と送り状は**分けない**（今までどおり1社1行・1社1通）
- **株式会社KMT と 一般社団法人KMT は別のCSV**（`MF_ENTITIES` の1箇所。振込先の口座もここ。
  株式会社＝普通 0683512／一般社団法人＝普通 0693258 と口座が違う）。
  `companies.kmt_corp_type` が空の企業は株式会社KMTに入れ、⚠️で知らせる

| CSVの項目 | 出どころ |
|---|---|
| 取引先名称・敬称 | `companies.name`（法人名なら御中／個人名なら様＝`_mfHonorific()`） |
| 件名 | 請求サイクル × 請求タイミング（`_mfSubjectRange()`）。当月末・初月末＝その月からNか月／終月末＝その月で終わるNか月／翌月末・終月の翌月末＝前の月で終わるNか月 |
| 請求日・売上計上日 | その月の末日 |
| お支払期限 | 振込＝**翌月末**／口座振替＝**引落日**（下の節）。`billing_timing` に「翌々◯日払い」と書いてあればそちらが優先（内村組・カワイ＝翌月末+10日／澤幡製作所＝+5日）。決めるのは `_mfDueDate()` の1箇所 |
| 請求書番号 | `K` + `folder_no`(3桁) + `-` + yyyymm + **通し番号(001から)**。並びはK番号の順 |
| タグ | 請求方法（送付）, 支払方法〔, 税込〕。**`口振` は `口座振替` に直す**（`_mfPayMethod()`）。**書き出したときの所属機関の内容が正**（元のExcelのタグは見ない） |
| 備考・振込先 | 支払方法で決まる（`MF_PAY` の `note` と `MF_ENTITIES` の `bank`。口座振替は振込先「-」） |
| 品名 | `人材名 様` |
| 単価 | その月のセルの金額 ÷ 請求サイクルの月数。**税込請求の企業だけ「×1.1 を10円単位に丸めた額」**（`_mfUnitPrice()`） |
| 数量・単位 | 請求サイクルの月数 ／ `か月` |
| 品目消費税率 | `10%` |

- **企業名を直しても紐づきは壊れない**（人材・チャット・面談記録・売上・書類の履歴などは全部 `company_id`）。
  名前で照合しているのは **決定報告履歴タブ（`_drCoNorm`）の1箇所だけ**で、
  株式会社/(株)・全角半角・スペース・記号の違いは吸収する。
  **カッコ内や工場名を足す／消すときだけ照合キーが変わる**ので、そこだけ確かめる。
  請求書CSVの敬称も名前で決まるので（`_mfHonorific`）、法人格を足すと 様 → 御中 に変わる
- **税込請求かどうかは `companies.tax_included`**（所属機関の【契約・委託費】のチェック）。
  委託費が10円で割り切れない企業（18,182円 など）＝契約が税込のきりのよい金額の企業に付ける
- **Shift_JIS で書き出す。** ブラウザに「Shift_JISで書く」機能は無いので、
  **`TextDecoder('shift_jis')` から逆引きの表を1回だけ作る**（`_sjisTable()` / `sjisBytes()`。
  外部ライブラリを増やさない）。表に無い文字（① 髙 絵文字）は `?` にしてトーストで知らせる
- 出す前の⚠️は **`_mfWarnings()` の1箇所**（送付方法が空／請求法人が空／請求サイクルが空／
  文字が入っている人材／その月の未確定）。1件＝ `{ t:見出し, items:[{label, sub, note, open}] }` で、
  **`items` を付けると［▼ 一覧を見る］がその場に開き、行を押すとその人材・企業が開く**
  （出し方は `_mfWarnHtml()` / `_mfWarnToggle()` の1箇所）
  - ⚠️ **数える相手は「表に出ている人」＝除外者を除いた全員**（`_fmIsExcluded()` の1箇所）。
    `_fmVisibleWorkers()` を見ると担当や国籍の絞り込みで件数と中身が食い違い、
    `FEE_EXCLUDED` だけで見ると**退職済で直近1年に記録がない人**（表に出ていない）まで数えて、
    確定ずみの月なのに「未確定が150件」と出る（実際に出た）
- ⚠️ **モーダルは広くする**（`#mfCsvModal .modal-box` の1箇所＝1180px）。
  既定の幅だと人数・金額の列が切れる

#### 🏦 口座振替スケジュール（集金代行サービス）

代行業者のスケジュール表は**半期ごと**に配られるので、**届いたぶんは登録して正とし、
まだの月だけ計算で出す**（見込み）。一覧は売上管理【🧾 請求情報】の
［🏦 口座振替スケジュール］（`openKozaSchedule`）。

- 実体は **`koza_transfer_schedule`**（**1行＝1月度**。`ym` は引落月）。
  **列は `KOZA_COLS` の1箇所だけ**＝ 口座振替依頼書 到着締切日／確定締切日（＋`confirm_time`）／
  口座振替日（引落日）／回収結果連絡日／回収金入金日
- **日を決めるのは `kozaTransferInfo(請求月)` の1箇所だけ**（請求書CSVの `_mfDueDate` も画面も同じものを見る）
  1. その月度が登録されていれば **`transfer_date` が正**（`fixed:true` ＝ ✅ 確定）
  2. 無ければ **請求月の翌々月4日**（`KOZA_DAY` / `KOZA_MONTHS_AHEAD`）。
     **土日・祝日・年末年始（12/31〜1/3＝`BANK_CLOSED_MMDD`）なら翌営業日**（～ 見込み）
     （例 2026年8月末請求 → 10月4日は日曜なので **10月5日(月)**）
- 請求月 → 月度の読み替えは **`kozaYmOf()` の1箇所**
- **祝日は社内の「📅 年間カレンダー」（`company_calendars.holidays`）が正**（`loadBankHolidays()`）。
  ⚠️ **その年度が未登録のときは土日と年末年始だけで計算する**ので、画面が⚠️で知らせる（`_kozaFyLoaded()`）
- **登録・修正できるのは経理担当と管理者だけ**（`_kozaCanEdit()` ＝ 月次確定と同じ `_fmCanLock()`）。
  ［✏️ スケジュール表を登録・修正］（`openKozaEdit` / `saveKozaSchedule`）で12か月ぶんまとめて入れる。
  **引落日を入れた月だけ「確定」**になり、全部空にすると行ごと消える
- 請求書CSVは、その月がまだ見込みのときに⚠️で知らせる（`_mfWarnings` の中の1箇所）
- 出す範囲は `KOZA_LIST_BACK` / `KOZA_LIST_FWD` の1箇所

#### 📮 送り状（書類送付のご案内）

請求書と一緒に入れる送り状を作る。入口は売上管理【🧾 請求情報】の［📮 送り状］（`openSoujo`）。
**請求方法が「郵送」の企業だけ**が対象で、**1社1通**。

- **郵送かどうかの判定は `_isPostCompany()` の1箇所**（`invoice_method` に「郵送」が入っていれば拾うので、
  「郵送,メール」でも出る）
- **文面は `SOUJO_*` の一かたまり、組み立ては `buildSoujoHtml()` の1箇所だけ**
  （画面のプレビュー・Word・印刷が同じものを見る）
- **同封できる書類は `SOUJO_DOCS` の1箇所**。3つの印で動きが決まる

  | 印 | 意味 |
  |---|---|
  | `auto:'支援委託費'` | 売上リスト（`sales_forecast`）にその月のこの項目があれば、はじめから✓が付く |
  | `koza:false` | 口座振替が使えない書類（在留資格申請費用）。①②／③の分け方に使う |
  | `bill:false` | 請求書ではないもの（領収書・案内）。①②の数え方から外す |

- 自動で付いた✓は**画面で押して足し引きできる**（`soujoToggleDoc`）。月を変えても手で直したぶんは覚えている
- **文面のちがいは支払方法の2つだけ**
  - 振込 … 本文はそのまま
  - 口座振替 … 「口座振替日」（`kozaTransferInfo()`）と「変更の連絡期限」（`_soujoChangeDue()`
    ＝ **確定締切日の前営業日**）が入る。スケジュール表が未登録の月は日付を出さず、
    ひな形どおり「口座振替スケジュールの確定締切日の前営業日の日付」と書く（画面にも⚠️を出す）
- **①②③ を振るのは「口座振替の企業に、口座振替できない書類が混ざるとき」だけ**。
  並びは **口座振替できる請求書 → 在留資格申請費用 → その他** で、本文の `①②` / `③` はこの順から作る
- 在留資格申請費用の注記は **`SOUJO_ZAIRYU_NOTE` の1箇所**
  （口座振替＝「別途お振込みが必要となります。」／振込＝「振込先が異なりますのでご注意ください。」）
- 和暦にするのは **`_soujoWareki()` の1箇所**
- 出力は Word（`soujoWord`）と 印刷・PDF（`soujoPrint`）。どちらも `_soujoFullHtml()` を通る
  - ⚠️ **印刷・PDF は `@page` の余白を 0 にして、余白は `.soujo` の padding で作る。**
    こうしないとブラウザが上下に URL・日付・ページ番号を刷り込んでしまい、そのまま送れない
    （Word は Word 側で余白を持つので今までどおり `@page WordSection1` の margin）
- **改行・空行・下線の位置は元のWordのとおり**（`SOUJO_KOZA_1` / `SOUJO_KOZA_N` の `<br>` と `<u>`、
  `buildSoujoHtml()` の `SP`）。読みやすさが変わるので勝手に詰めない。
  口座振替日は中央寄せ・太字・大きめ（`.kozabi`）、連絡の締切日と ※注記には下線
- ⚠️ **1通がA4 1枚におさまるようにしてある**（書類5点で約253mm・6点で283mm／A4は297mm。
  余白は `.soujo` の padding 18mm）。`_soujoCss()` の行間や空行（`.sp`）をゆるめると2枚に割れる
