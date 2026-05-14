# MatchingAspirationProAI

吸引カテーテル参照・同軸判定の音声ガイドツール  
（脳血管内治療従事者向け学習・参照目的 / **医療機器ではありません**）

バージョン: **v2.7 (Barge-in Edition)**

---

## 主な特徴

- 🎙 **完全ハンズフリー音声操作**
  ログイン後、AIが自動でガイドを開始し、選択モード／同軸モードを音声で案内します。
- 🔁 **Barge-in（割り込み）機能（v2.7 新搭載）**
  AIが読み上げている最中でも、術者が **「確認」** と声をかければ即座に読み上げを停止し、次の指示を受け付けます。読み上げ中も常にマイクが開いているため、操作が止まりません。
- 🧠 **誤認識耐性の強い辞書マッチング**
  デバイス名 / Yコネ名 / 血管径 の同音異義語・揺れに対応（Dice 係数 + 数字シグネチャ）。
- 📱 **PWA 対応**
  ホーム画面に追加するとフルスクリーン起動。アイコンは「MatchingAspirationProAI」専用デザイン。
- ☁️ **Netlify / Cloudflare Pages 両対応**
  ルート構成 + SPA リダイレクトを両方に同梱。

---

## 機能の流れ

1. **ログイン後、AIが自動起動**
   「選択確認ですか？ 同軸確認ですか？」と問いかけます。
2. **「選択」と答えた場合**
   血管径を聞き取り → 候補カテーテルを画面表示しつつ「候補1は…、候補2は…」と読み上げ。
3. **「同軸」と答えた場合**
   - 1本目のカテーテルを聞き取り → 自動選択
   - 2本目候補を提示・読み上げ → 選択
   - Yコネの使用有無 → Yコネ名 → 自動反映
   - 先端突出長を音声と画面で確認
4. **Barge-in（割り込み）**
   AIが話している間でも以下のいずれかを発話すると、即座に割り込み受付に切り替わります。
   - 「確認」「カクニン」「確認お願いします」
   - 「次」「ストップ」「キャンセル」「聞いて」「割り込み」「修正」
   - 明確な次指示（例:「血管径 2.1 ミリ」「SALVA71」「候補2」「富士」など）

---

## ファイル構成

```
.
├── index.html              # 本体（単一ファイルアプリ）
├── 404.html                # SPA フォールバック（index と同一内容）
├── manifest.json           # PWA マニフェスト（MatchingAspirationProAI）
├── netlify.toml            # Netlify 設定
├── _redirects              # Cloudflare Pages / Netlify 共通リダイレクト
├── _headers                # Cloudflare Pages / Netlify 共通ヘッダ
├── icons/
│   ├── icon-192.png        # PWA 192px
│   ├── icon-512.png        # PWA 512px
│   ├── apple-touch-icon.png# iOS ホーム画面用 180px
│   ├── icon-master.png     # 1024px マスター
│   └── icon.svg            # 代替 SVG
└── README.md               # このファイル
```

---

## Netlify へのデプロイ

### GitHub 連携（推奨）

1. GitHub リポジトリにこの ZIP の中身を **ルート直下** に配置してプッシュ。
2. Netlify ダッシュボードで **Add new site → Import an existing project**。
3. ビルド設定:
   - **Build command**: 空欄
   - **Publish directory**: `.`
4. Deploy を実行。`netlify.toml` が自動で読み込まれます。

### 手動デプロイ

ZIP を展開した中身（`index.html` が見える状態）を Netlify の **Deploys → drag & drop** に投入してください。

---

## Cloudflare Pages へのデプロイ

### GitHub 連携（推奨）

1. Cloudflare ダッシュボード → **Pages → Create application → Connect to Git**。
2. リポジトリを選択し、以下の設定で作成:
   - **Framework preset**: `None`
   - **Build command**: 空欄（不要）
   - **Build output directory**: `/`（または空欄）
3. Deploy を実行。`_redirects` と `_headers` が自動で適用されます。

### 直接アップロード

1. **Pages → Create application → Direct Upload**。
2. ZIP を展開した中身一式をドラッグ&ドロップ。

---

## ローカル動作確認

```bash
# Python 標準サーバで簡易確認
python3 -m http.server 8080
# → http://127.0.0.1:8080/ を開いて確認
```

---

## ログイン

初期パスワード: `dvx`
（クライアント検証用。`index.html` 内 `doLogin()` で変更可能）

---

## ブラウザ要件

| ブラウザ | 音声入力 | 音声出力 | 備考 |
| --- | --- | --- | --- |
| Safari (iOS 16+) | ✅ | ✅ | ホーム画面追加で全画面表示 |
| Chrome (Android / Desktop) | ✅ | ✅ | 推奨 |
| Edge (Desktop) | ✅ | ✅ | 動作確認済 |
| Firefox | ⚠ | ✅ | Web Speech API のサポートが限定的 |

マイク権限は **「許可」** に設定してください。  
HTTPS 必須（Netlify / Cloudflare Pages は標準で HTTPS 化されます）。

---

## Barge-in の挙動詳細（技術ノート）

- `aiState.bargeIn` が `true` の間、`SpeechRecognition` は常時 `continuous=true` で稼働。
- `speak()` 開始と同時にマイクをオープン。
- `rec.onresult` 内で interim transcript を監視し、`INTERRUPT_WORDS` か `looksLikeNextCommand()` 該当時に `speechSynthesis.cancel()` を即時実行。
- 「確認」のみの発話の場合は、最後の問いかけを再提示せず「はい、お話しください。」とだけ返して次指示を待ちます。
- `setInterval(4s)` でマイク稼働を監視し、止まっていたら再起動。
- Barge-in は右下パネルの **🟢 Barge-in: ON** ボタンで OFF にも切り替え可能。

---

## 注意

本ツールは **薬機法上の医療機器ではありません**。  
脳血管内治療に従事する医療従事者の **参照・学習目的** で提供しています。  
最終的な機器選択判断は、必ず術者が最新の IFU（添付文書）を参照のうえ行ってください。

---

## バージョン履歴

- **v2.7 (current)** — Barge-in（割り込み）機能、Cloudflare Pages 対応、README 同梱。
- v2.6 — ProAI ブランディング、ホーム画面アイコン刷新、認識辞書強化。
- v2.5 — 音声ガイド初期実装、Netlify 配布。
