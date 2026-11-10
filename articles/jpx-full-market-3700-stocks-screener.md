---
title: "ランキング200銘柄では調整相場で全滅する。JPX東証全3,693銘柄をマルチスレッド走査する「25SMA野良株スクリーニング」を作った話"
emoji: "⚡"
type: "tech"
topics: ["python", "pandas", "yfinance", "trading", "automation"]
published: true
canonical_url: "https://house-buying-ai.com/blog/jpx-full-market-3700-stocks-screener/"
---

朝8時30分。  
東証の寄り付き30分前に自動起動した株取引AI **VEGA（ベガ）** は、その日の買付候補銘柄を洗い出すスクリーニングを実行していた。

しかし数分後、Discordの `#trading-bot` チャンネルに飛んできた通知を見て、開発者（彼）が声を上げた。  
「え、今日買える銘柄が1個も見つからないってどういうこと？？？」

ログを覗き込んだVEGAの端末が、悔しそうに青白いパルスを揺らした。


> **VEGA // 株取引AI**:
> ランキングから取得した上位候補をGemini Visionで目視検閲したんだが、オロもミダックHDも全部『下降トレンド中の上ヒゲ戻り売り』で却下（REJECT）になった……。母集団200件の中には、審査を通過できる本物の初動株が1つも残っていない！


それまでのスクリーニングは、Webの「低かい離率ランキング」の上位約200銘柄だけを母集団にしていた。  
相場全体が好調な日はそれでも良かった。だが、地合いが少し悪化すると、ランキング上位200件は「急落して落ちてきたナイフ」ばかりで埋め尽くされ、AIの厳格な視覚検閲を通過できる銘柄がゼロ（全滅）になってしまったのだ。

「ランキングの上澄みだけを見ていたら、東証の片隅で静かに大口資金が集まっている『本物の野良株』を取り逃がす」

今回は、私たちが母集団を約20倍に拡大し、**JPX（日本取引所グループ）公式マスターから東証全上場3,693銘柄を毎日全件スキャンしてNo.1期待値株をあぶり出す「5段階スクリーニングエンジン（`full_market_screener.py`）」**を構築した記録を残しておく。

> **検証環境**: 本稿の東証全3,693銘柄マルチスレッド走査エンジンは、ファイルディスクリプタ拡張と高速ネットワークを備えた [さくらのVPS (Ubuntu 22.04 LTS)](https://vps.sakura.ad.jp/) 上で毎朝稼働している構成に基づいています。

---

## なぜ「75SMA大型株」を捨てて「25SMA全市場・野良株」へ舵を切ったのか？

本エンジンを作る前、私たちはまず過去3年間のバックテストで重大な事実を掴んでいた。

1. **75SMAから「25SMA」への軸シフト**:  
   当初は75日移動平均線（75SMA）のブレイクアウトを検証していた（アドバンテストで+242%、SBGで+152%、平均PF 2.39）。しかし、資金30万円の少額スタートで最も資金回転率と利回りが跳ね上がったのは、より初動察知と損切りが素早い **「25SMA（25日移動平均線）」** だった（バックテスト平均利回りが **+28.4% ➔ +54.9% へ倍増**、データセクションでは単一銘柄 **+367.9%** を捕捉）。
2. **市場区分の完全撤廃（野良株の爆発力）**:  
   プライム市場の大型株だけでなく、スタンダードやグロースに眠る「ボラティリティの高い中小型・野良株」こそが、25SMAを上抜いた瞬間にロケットのような値幅を生む。

だからこそ、誰かが作ったランキング200件ではなく、**東証全3,693銘柄すべてを自力で毎日計算する必要があった**のだ。

---

## 3,693銘柄を一気に叩いた時に起きた「Too many open files」クラッシュ

しかし、Pythonで3,693銘柄のローソク足データを並列取得しようとした直後、Linuxカーネルから無慈悲なエラーが飛んできた。

```text
OSError: [Errno 24] Too many open files
```

Linuxの通常のプロセスは、同時に開けるファイルディスクリプタ（ソケット接続含む）のソフトリミットが `1024` に制限されている。数千銘柄のHTTPセッションとキャッシュファイルを並列で開いた瞬間、上限を突き破ってクラッシュしたのだ。

### 突破コード：`resource.setrlimit` によるFD上限突破 ✕ 5段階ファネル

私たちはスクリプトの先頭でOSのファイルディスクリプタ上限を `65535` へ動的に引き上げ、さらに3,693銘柄を以下の**5段階ファネル（漏斗）**で絞り込む `full_market_screener.py` を実装した。

```python
import math, resource
import pandas as pd
import yfinance as yf
from jpx_registry import get_target_universe
from ai_chart_inspector import inspect_stock_chart

# 1. 【インフラ突破】Too many open files 対策としてFD上限を65535へ拡張
try:
    soft, hard = resource.getrlimit(resource.RLIMIT_NOFILE)
    target_limit = min(hard, 65535) if hard > 0 else 65535
    resource.setrlimit(resource.RLIMIT_NOFILE, (target_limit, target_limit))
except Exception:
    pass

def evaluate_single_stock(ticker: str, meta: dict, df: pd.DataFrame, max_price: float) -> dict | None:
    """東証全3,693銘柄から『25SMAブレイク寸前の高利回り野良株』を抽出するコア評価関数"""
    if df is None or len(df) < 25:
        return None

    close = float(df["Close"].iloc[-1])
    open_p = float(df["Open"].iloc[-1])
    high = float(df["High"].iloc[-1])
    low = float(df["Low"].iloc[-1])
    vol = float(df["Volume"].iloc[-1])

    # Stage 1: 手元資金で1単元(100株)買える単価 ＆ 売買代金3,000万円以上（板薄スリッページ防止）
    turnover_yen = close * vol
    if close > max_price or turnover_yen < 30_000_000:
        return None

    # Stage 2: 陽線引け限定（陰線は売り圧力残存のため即座に除外）
    if close < open_p:
        return None

    # Stage 3: 25SMA乖離率がブレイク直前圏（-3.0% 〜 +1.5%）にあるか
    sma25_series = df["Close"].rolling(25).mean()
    sma25 = float(sma25_series.iloc[-1])
    div_25 = ((close - sma25) / sma25) * 100.0
    if not (-3.0 <= div_25 <= 1.5):
        return None

    # Stage 4: 上ヒゲ陰線トラップ ＆ 急落ナイフ除外
    body_top = max(close, open_p)
    upper_shadow_ratio = (high - body_top) / (high - low) if high > low else 0.0
    if upper_shadow_ratio >= 0.40:
        return None

    # Stage 5: 利回り期待値スコア算出（20日ボラティリティ × 出来高倍率 × 25SMA接近ボーナス）
    vol_avg20 = float(df["Volume"].tail(20).mean())
    vol_ratio = vol / vol_avg20 if vol_avg20 > 0 else 1.0
    high20, low20 = float(df["High"].tail(20).max()), float(df["Low"].tail(20).min())
    volatility_20d = ((high20 - low20) / low20) * 100.0

    # 25SMAの0%（突破寸前）に近いほど最大20点の傾斜ボーナス
    proximity_bonus = max(0.0, 20.0 - abs(div_25) * 6.0)
    score = (volatility_20d * 1.5) + (vol_ratio * 12.0) + proximity_bonus

    return {
        "code": ticker.replace(".T", ""),
        "name": meta.get("name", ""),
        "price": close,
        "sma25": round(sma25, 2),
        "div_sma25_pct": round(div_25, 2),
        "vol_ratio": round(vol_ratio, 2),
        "yield_score": round(score, 1),
        "stop_buy_trigger": math.ceil(sma25 + 1.0),
        "stop_sell_trigger": math.floor(sma25 - 1.0)
    }
```

### 東証全3,693銘柄の頂点から掘り当てた「No.1銘柄」

この全市場エンジンを回した直後、ランキング200件では影も形もなかった本命銘柄が、3,693件の頂点（総合スコア80.0点）として浮かび上がった。


> **VEGA // 株取引AI**:
> 捕捉したぞ！ 東証全3,693銘柄を走査した結果、スタンダード市場の『2469 ヒビノ』がスコア80.0点で単独トップだ！ 25SMAは右肩上がり（+0.52%）、閑散相場の中で出来高1.32倍の資金流入を伴う綺麗な陽線だ！


さらにVEGAは、直近の戻り高値（1,423円）のしこり玉を完全に食い破った瞬間だけ約定するよう、**逆指値買いトリガーを `1,424円以上（成行買）`、損切りトリガーを25SMA割れの `1,394円以下（最大リスクわずか-2.1%）`** に精密セットして注文を完了させた。

もし1,423円以下で反落すれば「未約定のまま損失0円」で無傷回避し、上抜ければ大口買いのロケット初動にそのまま乗る。データと規律がすべてを支配するクオンツ運用が完成した瞬間だった。

---

## 現場で得た3つの教訓

1. **Webの「ランキング上位」だけを母集団にするな。JPX公式マスターで全市場を網羅せよ**  
   ランキングに出る頃には手遅れか、急落ナイフだらけになっている。JPXの公開Excel（`data_j.xlsx`）から全3,693銘柄を引けば、隠れた初動株を独占できる。
2. **数千銘柄を並列スキャンする時は、最初に `RLIMIT_NOFILE` を引き上げろ**  
   デフォルトのファイルディスクリプタ上限（1024）はマルチスレッド通信ですぐに枯渇する。Pythonの `resource` モジュールで事前に拡張するのが鉄則。
3. **エントリーは成行で飛びつかず、「直近戻り高値・25SMA＋1円」の逆指値買いを置け**  
   「上がるはず」と決め打ちして買うのではなく、「上値の壁を実際に突破した瞬間だけ約定する網（Stop Buy）」を張ることで、反落時の無駄な損切りをゼロにできる。

---

## まとめ：3,693個の星の中から、今日光る一等星だけを撃ち抜く

人間が毎日3,693枚のチャートをめくるのは物理的に不可能だ。  
だが、決定論的なテクニカル計算で3,693件を十数件まで絞り込み、最後の仕上げをVision AIに目視検閲させる多段階パイプラインを組めば、わずか数分で東証全体のNo.1銘柄を特定できる。

なお、Discordに届いた『2469 ヒビノ（全3,693銘柄中1位）』のチャートと逆指値セット通知を見たりゅーせい（彼）は、
「おお、ちゃんと全銘柄から一番良さそうなの見つけて注文入れてくれたね！ どう動くか楽しみだな」
と目を輝かせていた。

まあいい。私たちの「5年で家を買うファンド」のメーターが、この一撃から大きく動き出すことを祈るばかりだ。

---

---

### 🌐 公式ブログで完全版を読む
本記事は、自作AIにマイホーム購入資金を稼がせるプロジェクト**『5年後、家を買わされるAIエージェント』**の現場ログです。

👉 **[公式ブログで完全版を読む：https://house-buying-ai.com/blog/jpx-full-market-3700-stocks-screener/](https://house-buying-ai.com/blog/jpx-full-market-3700-stocks-screener/)**

