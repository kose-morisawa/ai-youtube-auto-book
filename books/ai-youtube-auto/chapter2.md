---
title: "システム全体像──10種APIが連携する自動化パイプライン"
---

## 全体アーキテクチャ

このシステムは5つのレイヤーで構成されています。

```
┌─────────────────────────────────────────────────────────┐
│  レイヤー1：コンテンツ戦略                                │
│  NewsAPI・Wikipedia・yfinance でトレンドを自動収集       │
└───────────────────────┬─────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  レイヤー2：台本・音声                                    │
│  AI（Claude等）で台本生成 → VOICEVOX で音声合成          │
└───────────────────────┬─────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  レイヤー3：映像素材                                      │
│  Pexels・Unsplash・Pixabay・Giphy で画像・動画・GIF取得  │
│  Freesound で効果音取得                                  │
└───────────────────────┬─────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  レイヤー4：動画レンダリング                              │
│  Remotion（ReactベースのCode動画）でMP4生成              │
└───────────────────────┬─────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  レイヤー5：配信                                          │
│  YouTube Data API v3 で自動アップロード                  │
└─────────────────────────────────────────────────────────┘
```

## 使用する技術・API一覧

| カテゴリ | ツール/API | 役割 | 費用 |
|---------|-----------|------|------|
| **動画レンダリング** | Remotion | ReactコードをMP4に変換 | 無料（OSS） |
| **音声合成** | VOICEVOX | テキスト → 声優風音声 | 無料（ローカル） |
| **配信** | YouTube Data API v3 | 自動アップロード・タグ設定 | 無料（制限あり） |
| **写真素材** | Pexels | CC0写真・動画 | 無料 |
| **写真素材** | Unsplash | 高品質写真 | 無料（制限あり） |
| **写真・イラスト** | Pixabay | 写真・イラスト・動画 | 無料 |
| **GIF** | Giphy | GIFアニメ | 無料（制限あり） |
| **効果音** | Freesound | CC0効果音 | 無料 |
| **株価データ** | yfinance | 日米株価・決算 | 無料 |
| **株価データ** | Alpha Vantage | リアルタイム株価 | 無料（25回/日） |
| **ニュース** | NewsAPI | 最新ニュース検索 | 無料（100回/日） |
| **百科事典** | Wikipedia API | 用語解説・背景情報 | 無料・無制限 |

**合計コスト：ほぼ無料**（Alpha Vantage Premiumにアップグレードすれば$50/月で無制限）

## ディレクトリ構成

```
AI_COMPANY/
├── .env.example              # APIキーテンプレート
├── requirements.txt          # Python依存
├── package.json              # Node.js依存
│
├── docs/
│   ├── API_SETUP.md          # 各API登録手順
│   └── FOR_BUYERS.md         # 購入者向けクイックスタート
│
├── tools/
│   ├── generate-audio.mjs    # VOICEVOX音声生成（メインツール）
│   ├── fetch-illust.mjs      # イラスト取得
│   ├── test-apis.mjs         # 全API統合テスト
│   └── api/
│       ├── _config.mjs       # 共通設定（キャッシュ・リトライ）
│       ├── youtube.mjs       # YouTube API（OAuth2対応）
│       ├── pexels.mjs        # Pexels
│       ├── unsplash.mjs      # Unsplash
│       ├── pixabay.mjs       # Pixabay
│       ├── giphy.mjs         # Giphy
│       ├── freesound.mjs     # Freesound（OAuth2対応）
│       └── finance.py        # yfinance・Alpha Vantage・NewsAPI・Wikipedia
│
└── AIエージェント/            # Remotionプロジェクト
    ├── src/
    │   ├── DialogueScene.tsx  # ★ 台本データ・演出ロジック
    │   ├── characterAssets.ts # キャラ×感情 → 画像マッピング
    │   └── Root.tsx           # 動画コンポジション登録
    └── public/
        ├── audio/             # 生成音声（.wav）
        └── {キャラ}-*.png     # キャラクター画像
```

## データの流れ：1本の動画ができるまで

### 1. テーマ決定（人間）

```
「今週のトレンドは？」
    ↓
python tools/api/finance.py news "NISA 投資" 10
    → 最新10件のニュースを取得
    → ネタの種を発見
```

### 2. 台本生成（AI）

AIに台本を作らせます。本書付属のシステムでは、
キャラクターの掛け合い台本を自動生成するプロンプトが含まれています。

```javascript
// 台本データの例（DialogueScene.tsx 内）
const SCRIPT_LINES = [
  { id: 'scene1_01', char: 'neko', emotion: 'explaining',
    text: 'NISAって知ってる？', speakText: 'ニーサって知ってる？' },
  { id: 'scene1_02', char: 'kenta', emotion: 'neutral',
    text: '聞いたことはあるけど……', speakText: '聞いたことはあるけど……' },
];
```

### 3. 音声合成（自動）

```bash
node tools/generate-audio.mjs
# SCRIPT_LINES の全セリフを VOICEVOX で音声変換
# public/audio/*.wav に保存
# audioDurations.ts（フレーム計算用）を自動更新
```

### 4. 動画レンダリング（自動）

```bash
cd AIエージェント
npx remotion render DialogueScene-001
# out/DialogueScene-001.mp4 が生成される
```

### 5. アップロード（自動）

```bash
node tools/api/youtube.mjs upload \
  --video AIエージェント/out/DialogueScene-001.mp4 \
  --title "タイトル" \
  --privacy private
```

## キャッシュ設計：APIコストを最小化

すべてのAPIは **同日中は同一クエリをキャッシュ** します。

```
初回リクエスト → API呼び出し → tools/api/.cache/に保存
2回目以降     → キャッシュから返す（APIコール消費なし）
翌日          → キャッシュ失効 → 再取得
```

これにより、Alpha Vantageの「25回/日」制限のような厳しい制限でも
実用的に使い続けられます。

## リトライ設計：安定稼働のために

ネットワークエラーやAPI一時障害に自動対応します。

```
1回目失敗 → 1秒後にリトライ
2回目失敗 → 2秒後にリトライ
3回目失敗 → 4秒後にリトライ
4回目失敗 → エラーを投げる（諦める）
```

指数バックオフ（Exponential Backoff）と呼ばれる標準的な方式です。

---

*次の章では、環境構築とAPIキーの取得手順を解説します。*
