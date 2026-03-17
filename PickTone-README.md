# PickTone — プロジェクトドキュメント

## 概要

PickToneは、ミュージシャン向けの練習プラットフォーム。クロマチックチューナーとメトロノームを基本機能とし、将来的にバッキングトラック再生、スケール可視化、AI演奏分析へ拡張予定。Web版（SEO集客）＋ネイティブアプリ（Android / Windows）のマルチプラットフォーム展開。

**URL:** https://picktone.org

**対象ユーザー:** ギタリスト、ベーシスト、その他の楽器プレイヤー（英語圏 + 日本語圏）

---

## プラットフォーム戦略

| プラットフォーム | 役割 | 技術 |
|------------------|------|------|
| Web（ブラウザ） | SEO集客のランディング。基本機能は無料で使える。アプリへの誘導 | HTML / CSS / JS（vanilla） |
| Android | メインのモバイルアプリ。フル機能 | React Native (Expo) |
| Windows | デスクトップアプリ。PC練習環境向け | React Native (Expo for Windows) |
| iOS | 将来対応（Apple Developer Program $99/年 + Mac必須のため後回し） | — |
| macOS | 将来対応（iOSと同タイミング） | — |

Web版は現状の単一HTMLファイルをそのまま維持。アプリ版はExpoベースで開発し、コアロジック（ピッチ検出、メトロノームスケジューリング）は共通モジュール化する。

---

## 技術スタック

### Web版（Phase 1、現在）

- **言語:** HTML / CSS / JavaScript（vanilla、フレームワークなし）
- **音声処理:** Web Audio API（`AudioContext`, `AnalyserNode`, `OscillatorNode`）+ soundfont-player（バッキング用ギター音源）
- **マイク入力:** `navigator.mediaDevices.getUserMedia`
- **フォント:** Google Fonts（Orbitron, Chakra Petch）
- **永続化:** localStorage
- **ホスティング:** Cloudflare Pages（Workers）
- **サーバーサイド:** 不要

### アプリ版（Phase 2以降）

- **フレームワーク:** React Native + Expo
- **言語:** TypeScript
- **音声処理:** expo-av + カスタムネイティブモジュール（ピッチ検出用）
- **認証:** Google Sign-In（Supabase Auth経由）
- **バックエンド/DB:** Supabase（PostgreSQL）
- **主なSupabase利用用途:**
  - Google OAuth認証
  - ユーザー設定のクラウド同期（BPM、拍子、A4基準音等）
  - 練習履歴の保存（Phase 3）
  - AI分析結果の保存（Phase 4）
  - 課金ステータス管理
- **決済:** Stripe（Web課金）/ Google Play Billing（Androidアプリ内課金）
- **AI分析:** Anthropic Claude API（Phase 4）
- **CI/CD:** EAS Build（Expo Application Services）

### 認証フロー

```
ユーザー → Googleログインボタン → Supabase Auth（Google OAuth）→ JWTトークン発行 → アプリ/Web
```

Googleログインのみに絞ることで、メールアドレス確認・パスワードリセット等の実装が不要。Supabase AuthのGoogle providerを有効にするだけで動く。

### データモデル（Supabase PostgreSQL）

```sql
-- ユーザー（Supabase Authが自動作成）
auth.users

-- ユーザー設定
user_settings (
  user_id     uuid references auth.users primary key,
  lang        text default 'en',
  bpm         int default 120,
  ts_num      int default 4,
  ts_den      int default 4,
  vol         float default 0.8,
  ref_hz      int default 440,
  met_first   boolean default false,
  updated_at  timestamptz
)

-- 練習セッション（Phase 3）
practice_sessions (
  id          uuid primary key,
  user_id     uuid references auth.users,
  started_at  timestamptz,
  ended_at    timestamptz,
  bpm         int,
  key         text,
  ts          text,
  notes       text
)

-- AI分析結果（Phase 4）
ai_analyses (
  id          uuid primary key,
  user_id     uuid references auth.users,
  session_id  uuid references practice_sessions,
  audio_url   text,
  feedback    jsonb,
  created_at  timestamptz
)
```

---

## 機能一覧

### 1. クロマチックチューナー

| 項目 | 詳細 |
|------|------|
| ピッチ検出方式 | 自己相関（autocorrelation）アルゴリズム |
| 入力 | デバイスのマイク or オーディオインターフェース経由 |
| 表示 | 音名、オクターブ、セント差（±）、周波数（Hz） |
| ゲージ | 針式SVGゲージ。中央緑ゾーン（±8°）、左右赤ゾーン |
| 合格判定 | ±5cents以内で緑表示 |
| 基準音 | A4 = 420〜460Hz（デフォルト440Hz、数値入力で変更可能） |
| 対応楽器 | クロマチックなので全楽器対応 |

### 2. メトロノーム

| 項目 | 詳細 |
|------|------|
| BPM範囲 | 20〜300 |
| BPM入力 | 数値直接入力 + スライダー |
| 拍子プリセット | 2/4, 3/4, 4/4, 5/4, 6/8, 7/8 |
| クリック音 | Web Audio APIのOscillatorNode。1拍目: 1200Hz、他: 800Hz |
| 音量 | 0〜100%スライダーで調整 |
| タップテンポ | TAPボタンを数回叩いてBPM自動算出（直近5回の間隔から平均） |
| ミュートビート | 各拍の丸ドットをタップでミュート/解除。ミュート中も枠は光る（視覚フィードバック維持） |
| ビート表示 | 均一サイズの丸ドット。1拍目はアンバー、他は緑で光る |
| スケジューリング | `setTimeout` + `AudioContext.currentTime`による先読みスケジューリング（ドロップアウト防止） |

### 3. UI/UX

| 項目 | 詳細 |
|------|------|
| デザインコンセプト | ハードウェア機材風（BOSSペダル/KORGメトロノーム的） |
| カラーテーマ | ダーク（メタルエンクロージャー風） |
| フォント | Orbitron（LCD/セグメント表示）、Chakra Petch（UI） |
| LCD表示 | 走査線オーバーレイ付き黒背景パネル |
| ボタン | ストンプスイッチ風（押し込みエフェクト付き） |
| LED | radial-gradient + glowで実LED風 |
| 装飾 | コーナーにネジ穴ディテール |
| パネル入替 | ヘッダーの⇅ボタンでチューナー/メトロノームの上下切替（モバイルのみ） |
| レスポンシブ | モバイル: シングルカラム。PC（760px以上）: チューナー/メトロノーム横並び2カラム |
| レイアウト | キーパネル最上部 → チューナー/メトロノーム横並び → フッター |
| タッチ対応 | `touch-action: manipulation`、`-webkit-tap-highlight-color: transparent` |

### 4. 多言語対応（i18n）

| 項目 | 詳細 |
|------|------|
| 対応言語 | 英語 / 日本語 |
| 初期言語 | `navigator.language`でブラウザロケール自動判定 |
| 切替 | ヘッダーのボタンで手動切替 |
| 保存 | 手動切替後はlocalStorageに保存、次回以降はその設定を優先 |
| 対象 | 全UIラベル、ボタンテキスト、ヘルプ文、ミュートヒント |

### 5. 設定の永続化（localStorage）

保存キー: `picktone_prefs`

| 保存項目 | デフォルト値 |
|----------|-------------|
| lang | ブラウザロケール |
| bpm | 120 |
| tsNum（拍子の分子） | 4 |
| tsDen（拍子の分母） | 4 |
| customTs | false |
| vol | 0.8 |
| refHz（A4基準音） | 440 |
| metFirst（パネル順序） | false |
| selKey（選択キー） | 0 (C) |
| selMode（メジャー/マイナー） | 'maj' |

ミュートビートのパターンとスケールモード（指板表示用）はセッション限り（保存しない）。

### 6. キー/スケール選択

| 項目 | 詳細 |
|------|------|
| キー選択 | 12キー（C〜B）のグリッドボタン。シャープキーは小さめフォント |
| モード | メジャー / マイナー切替 |
| スケール表示 | LCD内にスケール構成音を表示（チューナー検出音と連動ハイライト） |
| スケール外警告 | チューナーで検出した音がスケール外の場合、音名を赤表示+「OUT」タグ |
| 度数表示 | スケール内の音はローマ数字の度数（I, ii, iii等）をタグ表示 |
| レイアウト | ページ最上部に配置。PC/モバイル共通 |

### 7. ダイアトニックコード＆コード進行

| 項目 | 詳細 |
|------|------|
| ダイアトニックコード | 選択キーのI〜VIIコードをチップ形式で一覧表示 |
| コード進行プリセット | メジャー: Pop/Rock, Axis, 50s, Canon, Blues。マイナー: Pop Minor, Andalusian, Sad, Minor Blues |
| バッキング再生 | プログレッション行の▶ボタンでメトロノーム+コードストローク同時スタート |
| 音源 | soundfont-player (acoustic_guitar_steel, MusyngKite) |
| ストロークパターン | 8ビート: ↓↓↓↑_↓↓↑（ゴーストノート含む） |
| 自然化 | 6弦ボイシング（3バリエーション/コード）、±12msタイミングゆらぎ、ベロシティ変動±15%、スウィング6%、ストローク速度ランダム |
| BPMシーク対応 | スライダードラッグ中は再生停止、離すと自動再開 |

### 8. ギター指板スケール表示

| 項目 | 詳細 |
|------|------|
| 表示範囲 | 6弦（E2〜E4標準チューニング）× 15フレット |
| スケールモード | メジャー / マイナー / ペンタトニック / マイナーペンタトニック |
| 度数表示 | 各音に度数ラベル（R, 2, 3, 4, 5, 6, 7 / ♭3, ♭7等） |
| ルート強調 | ルート音はアクセントカラーでハイライト |
| レスポンシブ | 横スクロール対応（min-width: 420px） |

### 9. ヘルプ表示

| 画面 | 表示方式 |
|------|----------|
| チューナー（PC） | パネル下部にインラインテキスト。A4基準音とI/F対応の説明 |
| チューナー（スマホ） | 非表示 |
| メトロノーム | パネル下部にインラインテキスト。BPM設定、ミュートビート、タップテンポの説明 |

---

## リポジトリ構成

### 現在（Phase 1: Web版のみ）

```
picktone-app/
├── public/                      ← Cloudflare Pagesデプロイ対象
│   ├── index.html               ← メインアプリ（全機能内蔵）
│   ├── about.html               ← Aboutページ
│   ├── privacy.html             ← Privacy Policyページ
│   ├── sitemap.xml              ← サイトマップ
│   └── robots.txt               ← クローラー設定
├── PickTone-README.md           ← このドキュメント
├── LICENSE                      ← MIT License
├── wrangler.toml                ← Cloudflare Workers設定
└── .gitignore
```

### 将来（Phase 2以降: マルチプラットフォーム）

```
picktone/
├── web/                          ← Web版（SEO集客用）
│   ├── index.html
│   ├── about.html
│   ├── privacy.html
│   ├── guide.html
│   ├── sitemap.xml
│   ├── robots.txt
│   ├── og-image.png
│   └── favicon.ico
├── app/                          ← React Native (Expo) アプリ
│   ├── app/                      ← Expo Routerのページ
│   ├── components/               ← UIコンポーネント
│   ├── core/                     ← コアロジック（共通）
│   │   ├── pitch-detection.ts    ← autocorrelationアルゴリズム
│   │   ├── metronome-engine.ts   ← オーディオスケジューリング
│   │   ├── bpm-detector.ts       ← onset detection
│   │   ├── scale-utils.ts        ← スケール/コード理論
│   │   └── backing-engine.ts     ← バッキングトラック生成
│   ├── services/
│   │   ├── supabase.ts           ← Supabaseクライアント
│   │   ├── auth.ts               ← Google認証
│   │   └── ai-analysis.ts       ← Claude API連携
│   ├── app.json
│   ├── package.json
│   └── tsconfig.json
├── supabase/                     ← Supabaseマイグレーション
│   └── migrations/
├── docs/
│   └── README.md
└── .gitignore
```

---

## レスポンシブ仕様

| ブレークポイント | 動作 |
|------------------|------|
| < 760px（スマホ） | シングルカラム。チューナー説明非表示。パネル入替ボタン表示 |
| ≥ 760px（PC） | 横並び2カラム（max-width: 860px）。チューナー説明表示。パネル入替ボタン非表示 |
| ≥ 1100px（ワイドPC） | max-width: 960px。フォントサイズ拡大 |

---

## 既知の制限事項

1. **マイクアクセス**: `https://` または `localhost` でのみ動作。`file://` や `content://` では `getUserMedia` がブロックされる
2. **iOS Safari**: AudioContextの自動再生制限あり。ユーザーのタップで `resume()` する必要がある（対応済み）
3. **分母表示**: 拍子の分母は表示のみで動作に影響しない（業界慣例に合わせた仕様）

---

## プロダクトロードマップ

### Phase 1: PickTone基本版 ✅ 完了

チューナー＋メトロノームの公開・収益化。

| タスク | 状態 |
|--------|------|
| クロマチックチューナー（針式、A4基準音変更） | ✅ 実装済み |
| メトロノーム（BPM入力、拍子、ミュートビート、タップテンポ、音量） | ✅ 実装済み |
| ハードウェア風UIデザイン | ✅ 実装済み |
| 日英多言語対応（ロケール自動判定） | ✅ 実装済み |
| localStorage設定保存 | ✅ 実装済み |
| パネル入替（⇅、モバイルのみ） | ✅ 実装済み |
| インラインヘルプ（レスポンシブ） | ✅ 実装済み |
| PCレイアウト（横並び2カラム） | ✅ 実装済み |
| ドメイン取得（picktone.org） | ✅ 完了 |
| Cloudflare Pagesデプロイ | ✅ 完了 |
| About / Privacy Policyページ作成 | ✅ 完了 |
| SEO最適化（title, meta, OGP, sitemap） | ✅ 完了 |
| Google Search Console登録 | ✅ 完了（サイトマップ送信済み、インデックス登録済み） |
| Google AdSense申請 | ⏳ 審査待ち（コード埋め込み済み） |

SEOターゲットキーワード: "online tuner", "online metronome", "free chromatic tuner", "chromatic tuner metronome", "チューナー オンライン", "メトロノーム オンライン", "無料 チューナー"

広告配置案: メトロノームパネル下 or パネル間。代替としてBuySellAds / Carbon Ads。

---

### Phase 2: バッキングトラック＋スケールガイド＋アプリ版立ち上げ ← 現在ここ（Web機能完了、アプリ版未着手）

ギタリストの即興練習を支援する機能追加と、Expoアプリの初期構築を同時進行。

**2a. 機能追加（Web版 → アプリ版に共通）**

| 機能 | 概要 | 技術 | 状態 |
|------|------|------|------|
| キー/スケール選択 | 12キー × メジャー/マイナーの選択UI | JSの状態管理 | ✅ 実装済み |
| ダイアトニックコード表示 | 選択キーのI〜VIIコードをチップ表示 | 音楽理論をJSで実装 | ✅ 実装済み |
| コード進行プリセット | Pop/Rock, Axis, 50s, Canon, Blues等の定番進行 | 配列ベース | ✅ 実装済み |
| バッキング再生 | プログレッション行の▶でメトロノーム+コードストローク同時再生 | soundfont-player（acoustic_guitar_steel） | ✅ 実装済み |
| バッキング自然化 | 6弦ボイシング、タイミングゆらぎ±12ms、ベロシティ変動、スウィング | Web Audio API humanization | ✅ 実装済み |
| 指板スケール表示 | 6弦×15フレットのギター指板にスケール音を度数付きで表示 | HTML table + CSS | ✅ 実装済み |
| スケールモード切替 | メジャー/マイナー/ペンタ/マイナーペンタの4モード | SCALE_MODES定数 | ✅ 実装済み |
| スケールマップ | チューナー検出音をスケール表示上でリアルタイムハイライト | ピッチ検出＋スケールマッチング | ✅ 実装済み |
| スケール外警告 | スケール外の音を弾くと赤表示+OUTタグ | degTag + CSS | ✅ 実装済み |

**2b. アプリ版の基盤構築**

| タスク | 詳細 |
|--------|------|
| Expoプロジェクト初期化 | `npx create-expo-app picktone-app` |
| Supabaseプロジェクト作成 | Google OAuth設定、user_settingsテーブル作成 |
| Google Sign-In実装 | Supabase Auth + expo-auth-session |
| Phase 1機能のアプリ移植 | チューナー、メトロノーム |
| 設定のクラウド同期 | Supabaseのuser_settingsとlocalの双方向同期 |
| Android APKビルド | EAS Build |
| Windowsビルド | Expo for Windows (react-native-windows) |
| Google Play Store申請 | 開発者アカウント（$25一回） |
| Microsoft Store申請 | 開発者アカウント（個人$19一回） |

想定される追加SEOキーワード: "online backing track", "guitar scale practice", "scale visualizer", "ギター スケール 練習", "バッキングトラック オンライン"

UI構成案:
- チューナーパネル（既存）
- メトロノームパネル（既存）→ バッキングトラック再生ボタンと統合、またはタブ切替
- スケールマップパネル（新規）→ ギターの指板をSVGで表示、光る音を示す

---

### Phase 3: 練習記録・履歴

ユーザーの継続利用を促進するトラッキング機能。Supabaseにデータを保存し、デバイス間で同期。

| 機能 | 概要 | 技術 |
|------|------|------|
| 練習セッション記録 | 開始/終了時刻、使った機能、BPM、キー | Supabase DB (practice_sessions) |
| 練習時間グラフ | 日別/週別の練習時間を棒グラフで可視化 | SVG / react-native-svg |
| BPM推移 | 特定曲のBPMを日付ごとに記録し上達を可視化 | Supabase + グラフ |
| よく使うキー・拍子 | 統計表示 | 集計クエリ |
| オフライン対応 | ネット未接続時はローカルに保存、復帰時に同期 | AsyncStorage + Supabase sync |

Web版（未ログイン）ではlocalStorage/IndexedDBにフォールバック。ログイン済みならSupabaseに保存。

---

### Phase 4: フレーズ録音＋AI分析（課金導入）

AIによる演奏フィードバック。ここからフリーミアムモデルに移行。

| 機能 | 概要 | 技術 |
|------|------|------|
| フレーズ録音 | マイク入力を録音 | MediaRecorder API（Web）/ expo-av（アプリ） |
| 録音再生 | 録った音を再生して確認 | Audio要素 / expo-av |
| 音声アップロード | 録音をSupabase Storageにアップ | Supabase Storage |
| AI分析 | 録音データをClaude APIに送り、演奏のフィードバックを取得 | Anthropic Messages API（サーバーサイドで実行、Supabase Edge Functions） |
| 分析内容 | スケール適合度、リズム安定性、フレーズの特徴、改善提案 | プロンプトエンジニアリング |
| 分析履歴 | 過去の分析結果を一覧表示 | Supabase DB (ai_analyses) |

収益モデル:

| プラン | 価格案 | 内容 |
|--------|--------|------|
| Free | ¥0 | チューナー、メトロノーム、バッキング、スケール可視化、練習記録。すべて無制限。広告あり |
| Pro | ¥500〜800/月 | AI分析 月30回、広告非表示、データクラウド同期（将来） |
| Unlimited | ¥1,500/月 | AI分析 無制限、優先サポート |

API費用の見積もり（Claude Sonnet想定）: 1回の分析あたり入力トークン〜2000 + 出力トークン〜1000 → 約¥1〜3/回。Pro月30回で¥30〜90のAPI費用。十分にマージンが取れる。

決済: Stripe（国際対応、開発者フレンドリー）

---

### 将来の機能拡張案（Phase 5以降）

- クリック音のカスタマイズ（ウッドブロック、ハイハット、リムショット等のサンプル音）
- メトロノームプリセット保存（BPM＋拍子＋ミュートパターンのセット、曲名付き）
- PWA対応（オフライン動作、ホーム画面に追加）
- ダークモード/ライトモード切替
- キーボードショートカット（PC: スペースで再生/停止、矢印でBPM増減）
- コード進行のカスタム編集（ユーザーが自分でコード進行を入力）
- ドラムパターン再生（8ビート、16ビート、ボサノバ等のプリセット）
- MIDI入力対応（Web MIDI API）
- マルチデバイス同期（WebSocket経由でバンド練習用に複数端末で同期メトロノーム）
- アカウント機能＋クラウド保存（Firebase等）

---

## 開発メモ

### メトロノームの音が鳴らない問題（解決済み）
原因: `gain.gain.exponentialRampToValueAtTime()` を呼ぶ前に `gain.gain.setValueAtTime()` でベース値を設定する必要があった。
対処: `setValueAtTime(value, time)` → `exponentialRampToValueAtTime(0.001, time + 0.08)` の順序を徹底。

### スマホでボタンが押せない問題（解決済み）
原因: SVGゲージがタッチイベントを吸収していた。`content://` プロトコルではWeb APIがブロックされる。
対処: SVGに `pointer-events: none` を追加。`http://` でのアクセスを必須とする。

### autocorrelationのエッジケース（対処済み）
`maxpos` が配列の端に来た場合の放物線補間でindex out of rangeが発生する可能性があったため、ガード条件を追加。
