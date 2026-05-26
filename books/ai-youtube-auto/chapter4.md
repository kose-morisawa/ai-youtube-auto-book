---
title: "実践──動画1本を30分で作るワークフロー"
---

## ゴール：この章を読み終えたらMP4が1本できている状態

手順に沿って進めれば、初回でも動画を出力できます。

---

## STEP 1：ネタ・トレンドを調べる（5分）

```bash
# 投資・マネー関連のニュースを10件取得
python tools/api/finance.py news "NISA 積立 投資" 10
```

出力例：
```json
[
  {
    "title": "新NISAの積立枠、利用者が過去最多を更新",
    "source": "日経新聞",
    "published_at": "2025-05-01T08:00:00Z",
    "url": "https://..."
  },
  ...
]
```

このニュースを参考に、動画テーマを1つ決めます。

---

## STEP 2：台本データを `DialogueScene.tsx` に書く（10分）

台本は `AIエージェント/src/DialogueScene.tsx` の `SCRIPT_LINES` 配列に書きます。

### セリフの基本構造

```javascript
{
  id: 'scene1_01',          // ユニークなID（音声ファイル名になる）
  char: 'char_a',           // channel.config.json で定義したキャラID
  emotion: 'explaining',    // 感情キー（channel.config.json の emotions に定義）
  text: 'この動画で資産形成の全体像がわかります',  // 字幕に表示するテキスト
  speakText: 'この動画で資産形成の全体像がわかります',  // VOICEVOXに読ませるテキスト
}
```

:::message
`char` に指定する値は `channel.config.json` の `characters` キーと一致させてください。  
たとえば `"char_a"` と設定したなら、台本でも `char: 'char_a'` と書きます。
:::

### 感情キー一覧

感情キーは `channel.config.json` の `emotions` に定義したキーを使います。

| キャラの役割 | よく使う感情キー例 | 使い所 |
|------------|----------------|-------|
| 先生役（char_a） | `neutral` | 通常 |
| 先生役（char_a） | `explaining` | 解説中 |
| 先生役（char_a） | `smug` | ドヤ顔・決めゼリフ |
| 先生役（char_a） | `sigh` | あきれ・溜め息 |
| 生徒役（char_b） | `neutral` | 通常 |
| 生徒役（char_b） | `excited` | 驚き・テンション高め |
| 生徒役（char_b） | `shock` | ショック・落ち込み |
| 生徒役（char_b） | `smile` | 納得・嬉しい |

:::message
感情キーは自由に追加できます。`channel.config.json` に追加した後、対応する画像を `AIエージェント/public/` に置けばすぐ使えます。
:::

### speakText のコツ：VOICEVOXの誤読を防ぐ

VOICEVOXは英字・略語・数字の読み間違いがあります。
`speakText` に「読ませたい読み方」を書くことで回避できます。

```javascript
// ❌ 誤読される例
{ text: 'NISAで1000万円', speakText: 'ニーサで1000万円' }

// ✅ 正しく読ませる
{ text: 'NISAで1000万円', speakText: 'ニーサで1000万円' }
{ text: 'AIが分析', speakText: 'エーアイが分析' }
{ text: '30年後', speakText: '30ねんご' }
```

:::message
`speakText` は必須ではありません。不要な行は省略できます。
英字・数字が含まれる行だけ設定すれば十分です。
:::

### 台本の例（5セリフ）

> 以下は「金融解説チャンネル」の例です。`char_a` / `char_b` をあなたの設定に合わせて変更してください。

```javascript
const page1Sources = [
  { id: 'ep1_s1_01', char: 'char_b', emotion: 'neutral',
    text: '先生、新NISAって結局どうなんですか？',
    speakText: '先生、新ニーサって結局どうなんですか？' },

  { id: 'ep1_s1_02', char: 'char_a', emotion: 'explaining',
    text: '良い質問！まず「非課税」の意味から説明しようか',
    speakText: '良い質問！まず「非課税」の意味から説明しようか' },

  { id: 'ep1_s1_03', char: 'char_b', emotion: 'excited',
    text: '非課税って、税金がかからないってこと？',
    speakText: '非課税って、税金がかからないってこと？' },

  { id: 'ep1_s1_04', char: 'char_a', emotion: 'smug',
    text: 'そう。普通は利益の20%が税金で持っていかれる。\nNISAならそれがゼロ。',
    speakText: 'そう。普通は利益の20%が税金で持っていかれる。\nニーサならそれがゼロ。' },

  { id: 'ep1_s1_05', char: 'char_b', emotion: 'shock',
    text: '20%……！ それって結構大きいですよね',
    speakText: '20%……！ それって結構大きいですよね' },
];
```

---

## STEP 3：音声を生成する（自動・3〜5分）

VOICEVOXが起動していることを確認してから実行します。

```bash
node tools/generate-audio.mjs
```

出力例：
```
[1/5] ep1_s1_01_kenta.wav ... OK (1.23s)
[2/5] ep1_s1_02_neko.wav  ... OK (2.11s)
[3/5] ep1_s1_03_kenta.wav ... skip（既存ファイルあり）
[4/5] ep1_s1_04_neko.wav  ... OK (3.45s)
[5/5] ep1_s1_05_kenta.wav ... OK (1.89s)

audioDurations.ts を更新しました
```

:::message
一度生成した音声は2回目以降スキップされます。
台本を変えた行だけ再生成したい場合は、その `.wav` ファイルを削除してから実行してください。
:::

---

## STEP 4：Remotion Studio でプレビュー確認（5〜10分）

```bash
cd AIエージェント
npx remotion studio
```

ブラウザで `http://localhost:3001` を開き、作成したコンポジション（例：`DialogueScene-001`）を選択してプレビューを確認します。

### チェックポイント

- [ ] キャラクターの感情表現が合っているか
- [ ] 字幕のテキストが正しく表示されているか
- [ ] 音声のタイミングがズレていないか
- [ ] 章タイトル・SE（効果音）が適切に入っているか

問題があれば `DialogueScene.tsx` を修正して保存 → ブラウザが自動リロードします。

---

## STEP 5：動画をレンダリングする（自動・5〜15分）

プレビューで確認できたらレンダリングします。

```bash
npx remotion render DialogueScene-001
```

```
Rendering: DialogueScene-001
 ██████████████████░░  90% | frame 543/600 | 00:14 remaining
Rendering done.
Output: out/DialogueScene-001.mp4
```

動画の長さによって5〜15分かかります。

---

## STEP 6：YouTubeにアップロードする（自動）

```bash
cd ..  # プロジェクトルートに戻る

node tools/api/youtube.mjs upload \
  --video AIエージェント/out/DialogueScene-001.mp4 \
  --title "新NISAで20%の税金がゼロになる仕組みを5分で完全理解した" \
  --desc "新NISAの非課税メリットをAIキャラクターが分かりやすく解説。
#NISA #新NISA #投資 #資産形成" \
  --tags "NISA,新NISA,投資,資産形成,非課税,AI" \
  --privacy private
```

```
アップロード開始: AIエージェント/out/DialogueScene-001.mp4 (245.3 MB)
  [████████████████████] 100% (245.3MB / 245.3MB)
アップロード完了: https://youtu.be/xxxxxxxxx
```

**`--privacy private` で非公開アップロードしてから、YouTubeの管理画面で確認してください。**  
問題なければ公開設定を「公開」に変更します。

---

## 素材の自動収集（オプション）

サムネイルや関連画像が必要な場合、Pexelsなどから自動収集できます。

```bash
# 「money investment」で写真を検索・ダウンロード
node -e "
import('./tools/api/pexels.mjs').then(async m => {
  const photos = await m.searchPhotos('money investment', 5);
  await m.download(photos[0], 'materials/photos');
  console.log('Downloaded:', photos[0].photographer);
});
"
```

ダウンロードされた画像は `materials/photos/` に保存され、
`pexels_XXXX.json` に撮影者・ライセンス情報も自動保存されます。

---

## 所要時間のまとめ

| 作業 | 所要時間 | 自動化度 |
|------|---------|---------|
| ネタ調査 | 5分 | ▲（結果は自動、判断は人間） |
| 台本作成 | 10分 | △（AIと共同作業） |
| 音声生成 | 3〜5分 | ✅ 全自動 |
| プレビュー確認 | 5〜10分 | ▲（目視確認は必要） |
| レンダリング | 5〜15分 | ✅ 全自動（待つだけ） |
| アップロード | 1〜2分 | ✅ 全自動 |
| **合計** | **30〜47分** | |

従来の10〜21時間から、**30〜47分**へ。約25倍の効率化です。

---

*次の章では、このシステムで収益化・FIRE達成への道を説明します。*
