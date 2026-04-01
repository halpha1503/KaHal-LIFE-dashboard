# Phase 1c — Google Calendar 連携（Next Up カード）設計書

作成日: 2026-04-01

---

## 概要

`index.html` の左カラム下部にある `ph-card`（プレースホルダー）を、Google Calendar API と連携した実際の「Next Up」カードに置き換える。単一ファイル構成を維持しつつ、既存の GitHub Issues 連携パターンに準拠して実装する。

---

## 対象ファイル

- `index.html`（CSS / HTML / JS すべて含む単一ファイル）

---

## HTML 変更

### Next Up カード（`ph-card` → `next-card`）

```html
<div class="next-card" id="next-card">
  <div class="card-eyebrow">Next Up</div>
  <div class="card-heading">予定</div>
  <div class="event-list" id="event-list">
    <div class="card-loading"><div class="spinner"></div>取得中…</div>
  </div>
  <div class="card-footer" id="next-footer"></div>
</div>
```

状態別の `event-list` 中身:

| 状態 | 表示内容 |
|------|---------|
| `config` | 「Google カレンダーを設定してください」（no-config パターン） |
| `loading`（キャッシュなし） | spinner + 取得中… |
| `data` | イベントアイテムのリスト（最大3件） |
| `error`（キャッシュあり） | キャッシュデータ + card-error-badge「オフライン」 |
| 件数0 | 「予定なし」（task-empty パターン） |

### 設定モーダル — Calendar タブ追加

既存タブ（タイマー / テーマ / GitHub）に「Calendar」を追加:

```html
<button class="m-tab" data-tab="calendar">Calendar</button>
```

`pane-calendar` の構成:

1. クライアントID 入力フィールド（`.g-field` / `.g-inp` パターン）
2. 「保存」ボタン（`.g-save` パターン）
3. 認証状態表示エリア `#gcal-auth-status`:
   - 未サインイン時: 「Googleでサインイン」ボタン
   - サインイン済み時: アカウント名 + 「サインアウト」ボタン

---

## CSS 追加

### Next Up カード

```css
.next-card {
  /* .today-card と同等のスタイル（flex-basis は today-card と揃える） */
}
.event-item {
  display: flex; flex-direction: column; gap: 4px;
  padding: 9px 8px; border-radius: 10px;
  border-bottom: 1px solid rgba(255,255,255,.038);
  flex-shrink: 0;
}
.event-item:last-child { border-bottom: none; }
.event-time {
  font-size: 10px; font-weight: 500; letter-spacing: .06em;
  color: var(--accent);           /* #f0a830 */
}
.event-name {
  font-size: 12.5px; font-weight: 400;
  color: var(--text-1); line-height: 1.35;
}
```

### Calendar 設定ペイン

既存の `.g-field` / `.g-inp` / `.g-save` を流用。

サインインボタン（`.c-signin`）:

```css
.c-signin {
  /* g-save と同等だが accent 色を Google ブランドカラー (#4285f4) に変更 */
}
.c-signout {
  /* g-save と同等だが薄いグレー系 */
}
.c-account-name {
  font-size: 12px; color: var(--text-2); margin-bottom: 10px;
}
```

---

## JavaScript 追加

### 定数

```js
const GCAL_CACHE_KEY    = 'kahal_gcal_events_cache';
const GCAL_CONFIG_KEY   = 'kahal_gcal';          // { clientId }
const GCAL_TOKEN_KEY    = 'kahal_gcal_token';    // { access_token, expiry_ms }
const GCAL_REFRESH_MS   = 30 * 60 * 1000;        // 30分
const GCAL_SCOPE        = 'https://www.googleapis.com/auth/calendar.readonly';
```

### ストレージ関数

```js
loadGcalConfig()   → { clientId } | null
saveGcalConfig(cfg)
loadGcalToken()    → { access_token, expiry_ms } | null   (期限切れなら null)
saveGcalToken(tokenResponse)
clearGcalToken()
loadCachedEvents() → events[] | null
cacheEvents(events)
getCachedEventsUpdatedAt() → ISO文字列 | null
```

### GIS ロード＆初期化

```js
function loadGIS()
  // <script src="https://accounts.google.com/gsi/client"> を動的に挿入
  // onload で initGIS() を呼ぶ

function initGIS(clientId)
  // google.accounts.oauth2.initTokenClient({
  //   client_id, scope, callback: onTokenResponse
  // })

function onTokenResponse(response)
  // エラー時: console.warn してサインアウト状態に戻す
  // 成功時: saveGcalToken(response), fetchUpcomingEvents()
```

### 認証フロー

```js
function signInGCal()
  // tokenClient.requestAccessToken()

function signOutGCal()
  // clearGcalToken(), renderNextCard(null, 'config'), syncCalendarPane()
```

### Calendar API 取得

```js
async function fetchUpcomingEvents()
  // 前提確認: clientId が設定済み + token が有効
  // → 未設定なら renderNextCard(null, 'config'); return
  // → token なし: renderNextCard(cached, 'config'); return
  // キャッシュがないなら loading 表示
  // GET https://www.googleapis.com/calendar/v3/calendars/primary/events
  //   ?maxResults=3&orderBy=startTime&singleEvents=true
  //   &timeMin=<now ISO>
  //   Authorization: Bearer <access_token>
  // 成功: cacheEvents(events), renderNextCard(events, null)
  // 失敗: console.warn, renderNextCard(cached, 'error')
  //   401の場合は clearGcalToken() も実行
```

### 日時フォーマット

```js
function formatEventTime(start)
  // start: { dateTime } | { date } (終日)
  // todayStr() を使って今日/明日/それ以降を判定
  // 「今日 14:00」「明日 09:30」「4/3」(終日) or「4/3 10:00」
```

### カード描画

```js
function renderNextCard(events, state)
  // state: null | 'config' | 'loading' | 'error'
  // GitHub の renderTodayCard() と同等パターン
  // フッタ: getCachedEventsUpdatedAt() → formatFetchedAt() (既存関数を流用)
```

### 設定ペイン同期

```js
function syncCalendarPane()
  // openModal() から呼ぶ (syncGithubInputs() と同列)
  // clientId input にロード済み値を反映
  // 認証状態に応じて #gcal-auth-status を更新
```

### インターバル

```js
setInterval(fetchUpcomingEvents, GCAL_REFRESH_MS);
// 既存の setInterval(checkDayReset, 60_000) と並走
```

### Init

```js
// 既存の refreshTodayIssues() の後に追加
const gcalCfg = loadGcalConfig();
if (gcalCfg?.clientId) loadGIS();
```

---

## 認証状態フロー

```
App起動
  ├─ clientId なし → renderNextCard(null, 'config')
  └─ clientId あり → loadGIS() → initGIS(clientId)
        ├─ token なし → renderNextCard(null, 'config')
        └─ token 有効 → fetchUpcomingEvents()

設定ペイン「Googleでサインイン」タップ
  → tokenClient.requestAccessToken()
  → onTokenResponse()
  → saveGcalToken() → fetchUpcomingEvents() → renderNextCard()

設定ペイン「サインアウト」タップ
  → clearGcalToken() → renderNextCard(null, 'config')
```

---

## エラーハンドリング

| 状況 | 対応 |
|------|------|
| API 401 | `clearGcalToken()` + config 状態表示 |
| API その他エラー | キャッシュ表示 + error バッジ |
| GIS ロード失敗 | `console.warn`、設定ペインにエラー表示 |
| 終日イベント（`date` のみ）| 日付のみ表示（時刻なし） |

---

## localStorage キー一覧（追加分）

| キー | 内容 |
|------|------|
| `kahal_gcal` | `{ clientId: string }` |
| `kahal_gcal_token` | `{ access_token: string, expiry_ms: number }` |
| `kahal_gcal_events_cache` | `{ events: [], fetchedAt: ISO文字列 }` |

---

## 変更しないこと

- 単一ファイル構成（`index.html`）
- 既存機能（F1/F3/F4/F5）への影響なし
- 新ライブラリの追加なし（GIS は CDN 動的ロード）
- `todayStr()` / `formatFetchedAt()` / `.card-error-badge` など既存ユーティリティを再利用
