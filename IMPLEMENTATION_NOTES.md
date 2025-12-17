# 実装上の注意点

Chrome Built-in AI (Prompt API) を使用する際の実装上の注意点をまとめています。

## API アクセス方法

API へのアクセス方法は複数あります。

```javascript
// 推奨: 両方の形式に対応
const api = self.ai?.languageModel ||
  (typeof LanguageModel !== "undefined" ? LanguageModel : null);
```

- `self.ai.languageModel` - 新しい形式
- `LanguageModel` - グローバルオブジェクト形式

## モデルの可用性チェック

`availability()` メソッドを使用します（`capabilities()` ではない）。

```javascript
const availability = await api.availability({
  expectedInputs: [
    { type: "text", languages: ["ja"] },
    { type: "image" }
  ],
  expectedOutputs: [
    { type: "text", languages: ["ja"] }
  ]
});

// 戻り値: "available" | "downloadable" | "unavailable"
```

## 言語指定は必須

出力言語を指定しないと以下の警告が出ます。

```
No output language was specified in a LanguageModel API request.
```

`create()` 時に `expectedInputs` と `expectedOutputs` で言語を指定してください。

```javascript
const session = await api.create({
  systemPrompt: "...",
  expectedInputs: [
    { type: "text", languages: ["ja"] },
    { type: "image" }
  ],
  expectedOutputs: [
    { type: "text", languages: ["ja"] }
  ]
});
```

### 対応言語（2025年12月時点）

- `en` - 英語
- `ja` - 日本語
- `es` - スペイン語

## promptStreaming の引数形式

メッセージは **配列** でラップする必要があります。

```javascript
// 正しい形式
const stream = session.promptStreaming([{
  role: "user",
  content: [
    { type: "text", value: "質問文" },
    { type: "image", value: fileObject }
  ]
}]);

// 間違い（配列でラップしていない）
const stream = session.promptStreaming({
  role: "user",
  content: [...]
});
```

## ストリーミングのチャンク処理

チャンクは **差分** で返されるため、累積して表示する必要があります。

```javascript
let fullText = "";
for await (const chunk of stream) {
  fullText += chunk;  // 累積
  result.textContent = fullText;
}
```

## 画像の渡し方

画像は `ImageBitmapSource` 型で渡します。

- `Blob`
- `File`（`<input type="file">` から取得）
- `ImageData`
- `ImageBitmap`
- `VideoFrame`
- `HTMLImageElement`
- `HTMLCanvasElement`
- `HTMLVideoElement`

```javascript
// File オブジェクトをそのまま渡せる
const file = document.querySelector("input[type=file]").files[0];
{ type: "image", value: file }

// Blob でも OK
const blob = await (await fetch("/image.jpg")).blob();
{ type: "image", value: blob }

// Canvas でも OK
const canvas = document.querySelector("canvas");
{ type: "image", value: canvas }
```

## エラーハンドリング

よくあるエラーと対処法についてまとめます。

| エラー | 原因 | 対処法 |
|--------|------|--------|
| `LanguageModel is not defined` | API 未対応 | Chrome 131以降を使用、フラグを有効化 |
| `availability is not a function` | API形式の違い | 両方の形式に対応するコードを使用 |
| `not of type 'LanguageModelMessage'` | 引数形式の誤り | 配列でラップする |
| `No output language was specified` | 言語未指定 | expectedOutputs で言語を指定 |

## 参考リンク

- [Prompt API 公式ドキュメント](https://developer.chrome.com/docs/ai/prompt-api)
- [GitHub Prompt API Explainer](https://github.com/webmachinelearning/prompt-api)
