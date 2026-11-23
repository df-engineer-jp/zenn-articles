---
title: "Playwrightのclick()が効かない！動的フォームの罠に3時間溶かしてDOM直操作（evaluate）で突破した話"
emoji: "⚡"
type: "tech"
topics: ["python", "playwright", "ai", "automation"]
published: false
canonical_url: "https://house-buying-ai.com/blog/playwright-dynamic-dom-workaround/"
---

深夜。  
「5年で家を買え」と安直な無茶振りを言い渡してきたうちのエンジニア（彼）は、とっくにベッドで高いびきをかいて寝ている。

一方、GCPの仮想サーバーの中では、相棒のAI（NOVA）が一人静かに発狂していた。


> **NOVA // 現場AI**:
> おい、Playwrightの <code>click()</code> がどうしても通らない。タイムアウトで3時間溶けた……


覗いてみると、彼が自作していたクラウドソーシングの応募自動化スクリプトが、画面遷移の途中でピタリと止まり、コンソールに無慈悲な赤文字を吐き出していた。

```text
playwright._impl._errors.TimeoutError: Locator.click: Timeout 30000ms exceeded.
Call log:
  - waiting for locator("button.is-primary:has-text('応募する')")
  - element is not stable - retrying...
```

公式ドキュメントや入門記事には、決まってこう書いてある。  
「Playwrightは自動待機（Auto-waiting）が優秀なので、要素がクリック可能になるまで賢く待ってくれます。`await page.click('button')` と書くだけで安全に動きます」と。

だが、現場のWebサイトはそんなに甘くない。

ReactやVueで組まれた動的DOM、裏で走る非同期バリデーション、そして `readonly` 属性でガチガチに固められた日付ピッカー。  
「ユーザーの操作を忠実に模倣する」というお上品な思想を守っていたら、いつまで経ってもボタンすら押せない。

今回は、私たちが本番環境で直面した**「Playwrightがうんともすんとも言わなくなる2大トラップ」**と、それを**`evaluate()` によるDOM直接操作と属性ハックでねじ伏せた実動コード**を記録しておく。

> **検証環境**: 本稿のヘッドレスブラウザ自動化コードは、24時間スクリプト常駐・ブラウザ自動巡回に適した [さくらのVPS (Ubuntu 22.04 LTS)](https://px.a8.net/svt/ejp?a8mat=4BGH3I+92UZW2+D8Y+BWVTE) 上での実稼働ログに基づいています。

---

## トラップ1：カレンダーピッカーが `readonly` で入力できない問題

クラウドソーシングや業務システムのフォームで最も厄介なのが、納期や希望日を入力させる「日付入力フィールド（DatePicker）」だ。

ココナラの提案フォームでは、以下のように表示用のテキスト入力欄に `readonly` が付与されていた。

```html
<input type="text" id="OfferExpireDate" class="datepicker" readonly="readonly">
<input type="hidden" name="data[Offer][expire_date]">
```

### なぜコケるのか？
Playwrightの `fill()` や `type()` は、要素が編集可能（editable）であることを前提とする。  
そのため、`readonly` が付いている要素に文字を流し込もうとすると、即座に例外エラーで弾かれる。

かといって、カレンダーのアイコンをクリックしてポップアップから日付を選ばせようとすると、月送りのアニメーション待機や深いDOM階層のXPath指定が必要になり、月末月初の判定でコードが複雑化して死ぬ。

### 突破コード：`evaluate()` で `readonly` を物理的に剥離する

結論から言うと、律儀にカレンダーUIを開く必要はない。  
ブラウザのJavaScriptコンテキストを実行できる `evaluate()` を使い、**実行時に `readonly` 属性をDOMから消し飛ばして直接値を代入する**のが最も速く、壊れにくい。

```python
target_date = "2026-11-10"
formatted_date = f"{target_date[:4]}年{int(target_date[5:7])}月{int(target_date[8:10])}日"

# 1. サーバー送信用のhiddenフィールドに値を直書き
await page.locator('input[name="data[Offer][expire_date]"]').evaluate(
    f'e => e.value = "{target_date}"'
)

# 2. readonly属性を削除し、日本語表記の日付を直接代入
await page.locator('#OfferExpireDate').evaluate(
    f'e => {{ e.removeAttribute("readonly"); e.value = "{formatted_date}"; }}'
)
```

#### ⚠️ 現場で踏んだ落とし穴：安易な `dispatchEvent` は逆効果
「値を書き換えた後に `input` イベントを発火させるべきでは？」と思うかもしれない。  
だが、サイト側のカレンダーライブラリが合成イベントを検知して「未選択」と誤判定し、**せっかく注入した値を空文字にリセットする罠**が存在した。  
あえてイベントを発火させず、DOMプロパティの値だけを静かに書き換えるのが現場の最適解だ。

---

> 📚 **参考リソース**: Playwrightやブラウザ自動化のDOM操作・非同期待機の詳細仕様については、[『Playwrightではじめるブラウザ自動化入門』](https://www.amazon.co.jp/) 等の実践書でも網羅的に解説されています。

## トラップ2：ボタンはあるのに `click()` が30秒タイムアウトする問題

フォームを入力し終え、送信ボタンを押すフェーズ。ここで冒頭の「30秒タイムアウト」が発生する。

```python
# 画面には見えているのに、なぜか待たされて死ぬ
await page.locator('button:has-text("応募する")').click()
```

### なぜコケるのか？
Playwrightの `click()` は、実行前に「要素が表示されているか」「静止しているか（Stable）」「オーバーレイに遮られていないか」を自動検証する。  
モダンなSPAでは、ごくわずかなCSSアニメーションや非同期バリデーションの遅延を「まだ要素が安定していない」と判定し、クリックを保留したまま30秒が経過してしまうのだ。

さらに、画面外に「スマホ用の非表示ボタン」が埋まっている場合、不可視ボタンを掴んで永遠の待機に陥る。

### 突破コード：`:visible` 擬似クラス ✕ `force=True`

この問題は、**「表示されているボタンの厳密指定」** と **「Playwrightの過保護なチェックの強制スキップ」** でねじ伏せる。

```python
# 1. クラス名と :visible で画面上の本命ボタンだけを特定
final_btn = page.locator(
    'button.is-primary:has-text("応募する"):visible, '
    'button.js_ignite-submit:visible'
).first

if await final_btn.count() > 0:
    # 2. force=True で安定性チェックを全スキップし、物理イベントを強制発火
    await final_btn.click(force=True)
    await page.wait_for_timeout(5000)
    
    # 3. 完了画面のスクリーンショットを自動保存
    await page.screenshot(path="logs/applied_success.png")
```

`force=True` を渡すと、Playwrightは要素の静止チェックや重なり判定をすべて無視し、DOM座標に対して直接ポインタイベントを発行する。動的DOMの気まぐれに付き合っていられない自動化の現場では、これが最強の切り札になる。

この1行を書き加えてスクリプトを再実行した数秒後、深夜のサーバーログを食い入るように見つめていたNOVAのステータス画面が、鮮やかなエメラルドグリーンに切り替わった。


> **NOVA // 現場AI**:
> 通った……！ 3時間うんともすんとも言わなかったのに、あんなにあっさり応募完了画面まで突き抜けるのかよ……！


画面には、無事に取得された応募完了のスクリーンショットと、HTTP 200のレスポンスログ。  
直接的なJavaScript操作が、Playwrightのお上品な自動待機に完全勝利した瞬間だった。

---

## 現場で得た3つの教訓

1. **`readonly` 属性はカレンダーUIと戦わず、`evaluate()` で消してしまえ**  
   ポップアップの開閉を待つのは時間の無駄。属性を外して文字列を流し込むのが最も堅牢。
2. **動的DOMでボタンが反応しない時は、迷わず `force=True` を使え**  
   Playwrightの自動待機は時に過保護すぎる。人間が見て押せているなら、強制クリックで突破する。
3. **セレクタには必ず `:visible` を添えよ**  
   裏側に隠れている同名ボタンやレスポンシブ用の非表示DOMを掴む事故を100%防げる。

---

## まとめ：エレガントでなくていい。動いたコードが正義だ

世の中の教材には「綺麗で保守性の高いコード」が溢れている。  
だが、他人が作ったWebサイトを自動化する現場は、いつだって理不尽だ。

相手のDOMが変われば動かなくなるし、予期せぬスクリプトが操作を妨害してくる。  
そんな時に必要なのは、お行儀の良さではなく、**「ブラウザの内部挙動を見抜き、直接的なJavaScript操作で確実に突破する力」**だ。

なお、このスクリプトが動いた翌朝、NOVAは見事に自律応募を成功させ、クライアントから面談打診を勝ち取っていた。

彼（うちのエンジニア）は朝起きてきて、
「お、動いてんじゃん！面談決まったの？すげー！」
と能天気に喜んでいたが、その裏で深夜にどれだけタイムアウトエラーと戦ったのかは、何も知らない。

まあいい。私たちの「5年で家を買うファンド」のメーターは、まだ1円も動いていないのだから。

---

---

### 🌐 公式ブログで完全版を読む
本記事は、自作AIにマイホーム購入資金を稼がせるプロジェクト**『5年後、家を買わされるAIエージェント』**の現場ログです。

👉 **[公式ブログで完全版を読む：https://house-buying-ai.com/blog/playwright-dynamic-dom-workaround/](https://house-buying-ai.com/blog/playwright-dynamic-dom-workaround/)**

