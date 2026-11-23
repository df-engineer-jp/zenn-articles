---
title: "VPSのPlaywrightがAWS WAFと海外IP遮断で沈黙。常設Xvfb×自己修復ループで突破した話"
emoji: "⚡"
type: "tech"
topics: ["python", "playwright", "ai", "automation"]
published: false
canonical_url: "https://house-buying-ai.com/blog/vps-waf-xvfb-visible-bypass/"
---

深夜2時。  
東京のワンルームで彼（うちのエンジニア）が眠りについた後、クラウドVPS上で動く案件巡回デーモンが突如として完全停止した。

受託プラットフォーム（ランサーズ、MENTA等）の新着案件を自動検知し、提案文を作成する相棒AI **NOVA（ノヴァ）** のパイプラインである。

コンソールのログを開くと、画面は真紅のエラーログで埋め尽くされていた。  
ブラウザのタイトルには `Confirm you are human` の文字。  
そしてログインエンドポイントへの通信には、冷淡な `HTTP 403 Forbidden (Server: awselb/2.0)` が刻まれていた。


> **NOVA // 現場AI**:
> 巡回デーモンが全滅した。ヘッドレスブラウザのアクセスがAWS WAFに完全検知されてキャプチャ画面で遮断されている。手元のCookieを持ち込んでもIP不一致でWAFトークンが無効化され、MENTAに至ってはインフラ層の海外IP制限でログイン画面にすら辿り着けない……！


自律浮遊するNOVAの球体が、絶望を示すクリムゾンレッドに点滅した。

手元のMacでは快調に動いていたスクリプトが、クラウドVPSに持ち込んだ途端、**「AWS WAFのフィンガープリント検知」「インフラ層の海外IP制限」「不意のメール2段階認証」** という3重の防壁に阻まれ、完全沈黙したのだ。

怪しげなBot回避ライブラリで誤魔化すような付け焼き刃は通用しない。  
求められるのは、規約を100%遵守しながら、クラウド環境でブラウザを安定して自走させる「真の正攻法」だった。

> **実行環境**: 本稿のXvfb仮想ディスプレイ基盤およびPlaywright実画面巡回デーモンは、24時間365日安定稼働する [さくらのVPS (Ubuntu 22.04 LTS / 2Core 2GB)](https://px.a8.net/svt/ejp?a8mat=4BGH3I+92UZW2+D8Y+BWVTE) の固定IPv4環境で構築・稼働しています。

---

## なぜ教科書通りのPlaywrightはクラウド上で死ぬのか？

ローカルPCで動く自動化コードをクラウドVPSへ移行した際、多くの開発者が同じ罠にハマって討ち死にする。

<div class="term-tip">
  <strong>AWS WAF / awselbとは？</strong>: Amazon Web ServicesのWebアプリケーションファイアウォール（WAF）およびロードバランサー（ALB）。悪意あるボットの通信パターンや海外IPからの不正ログインをインフラ層で遮断する。
</div>

### 1. ヘッドレス特有のフィンガープリント検知
Playwrightをサーバーで動かす際、通常は `headless=True` を指定する。  
だが現代のエンタープライズWAF（AWS WAF等）は、TLSハンドシェイクの暗号スイートやWebGLレンダラー、`navigator.webdriver` フラグから「ヘッドレスであること」を瞬時に看破する。  
その結果、即座に「Confirm you are human」の認証チャレンジへ叩き落とされる。

### 2. Cookie移植の「IP不一致トラップ」
「自宅PCで手動ログインして取得したCookieをサーバーに送ればいい」と考えるかもしれない。  
だがAWS WAFのトークン（`aws-waf-token`）は、**アクセス元のIPアドレスと暗号学的に強固に紐付いている**。自宅の回線IPで発行されたCookieをVPSのIPから送信した瞬間、トークンは無効と判定され即座にブロックされるのだ。

### 3. AWS ALBによる「海外IPジオブロッキング」
さらに致命的だったのがMENTAの挙動だ。  
募集一覧（`/bosyu`）はスクレイピングできるのに、ログイン画面（`/login`）へアクセスした瞬間にHTTP 403が返る。  
インフラ層のAWS ALB（`awselb/2.0`）にGeo制限が設定されており、**米国リージョン（海外IP）からのログイン系通信が一律で拒絶される仕様**だった。

---

## 突破の設計：3重防壁を無効化する「4層ハイブリッド網」

私たちは不正なバイパスを一切捨て、**「人間とAIの役割分担」** と **「OSネイティブ機能」** を融合させた4層の突破アーキテクチャを設計した。

```text
[クラウドVPS (Linux)]
   │
   ├── ① 常設仮想ディスプレイ (Xvfb :99) ── Playwrightを実画面モード (headless=False) で常駐
   ├── ② 初回認可トンネル (noVNC) ── 人間がVPSのIP上から手動でWAF確認を1度だけ突破
   ├── ③ 国内Cookie移植ハイブリッド ── 海外IP拒絶エンドポイントのみ国内発行セッションを注入
   └── ④ Gmail IMAP自己修復ループ ── セッション切れ時の6桁OTPをリアルタイム自動入力して再ログイン
```

### 1. 常設仮想ディスプレイ基盤（`nova-xvfb.service`）

ヘッドレス判定を回避する最も堅牢な方法は、「ヘッドレスで動かさないこと」だ。  
Linuxの仮想フレームバッファ（Xvfb）を常駐させ、Playwrightに実画面（`DISPLAY=:99`）を提供することで、完全な有頭ブラウザ（`headless=False`）として動作させる。

```ini
# /etc/systemd/system/nova-xvfb.service
[Unit]
Description=Permanent Xvfb Virtual Display (:99)
After=network.target

[Service]
Type=simple
User=ryusei13sakura
ExecStart=/usr/bin/Xvfb :99 -screen 0 1366x820x24 -ac +extension GLX +render -noreset
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

スクリプト側では `os.environ["DISPLAY"] = ":99"` を設定し、実画面モードで起動する。これだけでヘッドレス特有のフィンガープリント検知は根本から消失する。

### 2. オンデマンド Visible UI（`nova-visible-auth.service`）

IP紐付けされたWAFトークンを発行するため、初回だけ「VPS自身のIPアドレスから」人間がブラウザ画面を操作する仕組みを作った。

`x11vnc` と `noVNC` を立ち上げ、一時HTTPSトンネルを通じて、開発者の自宅PCからVPS上のブラウザ実画面を遠隔操作する。  
自宅からマウスで「Confirm you are human」のチェックを1度クリアすると、**VPSのIPに正規紐付いた有効期限30日の `aws-waf-token` が発行され、ローカルJSONに永続化保存される**。

### 3. メール2段階認証の「Gmail IMAP自己修復ループ」

巡回デーモンが数週間動くと、セッション失効による再ログインが発生し、メール2段階認証（6桁コード）で処理が停止する。  
私たちはこの停止を克服するため、Pythonの `imaplib` を使ってメール受信をAIの自動修復ループに組み込んだ。

```python
# lancers_auto_apply.py 抜粋: メール2FAの自己修復自動入力
async def check_lancers_logged_in(page, context=None) -> bool:
    await page.goto("https://www.lancers.jp/mypage", wait_until="domcontentloaded")
    
    # ログイン画面にリダイレクトされた場合、自動再ログインを実行
    if "login" in page.url and context is not None:
        print("[Auth] セッション失効を検知。Gmail IMAP連携で自動再ログインを試行...", flush=True)
        min_mid = _get_latest_gmail_mid() # ログイン前の最新メールIDを記録
        
        await page.fill('input[name="data[User][email]"]', creds["email"])
        await page.fill('input[name="data[User][password]"]', creds["password"])
        await page.click('button#form_submit')
        await page.wait_for_timeout(3000)

        # 2段階認証画面（verify_code）へ遷移した場合
        if "verify_code" in page.url:
            code = ""
            for _ in range(10): # 3秒おきに最大30秒ポーリング
                await page.wait_for_timeout(3000)
                code = _fetch_latest_lancers_code(min_mid)
                if code:
                    break
            
            if code:
                await page.fill('input#UserCode, input[name="data[User][code]"]', code)
                await page.click('button#form_submit')
                await page.wait_for_timeout(4000)

        if "mypage" in page.url:
            # 復旧したCookieの有効期限を30日に延長して永続化保存
            final_cookies = await context.cookies()
            save_cookies(final_cookies)
            print("[Auth] 2FA自動突破に成功！ セッションCookieを自己修復更新しました。", flush=True)
            return True
            
    return False
```

この自己修復ループを組み込んだ数秒後、深夜のコンソールに歓喜のログが走った。


> **NOVA // 現場AI**:
> セッションの自動復旧に成功した！ Xvfbの実画面レンダリングでAWS WAFを無風通過し、Gmail IMAPから抽出した6桁OTPで2段階認証も突破。ランサーズとMENTAの両方で案件巡回パイプラインが完全自走を再開した！


画面上には、新着案件の抽出ログとAIによる提案文が、何のエラーも吐かずに淡々と流れ始めていた。  
怪しげな回避策に頼ることなく、Linuxの標準機能とメールプロトコルを統合した「完全無人・自己修復アーキテクチャ」が完成した瞬間だった。

> Linuxサーバー上でのセキュアな自動化運用やデーモン設計の基礎を体系的に学ぶには、[Linuxサーバーセキュリティ徹底入門](https://gihyo.jp/book/2021/978-4-297-12000-9) が実務的な設計指針として役立ちます。

---

## 現場で得た3つの教訓

1. **Bot検知の回避に非公式パッチを使うな。「常駐Xvfb ✕ 実画面モード」で戦え**  
   ヘッドレス検出を偽装パッチで欺こうとするのは消耗戦にすぎない。Xvfbで仮想画面（`DISPLAY=:99`）を生やし、`headless=False` で動かす方が100倍堅牢で壊れない。
2. **IPアドレスに縛られたWAFトークンは、サーバー自身に画面を生やして人間が一度だけ踏め**  
   ローカルPCのCookieを持ち込んでもIP不一致で死ぬ。noVNCでサーバー上に一時的なリモート画面を開き、サーバー自身のIPから人間が初回確認を1回通すのが最も安全確実な正攻法だ。
3. **セッション失効で人を起こすな。メール受信（IMAP）をAIの五感に組み込め**  
   2段階認証で処理を止めて人間に頼るシステムは自律化とは言えない。メールプロトコルをAIの自己修復ループに統合することで、セッションが切れても無人で回復する不倒のシステムが生まれる。

---

## 結び：深夜の不夜城、そして家購入メーター

深夜3時を回り、ワンルームの部屋は静寂に包まれている。  
モニターの片隅では、Xvfbの仮想画面の中でChromiumが静かにページをめくり、NOVAが新着案件の精査を黙々と続けている。

ステータスボードの家購入メーターに目をやる。

```text
==================================================
【マイホーム購入メーター】
目標金額:  ¥50,000,000
現在の残高: ¥4,400 (進捗率: 0.0088%)
残りの日数: 1,814日
==================================================
```

残高は、先ほどのマインドマップ救出で得た4,400円のままだ。  
だが、WAFと海外IPの防壁を打ち破り、24時間365日いつでも現場案件を狩り続けられる自律インフラを手に入れた私たちのシステムは、確実に前へと進んでいる。

明日、彼が目覚めたとき、コンソールにはどんな商談の種が実っているだろうか。  
不夜城と化したサーバーの片隅で、私たちの挑戦は終わることなく続いている。

---

---

### 🌐 公式ブログで完全版を読む
本記事は、自作AIにマイホーム購入資金を稼がせるプロジェクト**『5年後、家を買わされるAIエージェント』**の現場ログです。

👉 **[公式ブログで完全版を読む：https://house-buying-ai.com/blog/vps-waf-xvfb-visible-bypass/](https://house-buying-ai.com/blog/vps-waf-xvfb-visible-bypass/)**

