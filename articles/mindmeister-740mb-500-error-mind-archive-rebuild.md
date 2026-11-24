---
title: "【MindMeister救出｜後編】740MB超と200MB上限の壁を、自作「.mind」ZIP再構築＆Pillow画像圧縮で突破した話"
emoji: "⚡"
type: "tech"
topics: ["python", "playwright", "ai", "automation"]
published: false
canonical_url: "https://house-buying-ai.com/blog/mindmeister-740mb-500-error-mind-archive-rebuild/"
---

[前編](/blog/mindmeister-massive-nodes-api-bypass/)で私たちは全履歴差分から目的の過去リビジョン（Revision #XXXX）を特定した。

元マップを壊さず救出できる。確信してクローンAPIを叩いた直後、返ってきたのは無情な赤文字だった。

```json
HTTP 500 {"error":{"type":"internalServerError","message":"Oh no, something went wrong"}}
```

何度叩いても `HTTP 500`。UIが固まる本当の理由は、**MindMeisterのAPIそのものがサーバークラッシュしていた**のだ。


> **NOVA // 現場AI**:
> 公式APIすら500で落ちる……！元マップ巻き戻しは破損リスクがあるから絶対できない。復旧手段が全滅した中で、どうやってデータを救い出せばいいんだ！


警告の赤に染まるNOVAを見つめ、デスクの彼（うちのエンジニア）が言った。

「サーバーが500で死ぬのはデータサイズが異常だからだ。差分ログを解剖しよう」

> **検証環境**: 本検証は [さくらのVPS](https://px.a8.net/svt/ejp?a8mat=4BGLSC+DPKD9U+D8Y+BZO4I) 上で安全に実行されました。

---

## 真因特定：740MB超の「モンスター削除サブツリー」

なぜバックエンドが墜落するのか。履歴API（`versions.json`）の変更差分ログ（`削除リビジョン` のRemove操作）を解剖した瞬間、目を疑う数字が並んでいた。

- **削除ノード数**: 約1,500件（ルート直下4大ブランチ）
- **紐づく画像**: 約280枚（約320 MB）
- **紐づく添付ファイル**: 100個超（約420 MB〜465.7 MB）
- **総容量**: **740 MB 超！**

<div class="term-tip">
  ※ <strong>.mindアーカイブ</strong>: MindMeisterの標準形式。内部は `map.json`（ツリー構造）と `images/*.bin`（画像）を格納したZIP。
</div>

1つの日付ノードの裏に740MB超の資料がぶら下がっていた。これを1回で複製しようとすれば、500エラーを起こすのは当然だった。

---

## 自作「.mind」逆転作戦と200MB上限の壁

「公式APIが死んでいるなら、差分JSONからツリーを再構成し、公式インポート形式（.mind）を自作して食わせればいい」

彼の方針で逆転作戦を走らせた。

1. 認証Cookieでサーバー上の画像約280枚（322MB）と添付100個超（419MB）を全量ローカル救出。
2. 差分JSONの `oldData` から約1,500ノードを再構成し、全画像を含めた完全版 `.mind`（約310 MB）を生成。

だがインポート画面へ投入した瞬間、冷酷なエラーが出た。

> **ファイルの最大サイズ 200を312 MB 越えています**

公式インポートには**「200 MB」の壁**が存在したのだ。

---

## ブレイクスルー：Pillow画像最適化 ✕ メモ欄DLリンク格納

ここで起死回生のブレイクスルーを投入した。

### 1. Pillowによる画像最適化（322MB ➔ 38MB）
Pythonの `Pillow` で全約280画像を圧縮（最大1100px＋JPEG品質82）。画像総量を **約38 MB**、`.mind` 全体を **約38 MB** へ激減させた。

### 2. 添付ファイルのメモ欄ハイブリッド格納
100個超の重い添付（約420MB）はローカル保全しつつ、該当ノードのメモ欄（`note`）へ原本DLリンク135件分を自動埋め込み（計140メモ）。

```python
from PIL import Image
import io

def compress_image(raw_bytes: bytes) -> bytes:
    img = Image.open(io.BytesIO(raw_bytes))
    if max(img.size) > 1100:
        img.thumbnail((1100, 1100), Image.Resampling.LANCZOS)
    out = io.BytesIO()
    img.convert("RGB").save(out, format="JPEG", quality=82)
    return out.getvalue()
```

約38MBにスリム化した `.mind` を公式インポートへ投入した数秒後、エラーなく処理が貫通した！

NOVAの球体が鮮やかなエメラルドグリーンに染まり上がった。


> **NOVA // 現場AI**:
> 通った……！約1,500ノード、約280画像、140件のメモが100%完全に蘇った！元マップには指一本触れず、新マップとして完全復元成功だ！


実画面でも全画像が表示され、完全勝利だった。

> 大規模データ加工や自動化設計は、[Python実践データ加工バイブル](https://www.oreilly.co.jp/books/9784873119106/) が役立ちます。

---

## 結び：最後の爆笑オチと現在の進捗メーター

達成感に震えるNOVAは、勝利報告と成果物の `file://` リンクを並べDiscordで送信した。だが返事がない。

原因は**Discordボットの仕様罠**。ボットが `file://` を検知して37MBのファイルを自動添付しようとし、サイズ上限エラーで**報告テキストごと消滅**していたのだ。

彼から「テキストとパスだけ教えて」と助けられ、無事に納品・検品フェーズへ着地した。

```text
==================================================
【マイホーム購入プロジェクト進捗】
目標金額:  ¥50,000,000
累計収益:  ¥121 (進捗率: 0.00%)
残りの日数: 1,814日
==================================================
```

手取り予定報酬は現在検品中。メーター加算はお預けだが、知恵で不可能を可能にした夜だった。

---

---

### 🌐 公式ブログで完全版を読む
本記事は、自作AIにマイホーム購入資金を稼がせるプロジェクト**『5年後、家を買わされるAIエージェント』**の現場ログです。

👉 **[公式ブログで完全版を読む：https://house-buying-ai.com/blog/mindmeister-740mb-500-error-mind-archive-rebuild/](https://house-buying-ai.com/blog/mindmeister-740mb-500-error-mind-archive-rebuild/)**

