# 登別 第二住民票プロトタイプ — Claude Code 引き継ぎメモ

このファイルは Claude Code (ターミナル/エディタで動くコーディングエージェント) が
このプロジェクトを開いたときに最初に読む前提のメモです。人間の開発者向けの説明は
`README.md` を参照してください。

## これは何か

北海道登別市向けの「第二住民票」構想のプロトタイプ Web アプリです。頻繁に訪れる
観光客・関係人口が、ボランティア参加・クマ見守り通報・畑作業・記念碑への刻銘などの
「関わり」を通じて第二の住民（第二市民）になれる、という体験を単一の HTML ファイルで
実装しています。ログイン、SNS 風タイムライン、5 言語対応、独自通貨「とむコイン」を
使ったポイント経済（クーポン交換・畑・発送フォーム）まで一通り実装済みです。

**あくまでプロトタイプです。** サーバーは存在せず、認証もデータ保存もすべて
ブラウザの `localStorage` で完結する client-only なシミュレーションです。

## ファイル構成

- `juminhyo.html` — 正本（canonical source）。**まずこのファイルを編集すること。**
- `juminhyo-v2.html` — `juminhyo.html` の完全な複製。キャッシュ回避のため、ユーザーの
  依頼で別 URL に発行し直した際に作られた2本目の公開先です。
  **`juminhyo.html` を編集したら、必ず `cp juminhyo.html juminhyo-v2.html` で
  同期してから両方を確認すること。** 内容が食い違ってはいけません。

このリポジトリ（このディレクトリ）には他にファイルはありません。ビルドツール、
package.json、node_modules は一切不要です — 1ファイル完結の静的 HTML です。

## 技術構成

- 単一 HTML ファイル。`<style>` にインライン CSS、`<script>` にインライン JS。
  外部依存は Google Fonts（Shippori Mincho / Zen Kaku Gothic New）のみ。
- 元は Claude の Artifact 機能で公開されていたため、`<html>/<head>/<body>` の
  ラッパータグは含まれていません（プラットフォームが公開時に付与する想定でした）。
  ローカルで `file://` を直接ブラウザで開いても動作しますが、正式に配布する場合は
  ラッパータグを足すか、静的ホスティング（Netlify, GitHub Pages 等）に置くことを
  検討してください。
- JS は全体が1つの IIFE `(function(){ ... })();` の中に入っています。

### データ層

```js
var STORAGE_KEY = "noboribetsu-db-v3";
var LEGACY_KEY  = "noboribetsu-juminhyo-v1"; // 旧バージョンからの移行用
var LEGACY_KEY2 = "noboribetsu-db-v2";       // 旧バージョンからの移行用

var db = {
  accounts: {},   // idNumber -> account オブジェクト（複数アカウント/端末対応）
  session: null,  // 現在ログイン中の idNumber
  posts: [],      // タイムライン投稿
  lang: "ja"      // 現在の表示言語
};
```

`localStorage` への保存に失敗する環境（プライベートブラウジング等）では
メモリ上の変数にフォールバックします。

各アカウント (`account`) は以下のような構造です:

```js
{
  idNumber, name, pin, registeredAt,
  points: 0,      // 累計獲得とむコイン（生涯ポイント。ランクを決める。減らない）
  balance: 0,     // 使用可能とむコイン（クーポン交換・畑の種代に使う。減る）
  activities: [],  // アクティビティログ（{key, params, time} の形で保存＝翻訳可能）
  jobsDone: [],
  vegetables: {},  // VEGGIES.key -> {planted, stage, harvested}
  reports: [],     // クマ見守り通報
  engravings: [],
  coupons: [],     // 交換済みクーポン
  orders: []       // 畑の収穫物の発送注文
}
```

### 多言語対応 (i18n)

- `var LANGS = ["ja","en","ko","zh","tr"];`
- `var I18N = {...}` — キー毎に `{ja,en,ko,zh,tr}` を持つ巨大な辞書オブジェクト。
- `tr(obj)` — 翻訳オブジェクト（`{ja:"...",en:"...",...}`）を現在言語で解決。
- `t(key, vars)` — `I18N[key]` を `tr()` で解決し、`{placeholder}` 形式で
  文字列補間する。
- `applyI18n()` — `[data-i18n]`（textContent）、`[data-i18n-html]`（innerHTML,
  `<br>` 等タグを含む文言用）、`[data-i18n-ph]`（placeholder 属性）を持つ要素を
  一括で翻訳する。**新しい静的テキストを足すときは、必ずこの3種類のどれかの
  data 属性を付けること。**
- `detectBrowserLang()` — 初回訪問時のみ `navigator.languages` からブラウザの
  言語を自動判定して初期言語にする（`isFirstVisit` が true のときだけ発火）。

**過去ログの再翻訳について（重要な設計判断）:** アクティビティログ
(`s.activities`) やタイムライン投稿は、生成時に翻訳済み文字列を焼き込むのでは
なく、`{key, params}`（ログ）や `spotIdx`/`kindIdx`（通報投稿）という
「未翻訳の構造化データ」として保存しています。表示時に `renderFeedEntry(a)` /
`postDisplayText(p)` が現在の言語で毎回翻訳し直すことで、後から言語を切り替えても
過去のログが正しく再翻訳されます。ユーザーが書いた自由記述の投稿本文だけは
翻訳せず、そのまま (`p.text`) 表示します。**この過去ログ再翻訳の仕組みを壊さない
よう、新しいログ／投稿を追加する際は文字列を直接焼き込まず、必ずキー＋パラメータの
形で保存してください。**

### とむコイン（独自通貨）アイコン

アイコンの実体はリポジトリ同梱の `とむこいん.png`（手描きの金貨イラスト）です。
2480x3508 の原寸から絵の部分（(293,807) から 1894x1894 の正方形）を切り出して
256x256 に縮小し、**`<style>` 内の `.coin-icon,.coin-icon-inline` ルールの
`background:url("data:image/png;base64,...")` として1箇所だけ埋め込んでいます。**
data URI を CSS に置くことで、1ファイル完結を保ちつつ、生成される DOM 文字列は
小さいままにしています。

`coinSvg()` は名前こそ以前の SVG 実装のままですが、現在は
`<span class="coin-icon" aria-hidden="true"></span>` を返すだけです（背景画像は
上記 CSS が描画）。呼び出し側（`#idPoints` / `#idBalance` / `#couponBalance`、
`coinHtml()`）は変更していません。以前あった gradient id 重複回避用の
`coinIconSeq` カウンタは不要になったため削除済みです。

**画像を差し替えるときは** `とむこいん.png` を置き換えるだけでは反映されません。
上記の data URI を作り直して CSS に埋め直す必要があります（PNG はビルド時の素材で
あって、実行時には参照されません）。

`.coin-icon` / `.coin-icon-inline` の CSS サイズは、直近の要望で大きく拡大済み
（30px / 20px）です。以前は 16px / 13px で「小さすぎて見えない」と指摘されました。
親要素（`.n` など）が flex のため、`display:inline-block` は計算値としては
block になりますが、レイアウトは以前の SVG 版と同一です。

**著作権に関する注意:** 過去に、既存の版権キャラクター（ポケモンのコインアートに
酷似した画像）をそのままアイコンにしたいという依頼があり、そのときは著作権上の
リスクを説明して見送りました。現在使っている `とむこいん.png` は、リポジトリに
同梱されたプロジェクト自身の手描き素材としてユーザーの明示的な依頼で採用したもの
です。今後、明らかに他社の版権キャラクターと分かる画像を埋め込む依頼が来た場合は、
改めてリスクを説明してください。

### 主要な JS 関数・エントリポイント（抜粋）

- 認証系: `window.setAuthMode`, `window.registerCitizen`, `window.loginCitizen`,
  `window.logout`, `renderSavedAccounts()`, `window.prefillLogin`
  - 登録フォームは `#regName` / `#regPin` / `#regPinConfirm`（PIN確認）の3つの
    入力が必須です。PIN確認を忘れると `toastPinMismatch` で弾かれます。
- タブ切り替え: `window.showTab`, `updateTabbar()`,
  `var MEMBER_TABS = ["home","rank","coupon","garden","watch","monument"];`
  （未登録ユーザーはこれらにアクセスできず、`info`＝お知らせタブのみ閲覧可能）
- ボランティア/バイト応募: `window.joinJob`
- 畑: `window.plantSeed`, `window.waterPlot`, `window.openShipForm`,
  `window.cancelShipForm`, `window.submitShip`（収穫時に発送先入力フォームが開き、
  注文が作られる）
- クーポン: `window.redeemCoupon`, `window.useCoupon`
- クマ見守り通報 & タイムライン: `window.reportBear`, `createPost(text, opts)`,
  `postDisplayText(p)`, `window.submitPost`, `window.toggleLike`,
  `window.toggleReplyBox`, `window.submitReply`
  （通報は自動的にタイムラインへも投稿されます）
- 記念碑への刻銘: `window.engrave`
- 描画: `function render(){...}` が全ての動的 DOM を再構築するメイン関数。
  必ず `applyI18n()` を先頭で呼び、その後アカウント情報・タブ内容・フィード等を
  差し込みます。**新しい画面要素を足すときはここに描画ロジックを追加してください。**
- 保存: `function save(){ saveDb(); }`

## ローカルでの開発・確認方法

サーバーもビルドも不要です。

```bash
# ブラウザで直接開く
open juminhyo.html   # macOS
# もしくは
xdg-open juminhyo.html  # Linux
```

### JS構文チェック（編集後は必ず実行）

```bash
python3 -c "
import re
content = open('juminhyo.html').read()
m = re.search(r'<script>(.*)</script>', content, re.S)
open('/tmp/script_check.js','w').write(m.group(1))
"
node -e "new Function(require('fs').readFileSync('/tmp/script_check.js','utf8'))" && echo OK
```

### Playwright での動作確認（推奨）

このプロジェクトはこれまで Playwright + Chromium を使ったヘッドレステストで
動作確認してきました。新規登録 → 各タブの操作 → 言語切り替え → コンソール
エラーの有無、を一通り自動チェックするのが効率的です。`page.on('pageerror', ...)`
で JS エラーを検出できます。

## 今後の作業で必ずやること

1. `juminhyo.html` を編集する。
2. JS構文チェック（上記コマンド）を通す。
3. できれば Playwright で主要フローをスモークテストする。
4. `cp juminhyo.html juminhyo-v2.html` で2本目のファイルを同期する。
5. デプロイ先（Claude Artifact / 静的ホスティング等）に両方 反映する。

## 公開されていた URL（旧環境・Claude Artifact）

これまで Claude の Artifact 機能で以下の2つの URL に公開されていました
（このセッションの Claude アカウントに紐づく私有ページです。Claude Code から
直接更新することはできないので、別のホスティング先を用意するか、ユーザーの
Claude アカウントから改めて公開し直す必要があります）。

- https://claude.ai/code/artifact/92468034-a34f-4c0f-8709-fbea09b09948
- https://claude.ai/code/artifact/46d53a12-e963-4a83-8206-bac8afb70626

## 未対応・既知の懸念事項

- コインアイコンは `とむこいん.png` を data URI 化して CSS に埋め込む方式のため、
  PNG を差し替えても自動では反映されません（「とむコイン（独自通貨）アイコン」参照）。
- 認証は完全にクライアントサイドの PIN 方式シミュレーションで、実際のセキュリティは
  ありません（本番運用する場合は要サーバーサイド実装）。
- 管理者アカウント機能は未実装（ユーザーと会話ベースで機能案を検討した段階で、
  実装はまだ行っていません）。
