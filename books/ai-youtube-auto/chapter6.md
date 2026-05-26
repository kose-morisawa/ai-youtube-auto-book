---
title: "カスタマイズ──自分のチャンネルに合わせた改造ガイド"
---

## このシステムは「改造前提」で設計されています

本書のシステムは特定のチャンネル向けに作られていますが、
以下のカスタマイズで**あなた独自のチャンネル向けに作り直せます。**

---

## カスタマイズ 1：キャラクターを変える

すべてのキャラクター設定は `channel.config.json` に集約されています。
このファイルだけ編集すれば、コードを一切触らずにキャラを変更できます。

### 画像の差し替え

1. キャラクター画像を `AIエージェント/public/` に置く
2. `channel.config.json` の `emotions` にファイル名を登録する

```json
{
  "characters": {
    "char_a": {
      "label": "あなたのキャラ名",
      "emotions": {
        "neutral":    "mychar-neutral.png",
        "explaining": "mychar-explaining.png",
        "smug":       "mychar-smug.png"
      },
      "fallback": "mychar-neutral.png"
    }
  }
}
```

:::message
画像は必ず `AIエージェント/public/` 直下に置いてください。
サブフォルダに入れると表示されません。
:::

### VOICEVOXの声を変える

`channel.config.json` の `voicevoxId` を変更するだけです。

```json
{
  "characters": {
    "char_a": { "voicevoxId": 3  },
    "char_b": { "voicevoxId": 12 }
  }
}
```

VOICEVOXで使える話者IDを確認するには：

```bash
curl http://localhost:50021/speakers | python -m json.tool | grep -E '"name"|"id"'
```

好きな話者のIDを `voicevoxId` に設定してください。

---

## カスタマイズ 2：コンテンツジャンルを変える

本書はマネー系ですが、他のジャンルにも応用できます。

### ジャンル別の変更点

| ジャンル | 変更が必要なAPI | ネタ収集コマンド例 |
|---------|--------------|-----------------|
| テック解説 | NewsAPI | `news "ChatGPT 新機能"` |
| 健康・医療 | NewsAPI + Wikipedia | `news "睡眠 最新研究"` |
| ビジネス | NewsAPI + Alpha Vantage | `news "スタートアップ 資金調達"` |
| 歴史解説 | Wikipedia | `wiki "江戸時代 経済"` |

### データ取得を新APIに差し替える

`tools/api/_config.mjs` の `withRetry` と `readCache/writeCache` を使えば、
任意のAPIクライアントを同じパターンで実装できます。

```javascript
// 新APIクライアントのテンプレート
import { readCache, writeCache, withRetry } from './_config.mjs';

export async function fetchData(query) {
  const cached = readCache(`my_api_${query}`);
  if (cached) return cached;

  const data = await withRetry(() =>
    fetch(`https://api.example.com/search?q=${query}`)
      .then(r => r.json())
  );

  writeCache(`my_api_${query}`, data);
  return data;
}
```

---

## カスタマイズ 3：演出を変える

`AIエージェント/src/DialogueScene.tsx` の以下の変数で見た目を調整できます。

### 背景画像の変更

`channel.config.json` の `theme.background` を変更します。

```json
{
  "theme": {
    "background": "MyBackground.png"
  }
}
```

差し替えた画像を `AIエージェント/public/` に置いて、ファイル名を設定するだけです。

### キャラクターサイズの変更

`channel.config.json` の `theme.charHeight` を変更します。

```json
{
  "theme": {
    "charHeight": 1150
  }
}
```

### 字幕の色・スタイル変更

`channel.config.json` の `theme` と各キャラの `badgeColor` で変更できます。

```json
{
  "characters": {
    "char_a": { "badgeColor": "#7c3aed" },
    "char_b": { "badgeColor": "#1d4ed8" }
  },
  "theme": {
    "subtitleBorder": "#fbbf24"
  }
}
```

---

## カスタマイズ 4：投稿スケジュールを自動化する

`--privacy scheduled` オプションで予約投稿ができます。

```javascript
// youtube.mjs の uploadVideo に scheduledFor を追加（拡張例）
await youtube.videos.insert({
  requestBody: {
    status: {
      privacyStatus: 'private',
      publishAt: '2025-06-01T09:00:00+09:00',  // JST 9:00に自動公開
    }
  }
});
```

cronやWindows タスクスケジューラと組み合わせれば、
毎週自動で動画を作って予約投稿するパイプラインが完成します。

---

## よくあるカスタマイズQ&A

**Q: 英語コンテンツにしたい**

VOICEVOXは日本語専用です。英語音声を使いたい場合は
`generate-audio.mjs` を ElevenLabs API や OpenAI TTS API に差し替えてください。

**Q: 字幕なしにしたい**

`DialogueScene.tsx` の `SubtitleBar` コンポーネントの `display` を `none` にします。

**Q: BGMを追加したい**

Remotion の `<Audio>` コンポーネントで追加できます。
Freesoundから取得したCC0音声をBGMとして使えます。

```tsx
import { Audio, staticFile } from 'remotion';

<Audio src={staticFile('audio/bgm.mp3')} volume={0.1} />
```

**Q: 動画を短くしたい（ショート動画対応）**

Remotion の Composition の `width` と `height` を変更します。

```tsx
// Root.tsx
<Composition
  width={1080}
  height={1920}  // 縦長（ショート対応）
  // ...
/>
```

---

## 次のステップ

本書で紹介したシステムの応用として、以下にチャレンジしてみてください。

1. **まず1本作る** → 完成の達成感を得る
2. **10本作る** → ワークフローが身につく
3. **100本を目標にシステムを磨く** → 本当の自動化が始まる

---

## おわりに

このシステムは「コードを書くこと」が目的ではありません。
**「仕組みを作って、それが稼いでくれる状態」を目指すためのツールです。**

動画が1本アップロードされれば、それはあなたが寝ている間も再生され続けます。
100本になれば、100倍の力で働いてくれます。

FIREへの道のりは長いですが、このシステムがその加速剤になれば幸いです。

購入・フィードバックはZennのコメント欄でお待ちしています。

---

## 付録：全コマンドリファレンス

```bash
# ── 音声生成 ──────────────────────────────────
node tools/generate-audio.mjs             # 新規のみ生成
node tools/generate-audio.mjs --force     # 全て再生成

# ── 動画 ──────────────────────────────────────
cd AIエージェント
npx remotion studio                        # プレビュー
npx remotion render DialogueScene-001      # レンダリング

# ── YouTube ───────────────────────────────────
node tools/api/youtube.mjs auth            # 初回認証
node tools/api/youtube.mjs status          # 認証確認
node tools/api/youtube.mjs upload \
  --video out/video.mp4 --title "タイトル" --privacy private

# ── 素材収集 ──────────────────────────────────
# Pexels
node -e "import('./tools/api/pexels.mjs').then(m => m.searchPhotos('money', 5).then(console.log))"

# ── データ取得 ────────────────────────────────
python tools/api/finance.py stock AAPL     # 株価
python tools/api/finance.py stock 7203     # 日本株（トヨタ）
python tools/api/finance.py earnings AAPL  # 決算
python tools/api/finance.py news "NISA"    # ニュース
python tools/api/finance.py wiki "FIRE"    # Wikipedia

# ── 全API テスト ──────────────────────────────
npm run test:apis
```
