---
title: "AIエージェントに領収書を読ませる — OCR×インボイス照合APIをx402で公開した"
emoji: "🧾"
type: "tech"
topics: ["x402", "ai", "azure", "cloudflare", "個人開発"]
published: true
---

前回、AIエージェント向けに日本の法人データを従量課金で売るAPI（[Kaisha API](https://kaisha-api.hp-vladic.workers.dev)）を公開した個人開発者です。今回はその第2弾として、**日本の請求書・領収書をOCRして、インボイス登録番号の実在照合までワンコールで返すAPI**を作りました。構想から本番稼働まで1日です。

https://denpyo-api.hp-vladic.workers.dev

## なぜ「領収書リーダー」か — 市場を先に測った

作る前に、x402エコシステム（Coinbase主導のHTTP 402決済。現在はLinux Foundation傘下）に何が出品されていて何が売れているかを、公開データでざっと実測しました。分かったことは3つ：

1. **RAG部品（embedding・チャンカー）は大量にあるが、ほぼ誰も買っていない**。エージェントは自前でできることにお金を払わない
2. **OCRも既に40件以上あるが、どれも汎用の薄いラッパーで、目立った売上がない**
3. 一方で、**縦特化のドキュメント処理**（例：EUの電子インボイス読取り）には買い手がちゃんと分散して付いている

つまり「OCRを売る」のではなく、**「この書類、経理処理して大丈夫？」という業務の答えを売る**のが筋。日本にはインボイス制度という格好の題材があります。適格請求書の登録番号（T+13桁）が実在するかは、LLMには原理的に判定できない決定的検証で、前作Kaishaで国税庁の公表データ505万件を既にD1に持っていました。これを繋げます。

## 作ったもの

- `POST /read/invoice`（$0.04）— 請求書PDF/画像 → 発行者・宛名・品目・税率別内訳のJSON ＋ **登録番号を国税庁データと照合**（実在/取消/未登録、登録名との名寄せ一致、法人登記情報まで）
- `POST /read/receipt`（$0.03）— 領収書・レシートの写真 → 店名・日時・品目・8%/10%内訳 ＋ 同じ照合。**仕入税額控除に使えるレシートかが1コールで分かる**
- `GET /verify`（$0.01）— 番号だけの照合（OCRなし）。支払い前の取引先チェック用

OCRエンジンはAzure AI Document Intelligenceのprebuiltモデル。日本語の請求書・レシートで試したところ、品目・税率・金額はほぼ完璧に取れます。ひとつ面白いのは、**登録番号はVendorTaxIdフィールドには入ってこない**こと。ただ生テキストには正確に写っているので、`T\d{13}`の正規表現で拾うのが確実でした。

決済はx402 v2（USDC・Base/Solana両対応）。解析に失敗した場合は4xxを返しますが、**x402のミドルウェアは4xx応答時に決済をキャンセルするので、失敗は課金されません**。ドキュメントに「Failed analyses are not charged」と書けるのは、この仕組みのおかげです。

## ハマりどころ（ここが本題）

### CDPのpaymentPayloadスキーマは「description 500字」上限

今回最大の地雷です。デプロイ後、GETの照合エンドポイントは決済が通るのに、**POSTの読み取り系だけ全支払いが402で拒否される**。facilitator（Coinbase CDP）のエラーは：

```
'paymentPayload' is invalid: must match one of [x402V2PaymentPayload, x402V1PaymentPayload]
```

これだけ。どのフィールドが悪いのか一切出ません。

支払いクライアントが送る`payment-signature`ヘッダーをフックで捕捉し、CDPのverifyエンドポイントを直接叩いてペイロードを1箇所ずつ変異させて調べた結果——**ルート説明文（resource.description）が500字を超えるとスキーマ全体が不一致になる**ことが判明。実測で500字=OK、501字=拒否。GETだけ通っていたのは説明文が438字だったからでした。

エージェントに発見されたい一心で説明文を盛ると、**そのルートは1円も売れなくなります**。Bazaar向けの説明は500字以内に。

### POSTルートのdiscovery宣言は`bodyType: 'json'`が必須

`@x402/extensions`の`declareDiscoveryExtension`はGET/POST共用ですが、POSTで`bodyType`を省略すると**GET型（queryParams）の宣言として整形されてしまい**、実際のルートと矛盾した宣言が402チャレンジに埋め込まれます。型定義上は`DeclareBodyDiscoveryExtensionConfig`の判別子なので、TypeScriptでも見落としやすい。POSTなら`bodyType: 'json'`を必ず書く。

### デバッグの型：ペイロードを捕捉して、facilitatorを直接叩く

上の2つを特定できたのは、この手順のおかげでした。

1. `wrapFetchWithPayment(capturingFetch, client)`に自前のfetchを渡し、`payment-signature`ヘッダー（Base64エンコードされたJSON）を捕捉
2. `HTTPFacilitatorClient`にCDPキーを渡して`verify(payload, payload.accepted)`を**ローカルから直接**呼ぶ
3. ペイロードのフィールドを1つずつ変異させて二分探索

サーバーのログにも支払いクライアントのエラーにも真因が出ないケースでは、facilitatorとの間に自分で入るのが一番早いです。

### 「前作の地雷」は踏まなかった

SolanaのATA事前作成、D1の10万バイト上限、`.well-known`より`/openapi.json`優先——前回の記事に書いた地雷は、テンプレ化しておいたおかげで今回ゼロ工数でした。2作目は本当に速い。OCR部分を含めても、市場調査からE2E決済確認まで1日です。

## タイミングの話

インボイス制度の経過措置（免税事業者からの仕入れの80%控除）は**2026年9月末で50%に下がります**。取引先が登録事業者かどうかの確認は、この10月から金銭的インパクトが一段上がる。経理まわりのエージェント自動化を作っている方には、ちょうどいい部品になるはずです。

## エージェントから使う方法

MCP対応クライアント（Claude Code等）なら1行：

```bash
claude mcp add denpyo -e EVM_PRIVATE_KEY=0x<USDC入りBase財布> -- npx -y denpyo-mcp
```

Solana派は`SVM_PRIVATE_KEY`（base58）でもOK。MCPサーバーはローカルで動くので、**手元のPDFやレシート写真のパスをそのまま渡せます**。「このフォルダの領収書、全部経費リストにして」が動きます。

ソース：https://github.com/vladic-corp/denpyo-api ／ 法人データ側の前作：[Kaisha API](https://kaisha-api.hp-vladic.workers.dev)

質問・感想歓迎です。「エージェントに読ませたい日本の書類」のアイデアも募集しています。
