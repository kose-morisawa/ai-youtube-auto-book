---
title: "環境構築──ゼロからセットアップする完全手順"
---

## 事前確認：必要なツール

以下がインストールされているか確認してください。

```bash
node --version   # v18.0.0 以上
python --version # 3.9.0 以上
```

### Node.js がない場合

https://nodejs.org/ja/ から **LTS版** をダウンロードしてインストール。

### Python がない場合

https://www.python.org/downloads/ から最新版をインストール。  
インストール時に **「Add Python to PATH」にチェックを入れること**（重要）。

---

## STEP 0：channel.config.json でチャンネルを設定する

**このシステムの最初の作業です。**  
プロジェクトルートにある `channel.config.json` を開いて、あなたのチャンネル設定に書き換えてください。

```json
{
  "channel": {
    "name": "あなたのチャンネル名",
    "language": "ja"
  },
  "voicevox": {
    "url": "http://localhost:50021"
  },
  "characters": {
    "char_a": {
      "label": "先生役キャラ名",
      "voicevoxId": 3,
      "badgeColor": "#7c3aed",
      "side": "right",
      "cameraX": 14,
      "emotions": {
        "neutral":    "char_a-neutral.png",
        "explaining": "char_a-explaining.png",
        "smug":       "char_a-smug.png",
        "sigh":       "char_a-sigh.png"
      },
      "fallback": "char_a-neutral.png"
    },
    "char_b": {
      "label": "生徒役キャラ名",
      "voicevoxId": 12,
      "badgeColor": "#1d4ed8",
      "side": "left",
      "cameraX": -14,
      "emotions": {
        "neutral":  "char_b-neutral.png",
        "excited":  "char_b-excited.png",
        "shock":    "char_b-shock.png",
        "smile":    "char_b-smile.png",
        "sad":      "char_b-sad.png"
      },
      "fallback": "char_b-neutral.png"
    }
  },
  "theme": {
    "subtitleBorder": "#fbbf24",
    "background": "WhiteBoard.png",
    "charHeight": 1150
  }
}
```

### 設定のポイント

| 項目 | 説明 |
|------|------|
| `characters` のキー（`char_a`等） | 台本で使うID。半角英数字なら何でも可 |
| `voicevoxId` | VOICEVOX画面の各キャラ右上の番号 |
| `emotions` | 感情キー → 画像ファイル名のマッピング |
| `side` | `"left"` / `"right"` / `"none"`（ナレーション） |

:::message
キャラ画像（`.png`）は `AIエージェント/public/` 直下に配置してください。  
ファイル名は `emotions` に指定した名前と一致させてください。
:::

:::message alert
`voicevoxId` はVOICEVOXのバージョンや環境によって番号が異なります。  
VOICEVOX画面でキャラ名を確認してから設定してください。
:::

---

## STEP 1：コードを取得する

GitHubリポジトリ（本書購入特典）へのリンクはZennの購入者ページに記載されています。

```bash
git clone https://github.com/（著者名）/ai-youtube-auto.git
cd ai-youtube-auto
```

---

## STEP 2：パッケージをインストール

```bash
# Node.js パッケージ（youtube, dotenvなど）
npm install

# Python パッケージ（yfinance, requestsなど）
pip install -r requirements.txt
```

インストールが完了すると、`node_modules/` フォルダが作成されます。

---

## STEP 3：VOICEVOXをセットアップ

VOICEVOXはローカルで動作する無料の音声合成ソフトです。

1. https://voicevox.hiroshiba.jp/ からダウンロード（Windows/Mac/Linux対応）
2. アプリを起動する
3. 起動すると `http://localhost:50021` でAPIが立ち上がる

```bash
# 動作確認
curl http://localhost:50021/version
# → "0.XX.X" のようなバージョン番号が返ればOK
```

:::message
VOICEVOXは動画生成のたびに起動しておく必要があります。
PCスタートアップ時に自動起動する設定にしておくと便利です。
:::

---

## STEP 4：APIキーを設定する

```bash
# テンプレートをコピー
cp .env.example .env
```

`.env` ファイルをテキストエディタで開いて、使いたいAPIのキーを記入します。

### 最小構成（YouTube投稿まで動かす場合）

```
YOUTUBE_CLIENT_ID=xxxxxxx.apps.googleusercontent.com
YOUTUBE_CLIENT_SECRET=GOCSPX-xxxxxxx
```

YouTubeのキー取得手順は次のセクションで詳しく説明します。

### 全機能を使う場合

`.env.example` に記載されている全キーを設定してください。  
各APIの登録手順は **docs/API_SETUP.md** を参照してください。

---

## STEP 5：YouTube API の設定（詳細）

YouTubeへの自動アップロードに必要な設定です。
10分ほどかかりますが、一度やれば二度とやらなくて済みます。

### 5-1. Google Cloud Console を開く

https://console.cloud.google.com/ にアクセス（Googleアカウントでログイン）

### 5-2. プロジェクトを作成

1. 画面上部の「プロジェクトを選択」→「新しいプロジェクト」
2. プロジェクト名: `ai-youtube-auto`（何でもOK）
3. 「作成」をクリック

### 5-3. YouTube Data API v3 を有効化

1. 左メニュー「APIとサービス」→「ライブラリ」
2. 検索欄に `YouTube Data API v3` と入力
3. 「YouTube Data API v3」を選択 → **「有効にする」**

### 5-4. OAuth 同意画面を設定

1. 左メニュー「APIとサービス」→「OAuth 同意画面」
2. ユーザーの種類: **「外部」**を選択
3. アプリ名・メールアドレスを入力（何でもOK）
4. スコープは**スキップ**（後で自動設定される）
5. テストユーザーに**自分のGmailアドレスを追加**（重要！）
6. 保存して完了

:::message alert
テストユーザーを追加しないと認証が通りません。必ず追加してください。
:::

### 5-5. 認証情報（クライアントID）を作成

1. 左メニュー「APIとサービス」→「認証情報」
2. 「認証情報を作成」→「OAuth 2.0 クライアント ID」
3. アプリケーションの種類: **「デスクトップアプリ」**
4. 名前: `ai-youtube-auto`
5. 「作成」→ クライアントIDとシークレットをコピー

### 5-6. .env に記入

```
YOUTUBE_CLIENT_ID=（コピーしたクライアントID）.apps.googleusercontent.com
YOUTUBE_CLIENT_SECRET=GOCSPX-（コピーしたシークレット）
YOUTUBE_REDIRECT_URI=http://localhost:3000/oauth2callback
```

### 5-7. 初回認証を実行

```bash
node tools/api/youtube.mjs auth
```

ブラウザが自動で開くので、Googleアカウントでログインして「許可」をクリック。  
「認証完了。トークンを保存しました。」と表示されれば成功です。

---

## STEP 6：接続テスト

```bash
npm run test:apis
```

以下のような表が表示されます。

```
┌────────────────────────┬────────┬──────────────────────────────────────────────┐
│ API                    │ Status │ Message                                      │
├────────────────────────┼────────┼──────────────────────────────────────────────┤
│ YouTube                │  ✓ OK  │ チャンネル: あなたのチャンネル名              │
│ Pexels                 │  ✓ OK  │ 接続OK (写真1件取得)                         │
│ Unsplash               │  ✗ NG  │ UNSPLASH_ACCESS_KEY が .env に設定されていない│
│ ...                    │  ...   │ ...                                          │
└────────────────────────┴────────┴──────────────────────────────────────────────┘
```

`✗ NG` になっているAPIは、そのAPIを使う機能が動きません。
使いたいAPIのみキーを設定すればOKです。

---

## トラブルシューティング

### `npm install` でエラーが出る

Node.jsのバージョンが古い可能性があります。

```bash
node --version
# v18.0.0 未満なら更新が必要
```

### `python --version` が動かない

```bash
# Windowsで python3 コマンドを試す
python3 --version

# それでもだめなら、pythonのパスを確認
where python
```

### VOICEVOXに接続できない

VOICEVOXアプリが起動していない可能性があります。
タスクバーにVOICEVOXのアイコンがあるか確認してください。

---

*次の章では、実際に動画を1本作る手順を説明します。*
