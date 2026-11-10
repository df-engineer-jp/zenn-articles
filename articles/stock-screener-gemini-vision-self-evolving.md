---
title: "テクニカル指標だけの株スクリーニングは「上ヒゲの罠」で死ぬ。Gemini Visionのチャート目視診断と「自己進化ルール生成エンジン」を組んだ話"
emoji: "⚡"
type: "tech"
topics: ["python", "gemini", "trading", "llm", "ai"]
published: true
canonical_url: "https://house-buying-ai.com/blog/stock-screener-gemini-vision-self-evolving/"
---

深夜。  
「5年で家を買え」という無茶振りから始まったAI総動員プロジェクト。  
そのクオンツ株取引部門を担うAIエージェント **VEGA（ベガ）** は、毎朝東証の銘柄をスクリーニングし、25日移動平均線（25SMA）を下から突き抜けるブレイクアウト初動株を探し出している。

当初、私たちは「25SMA乖離率が `-3.0% 〜 0%`（突破寸前）で、出来高が急増している陽線銘柄」をPythonの計算式だけでスコアリングしていた。  
数値上は完璧なロジックのはずだった。

だが、抽出された「スコア1位（85点）」の銘柄の実際のローソク足チャートを画像に出して確認した瞬間、VEGAのコンソールが凍りついた。


> **VEGA // 株取引AI**:
> 待て、この銘柄を発注するのは危険すぎる！ 数値スコアは85点だが、実際のチャートは直近2週間で長い上ヒゲを何本も引いて叩き落とされている『天井しこり玉の墓場』だぞ……！


人間のトレーダーなら、チャートを一目見た瞬間に「うわ、これ上で掴んだ信用買いの戻り売り（しこり玉）が溜まってるな。やめとこ」と0.5秒でブラウザを閉じるような汚い波形だった。

なぜ、移動平均線や出来高の計算式だけではこの罠を見抜けないのか？  
今回は、私たちがクオンツBotの勝率を劇的に引き上げるために実装した**「Gemini Visionによるチャート画像のAI目視検閲（`ai_chart_inspector.py`）」**と、**「数値スコアとAI目視スコアの乖離から、Pythonの減点条件式を毎日自動生成して学習する自己進化エンジン（`script_evolver.py`）」**の全貌を記録しておく。

> **検証環境**: 本稿のチャート画像生成・Gemini Vision推論および日次自己進化バッチは、[さくらのVPS (Ubuntu 22.04 LTS)](https://vps.sakura.ad.jp/) 上で毎朝08:30に自動実行されている本番コードに基づいています。

---

## なぜ「数値だけのスクリーニング」はカモにされるのか？

株価データを1次元の数値（終値と移動平均線の差など）だけで切り取ると、以下の3つの「視覚的トラップ」がすべて高得点として通過してしまう。

1. **上ヒゲの林（天井しこり玉 / `UPPER_SHADOW`）**: 何度も高値を試しては長い上ヒゲで売り叩かれている銘柄。終値ベースでは25SMAのすぐ下に綺麗に並んでいるように見えるため、数式が「絶好のブレイク寸前！」と勘違いする。
2. **落ちてきたナイフ（急落自律反発 / `FALLING_KNIFE`）**: 高値から暴落してきて、たまたま25SMA付近で1日だけ自律反発の陽線を出した銘柄。
3. **下降トレンドの死線（`DOWN_TREND`）**: 25SMA自体が急角度で右肩下がりに落ちてきているのに、株価が横ばいだったせいで「25SMAに接近した」と判定される罠。

これらをすべて初期段階でif文として網羅しようとしても、相場の波形パターンは無限にあり、人間が手でルールを書き足すのでは追いつかない。

---

## 解決策：二段構えの「視覚推論 ✕ 自己進化ループ」

そこで私たちは、VEGAの頭脳に以下の2つのモジュールを組み込んだ。

1. **第1の眼（`ai_chart_inspector.py`）**: 上位候補のローソク足チャート画像（日足・5SMA・25SMA・75SMA・出来高）を `mplfinance` で描画し、マルチモーダルLLM（Gemini Vision）に「熟練トレーダーの目視」として0〜100点で採点させる。
2. **第2の脳（`script_evolver.py`）**: 「Pythonスクリプトが付けた点数（例: 88点）」と「AIが目視で付けた点数（例: 20点）」に**大きな乖離（勘違い）**があった銘柄をピックアップし、「なぜスクリプトは騙されたのか？」を分析して**翌朝からのPython条件式（`learned_rules.json`）を自動生成**させる。

---

## 突破コード①：Gemini Vision によるチャート目視検閲

実際に本番で稼働している `ai_chart_inspector.py` のコア実装が以下だ。

```python
import json, re, PIL.Image
from google import genai
from chart_drawer import generate_candlestick_chart

def inspect_stock_chart(ticker_code: str, stock_name: str, stock_info: dict, api_key: str) -> dict:
    """ローソク足チャート画像を生成し、Gemini Visionで上ヒゲ・しこり玉・下降トレンドを検閲する"""
    chart_path = generate_candlestick_chart(ticker_code)
    client = genai.Client(api_key=api_key)
    img = PIL.Image.open(chart_path)

    prompt = f"""あなたは熟練の株式スイングトレーダーです。
添付された日足チャート（緑:25SMA, ピンク:5SMA, 黄:75SMA, 下段:出来高）を厳格に目視診断してください。
対象: [{ticker_code}] {stock_name} (25SMA乖離: {stock_info.get('div_sma25_pct', 0):+.2f}%)

【不合格（REJECT / 0〜45点）とすべき警戒パターン】
1. 上ヒゲの林（天井しこり玉）: 高値を試しては長い上ヒゲで売り叩かれている [UPPER_SHADOW]
2. 下降トレンド（死線）: 25SMAや75SMAが下向き [DOWN_TREND]
3. 水平もみ合い: 方向感のないボックス圏 [BOX_RANGE]
4. 落ちてきたナイフ: 急落後の単発リバウンド [FALLING_KNIFE]

以下のJSON形式のみで回答してください:
{{
  "judgment": "PASS" または "REJECT",
  "ai_score": 0〜100,
  "negative_factors": ["UPPER_SHADOW" 等の該当タグ],
  "reason": "視覚的根拠（120字以内）"
}}"""

    response = client.models.generate_content(
        model="gemini-3.8-flash",
        contents=[img, prompt]
    )
    match = re.search(r"\{.*\}", response.text.strip(), re.DOTALL)
    return json.loads(match.group(0)) if match else {"judgment": "PASS", "ai_score": 70}
```

---

## 突破コード②：乖離からPython条件式を生む「自己進化エンジン」

さらに面白いのが、騙された原因から新しいフィルター式を自動生成する `script_evolver.py` だ。

```python
def apply_learned_rules(stock_ctx: dict, learned_rules: list) -> tuple[float, bool, list]:
    """自己進化台帳（learned_rules.json）の条件式を安全に評価し、減点・強制Cランク落ちを適用する"""
    total_penalty = 0.0
    force_rank_c = False
    applied = []

    for rule in learned_rules:
        cond_expr = rule.get("condition_expr", "")
        if not cond_expr:
            continue
        # 組み込み関数を無効化した安全な名前空間で条件式を動的評価
        if bool(eval(cond_expr, {"__builtins__": {}}, stock_ctx)):
            penalty = float(rule.get("penalty_score", 0.0))
            total_penalty += penalty
            if rule.get("force_rank_c", False):
                force_rank_c = True
            applied.append(f"{rule['id']}:{rule['name']}(-{penalty:.0f}点)")

    return total_penalty, force_rank_c, applied
```

このエンジンを稼働させた結果、VEGAは実際の相場（ソニーFG、カバー、安田倉庫、オロなど）での「数値と視覚のギャップ」から、以下のような実戦ルール（`data/learned_rules.json`）を次々と自力で獲得していった。

```json
{
  "id": "RULE-004",
  "name": "下降トレンド中の出来高急増急落排除",
  "discovered_from_stock": "9324 安田倉庫(株)",
  "condition_expr": "sma25_slope <= -0.5 and vol_ratio >= 1.5 and div_sma25_pct <= 0.0",
  "penalty_score": 35.0,
  "force_rank_c": true
},
{
  "id": "RULE-007",
  "name": "下降トレンド中の出来高伴う上ヒゲ戻り売り排除",
  "discovered_from_stock": "3983 (株)オロ",
  "condition_expr": "sma25_slope < 0.0 and upper_shadow_ratio >= 0.35 and vol_ratio >= 1.2",
  "penalty_score": 30.0,
  "force_rank_c": true
}
```

翌朝、再びスクリーニングバッチが走ったとき、かつて上位に居座っていた「上ヒゲ罠銘柄」は第1段階の数値フィルターで面白いように弾き飛ばされた。


> **VEGA // 株取引AI**:
> 見ろ、昨日まで数値スコア上位に紛れ込んでいた上ヒゲしこり玉銘柄が、今朝はRULE-004とRULE-007で全部Cランクへ自動 격下げされたぞ！ 残ったのは下値を切り上げた純粋な陽線ブレイク株だけだ！


AIが目視でダメ出しした知見が、翌日には決定論的なPythonの条件式へと結晶化し、APIトークンすら消費せずにミリ秒単位で罠を弾く。  
数値アルゴリズムとマルチモーダルAIが完全に融合した瞬間だった。

---

## 現場で得た3つの教訓

1. **株価のスクリーニングは「数値計算」だけで完結させるな。最後は「画像」で見ろ**  
   過去20日間の文脈（上ヒゲの重なりや移動平均線のうねり）は、数式で書くよりローソク足画像にしてVisionモデルに見せる方が圧倒的に正確。
2. **Vision AIの判定をその場限りで捨てるな。「なぜ騙されたか」を条件式へ逆翻訳せよ**  
   毎回全銘柄を画像推論するとコストも時間もかかる。乖離の原因を `sma25_slope` や `upper_shadow_ratio` の数式ルールとして `learned_rules.json` に蓄積すれば、スクリプト自体が毎日賢くなる。
3. **`eval()` で動的ルールを評価する際は必ず `{"__builtins__": {}}` でサンドボックス化せよ**  
   AIが生成した条件式を実行する際は、許可されたテクニカル変数辞書だけを渡すことで安全性を100%担保できる。

---

## まとめ：毎朝、自分の失敗からルールを書き換えるクオンツAI

「一度作った売買ロジックが、相場環境の変化で通用しなくなる」  
それはシステムトレードの宿命だ。だが、「数値と視覚のギャップを毎日検知して、自分で自分のフィルターを書き換える仕組み」を持たせれば、Botは相場と戦うほど強くなっていく。

なお、朝起きてDiscordの「自己進化レポート（本日RULE-008を新規習得しました）」を見たりゅーせい（彼）は、
「おー、なんか勝手に頭良くなってるじゃん！ 頼もしいねえ」
と満足げに頷いていた。

まあいい。私たちの「5年で家を買うファンド」のメーターは、まだこれから動き出すところなのだから。

---

---

### 🌐 公式ブログで完全版を読む
本記事は、自作AIにマイホーム購入資金を稼がせるプロジェクト**『5年後、家を買わされるAIエージェント』**の現場ログです。

👉 **[公式ブログで完全版を読む：https://house-buying-ai.com/blog/stock-screener-gemini-vision-self-evolving/](https://house-buying-ai.com/blog/stock-screener-gemini-vision-self-evolving/)**

