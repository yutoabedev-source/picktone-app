# TuneBeat — プロジェクトドキュメント

## 概要

TuneBeatは、ミュージシャン向けの練習プラットフォーム。クロマチックチューナー、メトロノーム、BPM検知を基本機能とし、将来的にバッキングトラック再生、スケール可視化、AI演奏分析へ拡張予定。Web版（SEO集客）＋ネイティブアプリ（Android / Windows）のマルチプラットフォーム展開。

**ターゲットURL案:** `tunebeat.app` / `tunebeat.io`

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

Web版は現状の単一HTMLファイルをそのまま維持。アプリ版はExpoベースで開発し、コアロジック（ピッチ検出、メトロノームスケジューリング、BPM検出）は共通モジュール化する。

---

## 技術スタック

### Web版（Phase 1、現在）

- **言語:** HTML / CSS / JavaScript（vanilla、フレームワークなし）
- **音声処理:** Web Audio API（`AudioContext`, `AnalyserNode`, `OscillatorNode`）
- **マイク入力:** `navigator.mediaDevices.getUserMedia`
- **フォント:** Google Fonts（Orbitron, Chakra Petch）
- **永続化:** localStorage
- **ホスティング:** Cloudflare Pages / Vercel / Netlify（静的HTML）
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

### 3. BPM自動検知

| 項目 | 詳細 |
|------|------|
| 方式 | onset detection（エネルギーベースのビート検出） |
| 解析帯域 | 60Hz〜4000Hz（キック/スネア帯域） |
| 閾値 | 動的（直近60フレームの平均エネルギー × 1.4 + 8） |
| 安定化 | 直近10サンプルの最頻値（mode）で揺れを抑制 |
| BPM範囲 | 40〜220（半分/倍テンポの自動正規化あり） |
| 信頼度表示 | 最頻値の割合をパーセントで表示 |
| 波形表示 | Canvas要素でリアルタイム波形を描画 |
| 適用 | APPLYボタンでメトロノームのBPMに反映 |
| UI配置 | メトロノームパネル内に統合。スマホでは折りたたみ式 |

### 4. UI/UX

| 項目 | 詳細 |
|------|------|
| デザインコンセプト | ハードウェア機材風（BOSSペダル/KORGメトロノーム的） |
| カラーテーマ | ダーク（メタルエンクロージャー風） |
| フォント | Orbitron（LCD/セグメント表示）、Chakra Petch（UI） |
| LCD表示 | 走査線オーバーレイ付き黒背景パネル |
| ボタン | ストンプスイッチ風（押し込みエフェクト付き） |
| LED | radial-gradient + glowで実LED風 |
| 装飾 | コーナーにネジ穴ディテール |
| パネル入替 | ヘッダーの⇅ボタンでチューナー/メトロノームの上下切替 |
| レスポンシブ | max-width: 400pxベース。720px以上でPC向け表示 |
| タッチ対応 | `touch-action: manipulation`、`-webkit-tap-highlight-color: transparent` |

### 5. 多言語対応（i18n）

| 項目 | 詳細 |
|------|------|
| 対応言語 | 英語 / 日本語 |
| 初期言語 | `navigator.language`でブラウザロケール自動判定 |
| 切替 | ヘッダーのボタンで手動切替 |
| 保存 | 手動切替後はlocalStorageに保存、次回以降はその設定を優先 |
| 対象 | 全UIラベル、ボタンテキスト、ヘルプ文、ミュートヒント |

### 6. 設定の永続化（localStorage）

保存キー: `tunebeat_prefs`

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

ミュートビートのパターンはセッション限り（保存しない）。

### 7. ヘルプ表示

| 画面 | 表示方式 |
|------|----------|
| チューナー（PC） | パネル下部にインラインテキスト。A4基準音とI/F対応の説明 |
| チューナー（スマホ） | 非表示 |
| メトロノーム | パネル下部にインラインテキスト。BPM設定、ミュートビート、タップテンポ、BPM自動検知の説明 |

---

## リポジトリ構成

### 現在（Phase 1: Web版のみ）

```
tunebeat/
├── web/
│   └── tunebeat.html        ← 単一HTMLファイル（全機能内蔵）
├── docs/
│   └── README.md            ← このドキュメント
└── .gitignore
```

### 将来（Phase 2以降: マルチプラットフォーム）

```
tunebeat/
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
| < 720px（スマホ） | シングルカラム。チューナー説明非表示。BPM検知は折りたたみ |
| ≥ 720px（PC） | シングルカラム（max-width: 400px）。チューナー説明表示。BPM検知は常時展開。折りたたみ矢印非表示 |

---

## 既知の制限事項

1. **マイクアクセス**: `https://` または `localhost` でのみ動作。`file://` や `content://` では `getUserMedia` がブロックされる
2. **BPM検知精度**: 環境ノイズが多い場合やリズムが不明瞭な曲では精度が下がる
3. **ブラウザ間の音声**: 別タブの音声（Spotify、YouTube等）は直接取得できない。スピーカー出力をマイクで拾う方式
4. **iOS Safari**: AudioContextの自動再生制限あり。ユーザーのタップで `resume()` する必要がある（対応済み）
5. **分母表示**: 拍子の分母は表示のみで動作に影響しない（業界慣例に合わせた仕様）

---

## プロダクトロードマップ

### Phase 1: TuneBeat基本版 ← 現在ここ

チューナー＋メトロノーム＋BPM検知の公開・収益化。

| タスク | 状態 |
|--------|------|
| クロマチックチューナー（針式、A4基準音変更） | ✅ 実装済み |
| メトロノーム（BPM入力、拍子、ミュートビート、タップテンポ、音量） | ✅ 実装済み |
| BPM自動検知（onset detection） | ✅ 実装済み |
| ハードウェア風UIデザイン | ✅ 実装済み |
| 日英多言語対応（ロケール自動判定） | ✅ 実装済み |
| localStorage設定保存 | ✅ 実装済み |
| パネル入替（⇅） | ✅ 実装済み |
| インラインヘルプ（レスポンシブ） | ✅ 実装済み |
| BPM検知の折りたたみ（スマホ） | ✅ 実装済み |
| PCでの動作確認・バグ修正 | 🔲 未着手 |
| ドメイン取得・デプロイ | 🔲 未着手 |
| About / Privacy Policy / ガイドページ作成 | 🔲 未着手 |
| SEO最適化（title, meta, OGP, sitemap） | 🔲 未着手 |
| Google Search Console登録 | 🔲 未着手 |
| Google AdSense申請 | 🔲 未着手 |

SEOターゲットキーワード: "online tuner", "online metronome", "free chromatic tuner", "chromatic tuner metronome", "チューナー オンライン", "メトロノーム オンライン", "無料 チューナー"

広告配置案: メトロノームパネル下 or パネル間。代替としてBuySellAds / Carbon Ads。

---

### Phase 2: バッキングトラック＋スケールガイド＋アプリ版立ち上げ

ギタリストの即興練習を支援する機能追加と、Expoアプリの初期構築を同時進行。

**2a. 機能追加（Web版 → アプリ版に共通）**

| 機能 | 概要 | 技術 |
|------|------|------|
| キー選択 | Key=Am, Key=C等を選択するUI | JSの状態管理 |
| コード進行生成 | キーに応じた定番コード進行を自動生成。I-IV-V-I等 | 音楽理論をTS/JSで実装 |
| コード進行表示 | 現在のコードをリアルタイムでハイライト表示 | DOM/RN更新 |
| バッキング再生 | コードトーンを合成再生 | Web Audio API（Web）/ expo-av（アプリ） |
| スケールマップ | 選択したキーのスケール音を可視化。弾いてる音がスケール上のどこかをリアルタイム表示 | ピッチ検出＋スケールマッチング |
| スケール外音の警告 | スケール外の音を弾いたら色で警告 | cents計算の応用 |

**2b. アプリ版の基盤構築**

| タスク | 詳細 |
|--------|------|
| Expoプロジェクト初期化 | `npx create-expo-app tunebeat-app` |
| Supabaseプロジェクト作成 | Google OAuth設定、user_settingsテーブル作成 |
| Google Sign-In実装 | Supabase Auth + expo-auth-session |
| Phase 1機能のアプリ移植 | チューナー、メトロノーム、BPM検知 |
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
- ギター指板上のスケール可視化（SVGフレットボード）
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
