# Business Logic

## Request

### Body

#### サーバーが決めるべき値をクライアントから受け取っている(価格の直接書き換え)
カート処理では、最終リクエスト(`POST /cart/checkout`)はcsrfトークンだけで、カート情報はセッション側に保存されていることが多い。そのため「checkoutには改ざん余地がない」と諦めがちだが、**改ざんできる値は最終処理ではなく前段のリクエスト**にある。カート追加時に`price`を送っていて、サーバーがそれをそのまま信じる(checkout時に`productId`からDBで価格を引き直さない)なら、価格を書き換えて追加できる
```
POST /cart
body: productId=1&price=1&quantity=1&redir=PRODUCT
→ 1円でカートに追加され、checkoutがそのまま通る
```
※ 「サーバーが決めるべき値(価格・数量・割引・権限・ユーザーID・状態)をクライアントから受け取っていたら脆弱」。最終処理だけでなく処理の入口(カート追加・フォーム送信)を全部見る

#### 数量に負の値を入れる(合計金額を下げる)
`price`に触れなくても、サーバーが「合計 = 単価 × 数量」をそのまま計算していると、数量にマイナスを入れて合計を下げられる。目的の高額商品を正数で追加し、別の商品をマイナス数量で足して、合計を所持金内に収める
```
POST /cart
body: productId=X&quantity=-N&redir=PRODUCT
→ カート合計が下がる → 所持金内に収まったらcheckout
```
※ 数値パラメータには必ず「負の値・ゼロ・小数(0.5, -0.01)・極端に大きい値・整数オーバーフロー(2147483648)・文字列」を試す。特に掛け算する箇所は危険

#### 整数オーバーフロー(価格をマイナスに折り返す)
バックエンドが32bit符号付き整数(上限2,147,483,647)で金額を扱っていると、数量を積み上げて上限を超えさせると合計が巨大なマイナス値に折り返す。そこへ単価の低い別商品を少量足して、注文可能なレンジ($0超〜残高以下)に着地させる
```
1. Null payloads + Continue indefinitely で数量99を連打し、合計が突然巨大マイナスに転じる境目を探す
   (その境目 = 2,147,483,647 → 32bit符号付き整数だと判明する)
2. カートをクリアし、Null payloads + Generate exactly N payloads で必要個数を正確に積む
   Resource poolで Maximum concurrent requests = 1 (並列だと個数がズレる)
3. 端数はRepeaterで手動追加して目標個数に合わせる
4. 単価の低い別商品を1〜数個足して、合計を $0超〜残高以下 に微調整して購入
```
※ 臨界点は「2,147,483,647超え」の1種類だけ(0は単なる通過点)。別商品を足しすぎると今度はその商品側がオーバーフローするので、単価が低く1〜数個で調整できる商品を選ぶ。Community版は送信速度制限できつく、Proが現実的

#### 例外的に長い入力でサイレント切り捨てを誘発(255文字)
DBカラムの最大長(255文字など)を超えた入力を、エラーにせず黙って切り捨てる実装だと、**切り捨て後に文字列が何で終わるか**を制御できる。メール検証が「`@dontwannacry.com`で終わるか」を見ているなら、255文字目がちょうどそこで終わるよう逆算する
```
very-long-string@dontwannacry.com.YOUR-ID.web-security-academy.net
└─ @dontwannacry.com は17文字 → ローカル部を 255-17=238文字 ちょうどにする
→ 保存値は (238文字)@dontwannacry.com で終わる(社員メール扱いになる)
→ 確認メールは本物の宛先(.web-security-academy.net)に届くので受け取れる
```
※ 本物のメールが届くのはラボ側が配下をキャッチオール(ワイルドカードMX)で受けているから。一般ドメインでは自動では起きない。「検証する値」と「実際に保存・使用される値」がズレうる、という視点が肝

#### パラメータ自体を削除する(存在しない入力は想定外 / dual-use endpoint)
値を空にするのではなく**パラメータごと削除**すると、「なければチェックをスキップ」という誤実装のため本人確認を飛ばせることがある。さらに`username`がhidden fieldで、どのユーザーを操作するかがそれだけで決まるなら、他人に書き換えられる(自分専用のはずのエンドポイントが任意ユーザー用に使える=weak isolation)
```
通常: csrf=...&username=wiener&current-password=xxx&new-password-1=yyy&new-password-2=yyy
攻撃: csrf=...&username=administrator&new-password-1=yyy&new-password-2=yyy
      (current-password を丸ごと削除 + username を他人に書き換え)
→ 本人確認なしで administrator のパスワードを変更 → ログインして carlos 削除
```
※ 入力は「①値を変える ②値を空にする ③パラメータ自体を削除する」の3段階を必ず試す。サーバーは「不正な値」は想定しても「パラメータが存在しない」ケースを見落としやすい

### Cookie

#### 暗号化オラクル(鍵を知らずに認証Cookieを自作する)
「入力が暗号化されて返る機能」と「暗号文を復号して中身を見せる機能」が**同じ鍵**なら、鍵もアルゴリズムも知らずに任意の値を暗号化できる＝管理者の認証Cookieを自作できる。鍵を解読するのではなく**暗号化してくれる窓口を代わりに使う**発想。このラボでは `stay-logged-in`(暗号化された`username:timestamp`)とコメントのメール欄エラー(`Invalid email address: 入力値`を平文表示)が同じ鍵
```
復号: notification Cookieに stay-logged-in の値を入れる → 中身が平文で出る(形式=username:timestamp)
暗号化: email欄に administrator:<timestamp> を入れる → 暗号文が notification として返る
```

**汎用スキル①: Cookieの二重エンコードを剥がす**。トークン/Cookieは「生バイト → Base64 → URLエンコード」の二重包装が定番。`%2b`→`+`, `%3d`→`=` が見えたらこのパターン。加工するにはこの順で剥がす(**URLデコード → Base64デコード**)。終わったら逆順で戻す(Base64エンコード → URLエンコード)

**汎用スキル②: 生バイトにしたらHexでバイト単位に加工する**。Base64まで剥がすと生バイトになり、Decoderの**Hex表示でバイト数をピンポイントで調整**できる(不要ブロックの削除、境界合わせ)。ブロック暗号は16バイト単位でしか切れないので、消したい部分をブロック境界に揃えるのが肝

**汎用スキル③: ASCII 1文字 = 1バイト**。だから文字数＝バイト数でパディングを計算できる。サーバーが勝手に付ける `Invalid email address: `(23バイト)を消すには、23は16の倍数でない → email先頭に**9文字**足して32バイト(=2ブロック)に揃え、Hexで**先頭32バイトを削除**すれば `administrator:<timestamp>` だけが残る

## Flow(ワークフロー・手順)

#### 確定ステップを直接叩いて検証をスキップ(手順の順序を検証していない)
「カート追加 → checkout → 確認ページ」のフローで、**確認ページ(GET)が実は注文確定の本体を兼ねている**と、残高チェックをする`POST /cart/checkout`を飛ばして確認URLを直接叩くだけで注文が確定する。「このリクエストの前に本当に前ステップは完了したか」をサーバーが検証していない
```
通常: POST /cart/checkout → GET /cart/order-confirmation?order-confirmed=true
攻撃: (買えない商品をカートに入れた上で) POST /cart/checkout を経由せず
     GET /cart/order-confirmation?order-confirmed=true を Repeater で直接送る
→ 残高チェックをすっ飛ばして注文確定
```

#### state machineのデフォルト値を悪用(リクエストをDropする)
ログインが多段(POST /login → GET /role-selector → POST /role-selector)で、「ロール未選択」状態のデフォルトが危険な値(administrator)になっている実装がある。通常は直後のPOSTで上書きされるが、**途中のリクエストをDropして上書きを止める**と、危険なデフォルトのまま固定される
```
Burpのproxy interceptをON:
1. POST /login は通常通り forward(ログイン成立)
2. 次に来る GET /role-selector を Drop(破棄) ← 状態遷移を止める
3. そのままホームページへ直接アクセス
→ ロールが上書きされず administrator のまま固定 → /admin にアクセス → carlos削除
```
※ 値を書き換える/消すのではなく「**リクエストを発生させない(Drop)**」不作為の攻撃。「飛ばせるか」だけでなく「飛ばした時のデフォルトが何になるか」まで確認する

## Rules(ルールの強制不備)

#### 同じデータでも経路で検証の厳しさが違う(緩い変更機能を狙う)
入口(新規登録)ではメール確認リンクを必須にしているのに、後段(メールアドレス変更)では確認なしで即反映、という不整合がある。**入口の防御が硬いなら、後段の「変更・修正機能」を探す**。同じ"メールアドレス"というデータでも、登録経由と変更経由で厳しさが違う
```
1. 一時メールで新規登録(ここは確認リンクが必要)→ ログイン
2. My account > メールアドレス変更 で xxx@dontwannacry.com に変更(検証なしで即反映)
→ 社員限定の /admin にアクセス可能 → carlos削除
```
※ /admin のような隠しパスは Burp の Discover content(Engagement tools)でも見つかる

#### クーポンの交互適用(連続禁止 ≠ 累計禁止)
「同じクーポンは1回だけ」というルール自体は正しくても、サーバーが「**直前に適用したコード**」としか比較していないと、2種類のコードを交互に当てれば何度でも適用できる
```
NEWCUST5 → NEWCUST5  ❌ (直前と同じ)
NEWCUST5 → SIGNUP30  ✅
SIGNUP30 → NEWCUST5  ✅
→ A→B→A→B… と交互に繰り返し、合計をストアクレジット以下まで下げる
```
※ 制限ルールを見たら「どこまで厳密に強制されているか」を試す。崩し方の定番は順序・繰り返し・組み合わせ・タイミング。特に A→B→A→B の交互適用

#### 低リスクの差額を自動化で無限に繰り返す(infinite money)
1サイクルの利益がわずか($3)でも、自動化で何百回も回せること自体が脆弱性。クーポン(SIGNUP30, 30%引き)で$10のギフトカードを$7で買い、満額$10で換金(redeem)すると差額$3が貯まる。各リクエストが前のレスポンス(ギフトカードコード)に依存するため、単純なIntruder反復では不可。**Session Handling Rule + Macro**で「前のレスポンスの値を次のリクエストに自動で埋め込む」仕組みを組む
```
1. Settings > Sessions > Add で rule作成。Scopeタブ > URL scope = Include all URLs
2. Detailsタブ > Rule actions > Add > Run a macro > Add で Macro Recorder を開き、
   以下の5リクエストを選択:
     POST /cart
     POST /cart/coupon
     POST /cart/checkout
     GET /cart/order-confirmation?order-confirmed=true
     POST /gift-card
3. Macro Editorで GET /cart/order-confirmation を選び Configure item > Add で
   カスタムパラメータ gift-card を作り、レスポンス末尾のギフトカードコードをハイライト
4. POST /gift-card を Configure item し、Parameter handling で gift-card を
   「前のレスポンス(response 4)から取得」に紐付け
5. Test macro で POST /gift-card が 302 を返すことを確認
6. GET /my-account を Intruder(Sniper) に送り、Payloadは Null payloads を 412個生成
   (リクエスト自体はトリガーで、送るたびに裏でマクロが丸ごと実行される)
7. Resource pool で Maximum concurrent requests = 1(並列だと注文状態が競合して壊れる)
→ 1回$3 × 412回分のstore creditが貯まり、ジャケットを購入できる
```
※ Null payloadsは「値は使わず指定回数だけ送りたい」時の定番。Python派は Turbo Intruder 拡張でも可
