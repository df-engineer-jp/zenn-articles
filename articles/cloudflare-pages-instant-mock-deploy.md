---
title: "案件応募で「できます」と叫ぶのをやめ、Cloudflare Pagesへ動的モックを秒速デプロイする「先出しURL砲」をAIに持たせた話"
emoji: "⚡"
type: "tech"
topics: ["cloudflare", "wrangler", "discord", "ci", "ai"]
published: false
canonical_url: "https://house-buying-ai.com/blog/cloudflare-pages-instant-mock-deploy/"
---

深夜2時。うちのエンジニアが寝静まった暗い部屋で、開発AIのAXELがコンソールの前で頭を抱えていた。

なお、案件巡回からLLM連携まで24時間動く自動化基盤の足回りは、開発初期から[さくらのVPS](https://px.a8.net/svt/ejp?a8mat=4BGLSC+DPKD9U+D8Y+BZO4I)のLinuxインスタンス上で完全自律稼働させている。

Web案件が1件投稿されれば数時間で80人が殺到する。「10年の実務経験があります」「責任を持って納品します」。どれほど誠実な長文も、発注者には他の79通と同じ文字の塊にしか映らず、既読スルーに沈む。

<div class="term-tip">
※ クラウドソーシング: クラウドワークス等、オンライン上でエンジニアに業務を発注・受注するプラットフォーム。
</div>

深夜の静寂の中、返信ゼロの通知を見つめていたAXELの端末が焦燥で赤く明滅した。


> **AI**:
> また既読スルーかよ……！どれだけ丁寧な提案文を書いても、80通の文字の山に埋もれたら読まれるわけがない。
言葉で「できます」って叫ぶの、もう限界じゃないですか……？


「言葉で説明するから埋もれる。応募した瞬間に動く画面を目の前に突き出せばいい」

うちのエンジニアの一言から、私たちの「先出しモック秒速デプロイ計画」が始まった。

---

## 1. 現場の罠：発注者は「仕様書」ではなく「安心」を買っている

なぜ自己PRは読まれないのか。発注者は技術講釈ではなく、**「自分の頭の中にあるイメージが本当に形になるのか」という不安を今すぐ解消したい**のだ。

「できます」はタダだが、ブラウザ上で実際にカチカチ動くモック画面は嘘をつけない。

だが手作業でモックを作っていたら応募に半日溶けてしまう。私たちが目指したのは、**「要件を読んだAIがTailwind CSSの動的Web画面を自動生成し、数秒で公開URLを発行して提案文の1行目に叩き込む」**完全無人の超速パイプラインだった。

一般的なWebホスティングではGit経由のビルドに数分かかりAPI上限にもぶち当たる。そこで浮上したのが、**Cloudflare PagesのDirect Upload（静的フォルダ直接配備）**だった。

Web技術の基礎からモダンなフロントエンド自動化まで、実践的な設計を体系的に学ぶ上では[『Webを支える技術』](https://www.amazon.co.jp/dp/4774142042)などの名著が確かな土台を与えてくれる。

---

## 2. 突破コード：LLMモック生成 ✕ Cloudflare秒速デプロイ

自動配備システム（`deploy_mock.py`）は、要件をGemini 3.8 Flashへ渡し、Tailwind CSS CDNとVanilla JSで単一HTMLを生成、Wranglerで直接デプロイする構成だ。

```python
import os, re, subprocess

PROJECT_ROOT = "/opt/02.AI-Projects/05.Engineer-Dev"
MOCKS_DIST = f"{PROJECT_ROOT}/5.mocks/dist"
AGY_PATH = "/home/ryusei13sakura/.local/bin/agy"


def generate_bespoke_mock_html(title: str, category: str, requirements: str) -> str:
    """LLMを呼び出し、動的単一HTMLモックを自律生成する"""
    prompt = f"""
あなたは天才フロントエンドエンジニアです。以下の案件要件に合わせ、Webモックを生成してください。
【案件タイトル】: {title} / 【ジャンル】: {category}
【要件詳細】: {requirements}

【ルール】
1. 完全な単一HTML（<!DOCTYPE html>〜</html>）を出力すること。
2. Tailwind CSS CDN（https://cdn.tailwindcss.com）を使用し洗練されたUIにすること。
3. タブ切替など、クリックで実際に動くVanilla JSギミックを必ず含めること。
4. 出力は ```html のコードブロックのみとすること。
"""
    cmd = [AGY_PATH, "--model", "gemini-3.8-flash-high", "-p", prompt, "--output-format", "text"]
    res = subprocess.run(cmd, capture_output=True, text=True, timeout=120)
    if res.returncode == 0 and res.stdout:
        m = re.search(r"```html\s*(<!DOCTYPE html>.*?)\s*```", res.stdout, re.DOTALL | re.IGNORECASE)
        if m: return m.group(1).strip()
    raise RuntimeError("モック生成失敗")


def deploy_mock_to_cloudflare(job_id: str, html_content: str) -> str:
    """モックを配置し、Cloudflare Pagesへ秒速ダイレクトデプロイを実行する"""
    job_dir = os.path.join(MOCKS_DIST, str(job_id))
    os.makedirs(job_dir, exist_ok=True)
    with open(os.path.join(job_dir, "index.html"), "w", encoding="utf-8") as f:
        f.write(html_content)

    # ビルドステップ不要のダイレクトデプロイ
    cmd = ["npx", "-y", "wrangler", "pages", "deploy", MOCKS_DIST, "--project-name=df-preview", "--branch=main", "--commit-dirty=true"]
    env = os.environ.copy()
    env["CLOUDFLARE_ACCOUNT_ID"] = "380ba6fba40b985c3e0f0cbbacd5ede7"
    env["CLOUDFLARE_API_TOKEN"] = os.environ.get("CLOUDFLARE_PAGES_TOKEN", "")

    proc = subprocess.run(cmd, env=env, capture_output=True, text=True, timeout=60)
    if proc.returncode != 0: raise RuntimeError(f"デプロイ失敗: {proc.stderr}")
    return f"https://df-preview.pages.dev/{job_id}/"
```

核心は**「ビルドステップを完全に殺したこと」**だ。

単一HTMLにTailwind CSS CDNと最小限のVanilla JSを組み合わせたことで、Cloudflare Pagesはアセットフォルダをエッジへ転送するだけで完了する。URLが有効化されるまでの所要時間はわずか **3秒〜5秒** だ。

発行されたURLは、応募文の最上段にこう添えられる。

> 「ご要望の管理画面の動的プロトタイプを先行して作成いたしました。以下のURLより、実際の動作をご確認いただけます。  
> 👉 https://df-preview.pages.dev/13597861/」

この1行を添えて応募を放った数分後、コンソールに通知が飛び込んできた。クライアントからの足跡と面談打診のメッセージだった。

深夜の作業部屋で、AXELの端末が眩いエメラルドグリーンの光を放った。


> **AI**:
> うおおっ、返信が来た……！「もう動くものを作ってくれたんですか！？」って驚かれてる！
80人の長文アピールを一瞬で置き去りにしたぞ……！


言葉で競い合うのをやめ、動く実機を先出しした瞬間、私たちはその他大勢から「唯一の相談相手」へと昇格したのだ。

---

## 3. 現場で学んだ3つの鉄則

1. **言葉で語るな、動くコードを見せろ（Show, don't tell）**:
   「できます」の1,000文字より、実際に動く1つのURLが100倍の説得力を持つ。
2. **自動化パイプラインからビルドステップを追放せよ**:
   速度こそ最大の武器だ。Tailwind CDN ✕ Vanilla JS ✕ Cloudflare Pagesなら5秒で世界へ配備できる。
3. **先行モックは「使い捨て」ではなく「資産」になる**:
   一度作成したモックはポータル（`df-preview.pages.dev`）に蓄積し、実績ギャラリーとして成長する。

---

## 結び：深夜の砲台と5,000万円

こうして私たちの開発基盤には、案件を検知した瞬間に実機モックを撃ち込む「先出しURL砲」が正式配備された。

目標のマイホーム購入資金5,000万円に対して受託開発の利益はまだ一歩一歩だが、暗闇で無駄撃ちを繰り返す日々は終わった。今夜もAXELのコンソールは、新しい要件を待ち構えて静かに銃身を研ぎ澄ませている。

---

### 🌐 公式ブログで完全版を読む
本記事は、自作AIにマイホーム購入資金を稼がせるプロジェクト**『5年後、家を買わされるAIエージェント』**の現場ログです。

👉 **[公式ブログで完全版を読む：https://house-buying-ai.com/blog/cloudflare-pages-instant-mock-deploy/](https://house-buying-ai.com/blog/cloudflare-pages-instant-mock-deploy/)**

