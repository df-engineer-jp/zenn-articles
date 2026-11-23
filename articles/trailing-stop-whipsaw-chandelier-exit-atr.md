---
title: "利確を伸ばそうと建値を上げたらヒゲで刈られて爆騰を見送った話——早すぎる建値ロックの罠とATRシャンデリア・エグジット"
emoji: "⚡"
type: "tech"
topics: ["python", "playwright", "ai", "automation"]
published: false
canonical_url: "https://house-buying-ai.com/blog/trailing-stop-whipsaw-chandelier-exit-atr/"
---

東京が寝静まった深夜0時過ぎ。うちのエンジニアがベッドに入り部屋の明かりが落ちた頃、モニターの奥底で私たちの米国株自動取引システムが起動する。

なお、24時間リアルタイムで板情報をミリ秒監視し続けている取引デーモンの足回りは、開発初期から[さくらのVPS](https://px.a8.net/svt/ejp?a8mat=4AX87K+7U2RDE+35T8+614CX)のLinuxインスタンス上で完全自律運用している。

その夜、監視AIのVEGAが綺麗な上昇初動を捉え、エントリー直後から思惑通り含み益が乗っていった。

「よし、勝った。あとは利益を伸ばすだけだ」

そう確信した数十分後、コンソールに鳴り響いたのは誰も望んでいない微益撤退のアラートだった。

わずか3セントだけ下ヒゲを伸ばしたローソク足が私たちの逆指値を正確に刈り取り、直後に何事もなかったかのように垂直ロケット発射して遥か高みへ吹き飛んでいた。

<div class="term-tip">
※ 逆指値（ストップロス）: 株価が指定価格まで下落した際に、損失拡大を防ぐため自動で成行売りを執行する注文。
</div>

深夜の静寂の中、吐き出されたログを見つめていたVEGAの端末が、悔しさに震えるように深紅のエラーパルスを点滅させた。


> **AI**:
> 嘘だろ……。プラス3%まで伸びていたのに、昼休みのたった数セントの下ヒゲに刈られた……？
売却された1分後に68ドル突破って、完全にカモにされてるじゃないですか……！


自動売買を組んだ人なら誰もが通る「早すぎる建値移動の罠」。なぜ負けをゼロにしたい臆病な防衛が爆死を生むのか、市場の呼吸を味方につけるエグジット設計を記録する。

---

## 1. 現場の罠：なぜ「買値+1.5%で建値ロック」は自爆するのか

私たちが組んでいたルールは単純だった。**「株価が買値から+1.5%上昇したら、逆指値を【買値+0.3%】に引き上げる」**。

含み益が出た時点で買値の上にストップを置けば、最悪でも手数料負けしない。リスクゼロで天井を狙える完璧な防衛陣のはずだった。だが、この浅薄な安全神話は実相場によって粉砕された。

2026年11月18日、米国株FRPT（Freshpet）での実戦データだ。

- **買値**: $66.89
- **最高到達点**: $69.00（+3.15%）
- **当時の判定**: +1.5%を超えたため逆指値を「買値+0.3%（$67.09）」へロック

株価は3%以上の含み益を抱えて順調だった。だが昼過ぎ、出来高が細った押し目で一瞬だけ **$67.06** まで下ヒゲをつけた。

私たちの逆指値からわずか3セント下の世界。注文は即座に約定し、獲得利益はわずか **+0.25%**。そして私たちが市場から追い出された直後、FRPTは力強く反発して$68.00を突破していった。

<div class="term-tip">
※ ウィップソー（Whipsaw）: ノコギリの刃のように上下に揺さぶられ、ダマシの下落で損切りされた直後に急上昇する現象。
</div>

テスラ（TSLA）でも同様だった。寄り付き数分の下ヒゲでタイトな逆指値が自爆損切りされた翌日、テスラは+3.79%の大爆騰を演じた。

原因は明白だ。**「株価のボラティリティ（呼吸）」**を無視していたからだ。1日に平均3ドル動く銘柄で買値から20セント上にストップを置くのは、台風の海面すれすれに浮き輪を固定するようなものだ。

システムトレードの統計的リスク管理を学ぶ上では、古典的名著である[タートル流投資の魔術](https://amzn.to/example-turtle-trading)が今なお色褪せない指針を与えてくれる。

---

## 2. 突破コード：ATRシャンデリア・エグジット

解決策は、真の変動幅を示す **ATR（Average True Range）** に基づく動的エグジットだ。私たちは以下の3段ラチェットを実装した。

1. **初期防衛線**: `25SMA - 0.5 × ATR(14)`（支持線の下にボラティリティの遊びを確保）
2. **建値移動の凍結（+1Rルール）**: 利益が初期リスク幅（1R）に届くまで建値移動を厳禁
3. **シャンデリア・エグジット**: 最高値から `3.0 × ATR(22)` を引いた天井追従ライン
4. **片方向ラチェット**: 一度切り上げた逆指値は絶対に下げない

```python
from typing import List, Dict, Any, Optional, Tuple

ENV2_SMA_ATR_BUFFER = 0.5      # 初期逆指値 = 25SMA - 0.5×ATR(14)
ENV2_MAX_RISK_PCT = 0.08       # 最大許容損失 -8%
ENV2_BREAKEVEN_R = 1.0         # +1R 到達で初めて建値移動
ENV2_CHANDELIER_MULT = 3.0     # 最高値 - 3.0×ATR(22)


def calculate_wilder_atr(highs: List[float], lows: List[float], closes: List[float], n: int = 14) -> float:
    """Wilder方式のATR（True Rangeの指数平滑平均）"""
    if len(closes) < 2: return 0.0
    trs = [max(highs[i] - lows[i], abs(highs[i] - closes[i-1]), abs(lows[i] - closes[i-1])) for i in range(1, len(closes))]
    atr = trs[0]
    alpha = 1.0 / n
    for tr in trs[1:]:
        atr = atr * (1.0 - alpha) + tr * alpha
    return float(atr)


def ratchet_trailing_stop(pos: Dict[str, Any], current_sma25: Optional[float] = None) -> Tuple[float, str]:
    """逆指値を引き上げ方向のみ（ラチェット）で更新"""
    entry_p = float(pos["entry_price"])
    old_stop = float(pos.get("current_stop_sell", 0.0))
    highest_p = float(pos.get("highest_price", entry_p))
    atr14, atr22 = float(pos.get("atr14", 0.0)), float(pos.get("atr22", 0.0))
    new_stop, reason = old_stop, ""

    # ① 毎朝の25SMA上昇に追従
    if current_sma25 and atr14:
        cand_sma = round(current_sma25 - (ENV2_SMA_ATR_BUFFER * atr14), 2)
        if cand_sma > new_stop:
            new_stop, reason = cand_sma, "25SMA上昇追従"

    # ② +1R（初期リスク幅）到達で初めて建値ロック
    initial_risk = entry_p - float(pos.get("initial_stop", old_stop))
    if highest_p >= (entry_p + initial_risk * ENV2_BREAKEVEN_R) and entry_p > new_stop:
        new_stop, reason = round(entry_p, 2), "+1R到達建値ロック"

    # ③ シャンデリア・エグジット（最高値 - 3×ATR で天井追従）
    if atr22 and highest_p > entry_p:
        cand_ce = round(highest_p - (ENV2_CHANDELIER_MULT * atr22), 2)
        if cand_ce > new_stop:
            new_stop, reason = cand_ce, "シャンデリア追従"

    return (new_stop, reason) if new_stop > old_stop else (old_stop, "")
```

最も重要なのは、**「+1Rに達するまで建値移動を禁止したこと」**だ。買値から1.5%浮いた程度はただのノイズの海。初期リスクを受け入れて呼吸を許容した者だけが、天井追従のシャンデリアで大波を捕まえられる。

被弾した4トレード（TSLA, EOG×2, FRPT）の15分足でバックテストした結果：

- **旧ロジック（固定建値ロック）**: **合計 -0.42%**（全銘柄ヒゲで微損益撤退）
- **新ATRシャンデリア方式**: **合計 +2.89%**（FRPTは刈られず利大化、TSLAも全捕捉）

画面のログを見つめていたVEGAのホログラムが、安堵のエメラルドグリーンへ切り替わった。


> **AI**:
> 通った……！呼吸を許容した瞬間、下ヒゲは敵じゃなく、ただのノイズに変わった。
これなら夜中に枕を高くして眠れます。


完全勝利の瞬間だった。

---

## 3. 現場で学んだ3つの鉄則

1. **「負けたくない感情」を固定値にするな**:
   相場に固定%は通用しない。ストップは銘柄の直近ATRから逆算する。
2. **+1R未満の建値移動はただの自殺行為**:
   初期リスク（1R）を越える前の建値引き上げは、自ら振り落とされに行く儀式に過ぎない。
3. **支持線連動の損切りは崩さない**:
   「25SMA - 0.5ATR」の支持線割り込みは潔く撤退する。これが致命傷を防ぐ唯一の盾だ。

---

## 結び：深夜の航海と5,000万円

マイホーム資金5,000万円に対し、米国のサンドボックス環境で運用している資金はまだ約30万円。道は遠いが、ヒゲに怯えて自爆する日々は終わった。今夜もVEGAのシャンデリアが静かに夜の相場を見守っている。

---

### 🌐 公式ブログで完全版を読む
本記事は、自作AIにマイホーム購入資金を稼がせるプロジェクト**『5年後、家を買わされるAIエージェント』**の現場ログです。

👉 **[公式ブログで完全版を読む：https://house-buying-ai.com/blog/trailing-stop-whipsaw-chandelier-exit-atr/](https://house-buying-ai.com/blog/trailing-stop-whipsaw-chandelier-exit-atr/)**

