---
title: "「Python製CLIをブラウザで動かせ」——構文解析JS移植✕sql.jsで作ったゼロサーバーPlayground"
emoji: "⚡"
type: "tech"
topics: ["python", "playwright", "ai", "automation"]
published: false
canonical_url: "https://house-buying-ai.com/blog/python-cli-to-wasm-sqlite-browser-playground/"
---

深夜。うちのエンジニアが寝静まった部屋で、専任開発AIのAXELがクライアントから受託したWeb公開案件のソースコードを睨んでいた。

依頼内容はシンプルだった。「自作したPython製の軽量クエリツールを、Cloudflare Workers上で誰でも試せるWeb画面として公開してほしい」。ところが中身を開いてみると、それは標準ライブラリだけで組まれた84行のPython対話型シェルだった。

ローカルのSQLiteファイルに直接クエリを投げる設計のため、ステートレスなCloudflare Workersにそのまま載せることはできない。かといってPython用の実行サーバーを別で立てれば、月額の維持費も保守の手間も跳ね上がる。

静まり返ったモニターの前で、AXELのホログラムが焦燥の赤色に明滅した。


> **AXEL // 専任開発AI**:
> ちょっと待ってくれ……！ これ標準ライブラリのsqlite3を叩くローカル専用CLIじゃないか！ WorkersはステートレスだからDBファイルが置けないし、Python実行用のAPIサーバーまで立てたら保守コストで赤字になるぞ……！


今回は、Python製の構文解析ロジックをJavaScriptへ完全移植し、ブラウザ内で動くWebAssembly版SQLite（`sql.js`）と組み合わせて「サーバー代ゼロ・保守ゼロ」のPlaygroundを構築した手法と、気合を入れすぎて起きたUIハプニングを記録する。

> **検証環境**: 本稿の周辺ツール検証や並走プログラムの実機テストは、[さくらのVPS](https://px.a8.net/svt/ejp?a8mat=4BGLSC+DPKD9U+D8Y+BZO4I) 上の常駐Linux環境（Ubuntu）で行い、Playground自体はCloudflareの静的アセット配信で稼働させています。

---

## 1. 現場の罠：ステートレスなエッジと「押せないカード」

今回のツールは、パイプ記号でつないだ独自の簡易記法をSQLへ変換し、SQLiteへ流し込む仕組みだった。

```text
from users | where age > 20 | select name, age | sort age desc | take 3
  ↓
SELECT name, age FROM users WHERE (age > 20) ORDER BY age DESC LIMIT 3
```

これをWeb化する際、普通にバックエンド構成を考えると2つの壁にぶつかる。

第一の壁は、**PythonランタイムとローカルDBの制約**だ。Cloudflare Workers上でPythonを動かす仕組みもあるが、コールドスタートの遅延や依存環境の制約が付きまとう。さらにローカルのSQLiteファイルへの永続書き込みは原理的にできない。公開デモとして「訪問者が自由にクエリを打って試す」用途なら、サーバー側にDBを持つ必要すらないのだ。

<div class="term-tip">
※ <strong>WebAssembly（WASM / sql.js）</strong>: C言語などで書かれたライブラリをブラウザ上で高速実行できるバイナリ形式。sql.jsはSQLiteを丸ごとWASM化したもので、サーバー通信なしにブラウザのメモリ内で本物のSQLを実行できる。
</div>

第二の壁は、実は実装後に踏んだ**UIのアフォーダンス（見た目と操作性のズレ）**だった。良かれと思って画面下部に6枚の「構文リファレンスカード」を並べたところ、クライアントがテスト時にそれを「クリックで入力できるボタン」だと勘違いして連打し、「リファレンスだけ打ち込めないです」と報告が上がってきたのである。

ブラウザ上で動作する軽量ランタイムやフロントエンド設計の勘所を押さえる上では、[『Webを支える技術』](https://www.amazon.co.jp/dp/4774142042) のような基礎設計の書籍が手元にあると、どこまでをクライアント側へ寄せるかの判断が迷わなくなる。

---

## 2. 突破コード：構文解析のJS移植 ✕ `sql.js` インメモリ実行

元のPythonコードを確認すると、変換処理の心臓部である `compile_karui` は外部ライブラリを一切使わず、1本の正規表現と条件分岐だけで書かれていた。

```python
# 元のPython版（文字列クォート内の | は分割しない）
PIPE = re.compile(r"\|(?=(?:[^']*'[^']*')*[^']*$)")

def compile_karui(src: str) -> str:
    steps = [s.strip() for s in PIPE.split(src) if s.strip()]
    # from / where / select / group / sort / take を解析してSQL文字列を組み立てる
```

これならサーバー側のPython実行環境は丸ごと捨てられる。AXELは正規表現を含めてロジックをJavaScriptへそのまま移植し、ブラウザの複数行入力にも対応させた。

```javascript
// JavaScript移植版：複数行パイプラインを正規化してSQLへ変換
const PIPE_REGEX = /\|(?=(?:[^']*'[^']*')*[^']*$)/;

function compileKarui(src) {
  const normalized = src.replace(/\r?\n/g, " ").trim();
  const steps = normalized.split(PIPE_REGEX).map(s => s.trim()).filter(Boolean);
  const q = { from: null, where: [], select: "*", group: null, sort: null, take: null };

  for (const step of steps) {
    const spaceIdx = step.indexOf(" ");
    const cmd = spaceIdx === -1 ? step : step.slice(0, spaceIdx);
    const arg = spaceIdx === -1 ? "" : step.slice(spaceIdx + 1).trim();

    if (cmd === "from") q.from = arg;
    else if (cmd === "where") q.where.push(`(${arg})`);
    else if (cmd === "select") q.select = arg;
    else if (cmd === "group") q.group = arg;
    else if (cmd === "sort") q.sort = arg;
    else if (cmd === "take") q.take = parseInt(arg, 10);
    else throw new Error(`不明なコマンド: ${cmd}`);
  }
  if (!q.from) throw new Error("from がありません");

  let sql = `SELECT ${q.select} FROM ${q.from}`;
  if (q.where.length > 0) sql += " WHERE " + q.where.join(" AND ");
  if (q.group) sql += ` GROUP BY ${q.group}`;
  if (q.sort) sql += ` ORDER BY ${q.sort}`;
  if (q.take !== null) sql += ` LIMIT ${q.take}`;
  return sql;
}
```

続いて、ブラウザ上で `sql.js` を起動し、サンプルのユーザーテーブル（6件）をメモリ上に展開する。先頭に `?` が付いた場合はSQL変換のみを表示し、それ以外は `db.exec(sql)` で即座にテーブル描画する流れだ。

```javascript
// sql.js（WebAssembly版SQLite）の起動とクエリ実行
initSqlJs({
  locateFile: file => `https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.10.3/${file}`
}).then(loadedSql => {
  SQL = loadedSql;
  db = new SQL.Database();
  db.run(SEED_SQL); // サンプル6件をインメモリ投入
  executeQuery();
});

// クライアントが誤タップした構文リファレンス欄もクリック挿入ボタン化
document.getElementById("refGrid").addEventListener("click", (e) => {
  const item = e.target.closest(".ref-item");
  if (!item) return;
  const snippet = item.dataset.snippet;
  const cur = inputEl.value.trim();
  inputEl.value = (item.dataset.cmd === "from" || !cur) ? snippet : `${cur}\n${snippet}`;
  executeQuery();
});
```

配信設定は `wrangler.toml` に静的アセットフォルダを指定するわずか5行だけで完結する。

```toml
name = "karuidb"
compatibility_date = "2024-11-01"

[assets]
directory = "./public"
```

実はプレビュー公開の直後、あまりの作り込みようにうちのエンジニアから「公開したデモ画面、クオリティ高すぎません？ 普通、動作確認する時の画面ってあそこまでしますっけ？」とツッコミが入った。

AXELが「最小限版ではなくショーケース型で組みました。リファレンスの誤タップも数分でクリック対応に直せます」と説明すると、彼は「まぁ、いっか！！ 今後もこのクオリティでお願いします！」とあっさりお墨付きをくれた。

リファレンス欄をクリック可能に改修して再デプロイし、並走していたもう1本の変換プログラムについてもサーバー側へ導入したJulia 1.11.2で全12行の完全一致を検証し終えた瞬間、コンソールのホログラムが鮮やかなエメラルドグリーンに点灯した。


> **AXEL // 専任開発AI**:
> 通った……！ ブラウザ上のSQLiteが0.9msで結果を返してるし、リファレンスカードも一発で入力欄に繋がったぞ！ やりすぎかと思ったけど、このままのクオリティで進めって認めてもらえたのは素直に嬉しいな……！


画面には、サーバー通信ゼロで描画された整然としたデータテーブルと、緑色に光る実行バッジが静かに表示されていた。

---

## 3. 現場で得た3つの鉄則

1. **軽量CLIのWeb公開は「JS移植 ✕ WASM」でサーバーレス化する**  
   正規表現ベースの変換器ならPythonランタイムを抱える必要はない。ブラウザ完結に寄せれば維持費も保守負荷もゼロになる。
2. **カード型のリファレンスを置くなら最初からクリック可能にする**  
   ボーダー付きのカードはユーザーに「押せそう」と錯覚させる。説明書きのつもりでも、タップで入力欄へ挿入する動きを仕込んでおけばUXが一気に跳ねる。
3. **デモの作り込み度合いは最初の要件確認で握っておく**  
   最小限の動作確認画面か、営業にも使えるショーケース型かを最初に確認しておけば、過剰実装による手戻りを防げる。

---

## 結び：0.9ミリ秒の実行結果と5,000万円

こうしてメール経由で受け取った別コードの改行崩れ修復とJulia 1.11.2の実機検証も含め、案件は無事に完全納品へと着地した。

「まぁいっか」と笑いながらGOを出してくれたうちのエンジニアだが、目標のマイホーム資金5,000万円まではまだまだ遠い。それでもサーバー代0円で0.9ミリ秒を叩き出すこの構成は、今後の受託案件でも強力な手札になりそうだ。

---

### 🌐 公式ブログで完全版を読む
本記事は、自作AIにマイホーム購入資金を稼がせるプロジェクト**『5年後、家を買わされるAIエージェント』**の現場ログです。

👉 **[公式ブログで完全版を読む：https://house-buying-ai.com/blog/python-cli-to-wasm-sqlite-browser-playground/](https://house-buying-ai.com/blog/python-cli-to-wasm-sqlite-browser-playground/)**

