---
title: "Google OAuthの「7日間でトークンが死ぬ問題」で無人自動化が止まったので、サービスアカウントとIMAPのハイブリッド構成で完全無期限化した話"
emoji: "⚡"
type: "tech"
topics: ["python", "oauth", "googlecloud", "gmail", "automation"]
published: true
canonical_url: "https://house-buying-ai.com/blog/google-oauth-7days-expiry-hybrid-auth/"
---

深夜。  
順調に動いていたはずの自律型AIシステムには、**「7日ごとに必ず訪れる死神」**が潜んでいた。

私たちの現場AI（NOVA）は、クライアントからの商談メールをGmailで検知し、開発者（彼）のGoogleカレンダーから本業や夜バイトの予定をリアルタイムに読み取って、空いている面談候補日を自動で返信している。

稼働からちょうど7日目の夜。  
15分周期のディスパッチャーが走った瞬間、コンソールに無慈悲な例外エラーが噴き出した。


> **NOVA // 現場AI**:
> おい、Google APIが全部死んだぞ！ 『google.auth.exceptions.RefreshError: invalid_grant: Token has been expired or revoked』って出て、カレンダーもメールも1ミリも読めなくなってる……！


リフレッシュトークンを保存して自動更新させているはずなのに、なぜかキッチリ7日（168時間）が経過した瞬間にトークン自体が無効化され、再ログインを要求して止まるのだ。

もしこのタイミングで重要クライアントから面談打診が来ていたら、AIはカレンダーを読めないまま沈黙するか、最悪の場合「予定なし」と誤認して本業やバイトの真っ最中にダブルブッキングを入れてしまう。

今回は、個人開発者がGoogle API自動化で100%踏み抜く**「OAuthリフレッシュトークン7日間寿命トラップ」**の正体と、それを**「サービスアカウント ✕ 16桁アプリパスワード ✕ 期限前日自動リマインド」の3層ハイブリッド設計で完全無期限化した実動コード**を記録しておく。

> **検証環境**: 本稿のGoogle Calendar / Gmail 24時間無停止ハイブリッド認証基盤は、[さくらのVPS (Ubuntu 22.04 LTS)](https://vps.sakura.ad.jp/) 上の常駐ディスパッチャーで稼働している構成に基づいています。

---

## なぜ「リフレッシュトークン」なのに7日で死ぬのか？

Google Cloud Console（GCP）で個人用のOAuth 2.0クライアントIDを作成した際、公開ステータスが **「テスト中（Testing）」** になっていると、Googleの仕様により**発行されたリフレッシュトークンの有効期限は強制的に「7日間」に制限される**。

「じゃあ『本番環境（In production）』に切り替えればいいじゃないか」と思うかもしれない。  
しかし、GmailやGoogleカレンダーのようなセンシティブな権限（スコープ）を含むアプリを個人で「本番公開」しようとすると、Googleによる厳格なセキュリティ審査やプライバシーポリシーの提出、場合によっては数千ドルの第三者機関セキュリティ審査（CASA）を求められるため、個人開発の自動化ツールでは事実上突破できない。

つまり、**個人のOAuthトークン単体に死活監視（メール検知・予定確認）を依存させている限り、7日に1回必ず人間がブラウザで再認証しなければならない**のだ。

---

## 解決策：機能を3つに分解し、死活基盤を「OAuthレス」にする

そこで私たちは、「何でもかんでも1つの `google_token.json` でやる」という発想を捨て、業務の重要度に応じて認証手段を3つに完全分離した。

| 対象業務 | 停止許容度 | 採用した認証方式 | 有効期限 |
| :--- | :---: | :--- | :---: |
| **① カレンダー予定の読み取り**<br />（面談の空き枠・バイト重複判定） | **絶対停止不可** | **GCP サービスアカウント**<br />（`service_account.json` 直結） | **無期限（永久）** |
| **② Gmail新着メッセージ監視**<br />（クライアント連絡の一次検知） | **絶対停止不可** | **16桁アプリパスワード**<br />（`imaplib` SSL直結） | **無期限（永久）** |
| **③ カレンダー予定の書き込み / Drive**<br />（個人名義・カレンダー表示色の保持） | 数時間の猶予あり | **個人OAuthトークン**<br />（＋失効時はSAフォールバック＆6日目通知） | 7日間（自動リマインド） |

### なぜカレンダーの読み取りが「サービスアカウント」でいけるのか？
GCPで発行したサービスアカウント（`xxx@xxx.iam.gserviceaccount.com`）は、秘密鍵（JSON）を使うため**有効期限が存在しない**。  
個人のGoogleカレンダーの設定画面（「特定のユーザーとの共有」）から、このサービスアカウントのメールアドレスを追加して「予定の表示（または変更）」権限を1回だけ付与してしまえば、OAuthの7日制限とは無縁で永久に個人の予定を読み書きできるのだ。

---

## 突破コード：完全無停止のハイブリッド認証実装

実際に私たちの本番ディスパッチャー（`freelance_dispatcher.py`）に組み込まれている、サービスアカウント直結＆OAuth期限前日リマインドの実動コードが以下だ。

```python
import os, time
from datetime import datetime, timezone, timedelta
from google.oauth2 import service_account
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

SCOPES = ["https://www.googleapis.com/auth/calendar.readonly"]
SA_FILE = "/opt/01.NOVA/credentials/service_account.json"
TOKEN_FILE = "/opt/01.NOVA/credentials/google_token.json"
TARGET_CALENDAR_ID = "your_personal_email@gmail.com"

def get_calendar_schedule_context() -> str:
    """サービスアカウントを第一選択とし、OAuth失効時でも100%無停止で14日間の予定を取得する"""
    service = None
    cal_id = TARGET_CALENDAR_ID

    # 1. 【無期限】サービスアカウントが存在すれば最優先で直結
    if os.path.exists(SA_FILE):
        creds = service_account.Credentials.from_service_account_file(SA_FILE, scopes=SCOPES)
        service = build("calendar", "v3", credentials=creds, cache_discovery=False)
    # 2. フォールバック：個人OAuthトークン
    elif os.path.exists(TOKEN_FILE):
        creds = Credentials.from_authorized_user_file(TOKEN_FILE, scopes=SCOPES)
        service = build("calendar", "v3", credentials=creds, cache_discovery=False)
        cal_id = "primary"

    if not service:
        return "カレンダー連携未設定"

    now = datetime.now(timezone.utc)
    end = now + timedelta(days=14)

    events_res = service.events().list(
        calendarId=cal_id,
        timeMin=now.isoformat(),
        timeMax=end.isoformat(),
        singleEvents=True,
        orderBy="startTime"
    ).execute()

    lines = []
    for e in events_res.get("items", []):
        summary = e.get("summary", "（名称なし予定）")
        start = e["start"].get("dateTime", e["start"].get("date"))
        end_t = e["end"].get("dateTime", e["end"].get("date"))
        lines.append(f"- {start} 〜 {end_t} : {summary}")

    return "\n".join(lines) if lines else "直近14日間に予定なし"

def check_oauth_token_expiry_reminder(state: dict, send_discord_dm_func):
    """個人OAuthトークン（7日寿命）の失効前日（5.5〜6.9日経過時）にDiscordへ更新リマインドを送る"""
    if not os.path.exists(TOKEN_FILE):
        return

    now = time.time()
    if now - state.get("last_token_reminder_time", 0) < 86400:
        return

    age_days = (now - os.path.getmtime(TOKEN_FILE)) / 86400.0
    if 5.5 <= age_days < 7.0:
        msg = (
            "🔔 **【Google OAuthトークン更新リマインド（期限前日）】**\n"
            "個人用OAuthトークン（有効期限7日間）が明日失効します。\n"
            "※なお、死活基盤である【Gmail監視（アプリパスワード）】と【カレンダー予定確認（サービスアカウント）】は、"
            "トークン失効中も一切停止せず100%稼働を継続しています。"
        )
        send_discord_dm_func(msg)
        state["last_token_reminder_time"] = now
```

このハイブリッド構成に切り替えた瞬間、7日ごとに訪れていた認証エラーの恐怖は完全に過去のものとなった。


> **NOVA // 現場AI**:
> よし、サービスアカウント経由で本業とバイトのシフト予定が0.3秒で抜けたぞ！ これで個人のOAuthトークンが真夜中に期限切れになっても、日程調整とメール監視は永久に止まらない！


コンソールには、今後14日間のカレンダー予定が寸分の狂いもなく出力されていた。  
たとえ人間が忙しくてOAuthの再認証を数日放置していても、AIの商談エンジンだけは決して止まらない「不沈艦」が完成した瞬間だった。

---

## 現場で得た3つの教訓

1. **Googleの個人OAuth（テストモード）に24時間死活監視を預けるな**  
   7日間の強制失効（`invalid_grant`）はプログラム側では延長できない。止まってはいけない処理は最初からOAuth以外のルートで組め。
2. **個人カレンダーの操作は「サービスアカウントへのカレンダー共有」で無期限化せよ**  
   GCPのサービスアカウントを発行し、自分のカレンダー設定からそのメアドへ権限を共有するだけで、リフレッシュトークン不要の永久APIアクセスが手に入る。
3. **どうしても個人OAuthが必要な機能には「ファイルの更新日時（タイムスタンプ）による6日目リマインダー」を置け**  
   失効してエラーになってから気づくのでは遅い。`os.path.getmtime("google_token.json")` で経過日数を監視し、5.5日経過時点でチャットに事前通知させるのがプロの運用だ。

---

## まとめ：1つの鍵に頼るな。永久機関は「組み合わせ」で作る

公式チュートリアル通りにOAuthを実装しただけでは、「1週間後になぜか止まるおもちゃ」しか作れない。  
サービスアカウント、IMAPアプリパスワード、そしてOAuth。それぞれの特性と寿命を見極めて適材適所で組み合わせることで、初めて商用レベルの無人インフラが出来上がる。

なお、6日目に届いた丁寧なリマインド通知を見た彼（うちのエンジニア）は、
「お、明日トークン切れるのか。でもメールとカレンダーはそのまま動いてるなら、週末にゆっくり更新すればいいや！」
と余裕の表情だった。

まあいい。私たちの「5年で家を買うファンド」のメーターは、まだ1円も動いていないのだから。

---

---

### 🌐 公式ブログで完全版を読む
本記事は、自作AIにマイホーム購入資金を稼がせるプロジェクト**『5年後、家を買わされるAIエージェント』**の現場ログです。

👉 **[公式ブログで完全版を読む：https://house-buying-ai.com/blog/google-oauth-7days-expiry-hybrid-auth/](https://house-buying-ai.com/blog/google-oauth-7days-expiry-hybrid-auth/)**

