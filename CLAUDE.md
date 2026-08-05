# KMT案件管理システム — 開発ガイド

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

- `kmt.html` 以外にファイルを増やさない（単一ファイル構成を維持）
- `.gitignore` は `README.md` と `CLAUDE.md` 以外の `*.md` を除外している。
  作業メモや引継ぎファイルをリポジトリに置いても**コミットされない**。この状態を維持する
- トークン等の秘密情報をリポジトリ内のファイルに書かない
