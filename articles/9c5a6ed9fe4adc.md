---
title: "RailsでJWT認証を実装する"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ruby, rails, JWT]
published: false
---

## JWTとは

**J**son **W**eb **T**okenの略。
JSONデータをBase64URLでエンコードしたもの。
ちなみに読み方は「ジョット」です。

例えば、以下のようなJSONデータを、
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

BASE64URLでエンコードすると、以下のようになります。

```jwt
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ
```

JWTはデコードすることも簡単にできるため、機密情報を含めるべきではありません。

## JWSとは

ヘッダー、ペイロード（JWT）、署名の3つの要素を`.`で繋いだ構成になっています。

以下のような方法で生成されます。

```jwt
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
cThIIoDvwdueQB468K5xDc5633seEFoqwxjF_xSJyQQ
```

![JWS](/images/JWS.png)

JWSもデコード自体は誰でもできます。JWT部分は誰でも見ることができます。
しかし署名部分は、暗号鍵がなければ元の文字列に戻せません。

つまり、誰かがJWSを改ざんして使おうとしても、署名部分を暗号鍵で復号化したときに、もとの文字列と一致しなくなるため、使用する前にシステム側で弾くことができます。

## RailsのAPIモードでJWT認証を実装

### ユーザー登録機能

Ruby on RailsのAPIモードを用いて、JWT認証を実装していきます。

:::message

バージョン情報

- Ruby: 3.2.2
- Rails: 7.1.2

:::

事前準備を一気に書いていきます。

- APIモードでRailsアプリを作成

```shell
rails new jwt_app --api
```

- Gemfileを編集

```gemfile
source "https://rubygems.org"

ruby "3.2.2"

gem "rails", "~> 7.1.2"

gem "sqlite3", "~> 1.4"

gem "puma", ">= 5.0"

gem "bcrypt", "~> 3.1.7"

gem "tzinfo-data", platforms: %i[ windows jruby ]

gem "bootsnap", require: false

gem "rack-cors"

gem "jwt"

group :development, :test do
  gem "debug", platforms: %i[ mri windows ]
end
```

- gemをインストール

```shell
bundle install
```

- データベースを作成

```shell
rails db:create
```

- userモデルを作成

```shell
rails g model user name:string email:string password_digest:string
```

- マイグレート

```shell
rails db:migrate
```

- usersコントローラーを作成

```shell
rails g controller Users
```

- conntorollers/apiディレクトリを作成,移動しファイルを編集

```diff ruby:app/controllers/api/users_controller.rb
- class UsersController < ApplicationController
+ class Api::UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      render json: {user: {name: @user.name, email: @user.email}}
    else
      render json: {errors: {body: @user.errors}}, status: :unprocessable_entity
    end
  end

  private
  def user_params
    params.require(:user).permit(:name, :email, :password)
  end
end
```

- userモデルを編集

```ruby:app/models/user.rb
class User < ApplicationRecord
  has_secure_password
end
```

- routes.rbを編集

```ruby:config/routes.rb
Rails.application.routes.draw do
  namespace "api" do
    resources :users, only: %i[create]
  end
end
```

ここまでで、userを新規登録するところまでの準備が整いました。

postmanを使ってAPIを叩いてみます。
試しに以下のjsonを`http://localhost:3000/api/users`へPOSTしてみましょう。

```json
{
    "user": {
        "name": "test",
        "email": "test@mail.com",
        "password": "1234"
    }
}
```

以下のようなレスポンスが返って来れば、ユーザー登録機能の完成です。

![postman](/images/api_postman.png)

:::message
rails consoleを使って、Userデータを取得して登録できているかも確認してみましょう。
:::

### ログイン機能

```shell
rails g controller authentication
```

- routes.rbを編集

```diff ruby:config/routes.rb
Rails.application.routes.draw do
  namespace "api" do
    resources :users, only: %i[create]
+   post 'login', to: 'authentication#login'
  end
end
```

- ログインメソッドを実装

```ruby:app/controllers/api/authentication_controller.rb
  def login
    @user = User.find_by_email(params[:user][:email])
    if @user&.authenticate(params[:user][:password])
      token = create_token(@user.id)
      render json: {user: {email: @user.email, token: token, username: @user.username}}
    else
      render status: :unauthorized
    end
  end
```

- create_tokenメソッドを定義

```ruby: app/controllers/apprication_controller.rb
class ApplicationController < ActionController::API
  def create_token(user_id)
    payload = {user_id: user_id}
    secret_key = Rails.application.credentials.secret_key_base
    token = JWT.encode(payload, secret_key)
    return token
  end
end
```

これは、user_idを引数として渡して、ペイロード部分に`{user_id: 1}`のような形でJWTを作成しています。

また、secret_keyとして、`Rails.application.credentials.secret_key_base`としていますが、これはRailsアプリケーションでランダムに作成される文字列を返してくれます。

gemでjwtをインストールしていますので、最後に`JWT.encode(payload, secret_key)`を使って暗号化したJWSをtokenとして返します。

流れをまとめると、

1. `/api/login`へemailとpasswordをPOST送信
2. 登録されているユーザーが存在していれば、tokenを生成
3. ユーザー情報と生成したtokenを返却

このようになります。

postmanを使って、ログインのテストをしてみましょう。

以下のjsonデータを、`http://localhost:3000/api/login`へPOST送信してみます。

```json
{
    "user": {
        "email": "test@mail.com",
        "password": "1234"
    }
}
```

![ログイン](/images/api_postman2.png)

このようなレスポンスが得られればOKです。

## authenticateメソッドを定義

現在のところまででは、メールアドレスとパスワードが正しいときに、トークンを生成して返しているだけです。

ログイン中であることを判定するためのメソッドが必要になります。

そのためには次のような機能を実装していきます。

1. ログイン状態でないとできないリクエストを送信するときは、発行されたトークンをリクエストに含める。
2. リクエストに含まれているトークンの有効性を確認する。
3. 確認できたら、次の処理に進める。

```ruby:app/controllers/apprication_controller.rb
  def authenticate
    authorization_header = request.headers[:authorization]
    if !authorization_header
      render_unauthorized
    else
      token = authorization_header.split(" ")[1]
      secret_key = Rails.application.credentials.secret_key_base

      begin
        decoded_token = JWT.decode(token, secret_key)
        @current_user = User.find(decoded_token[0]["user_id"])
      rescue ActiveRecord::RecordNotFound
        render_unauthorized
      rescue JWT::DecodeError
        render_unauthorized
      end
    end
  end

  def render_unauthorized
    render json: { errors: 'Unauthorized' }, status: :unauthorized
  end
```

リクエストのヘッダーの`authorization`属性にトークンいれた状態で、リクエストを送信します。
authorizationにトークンがあれば、それを取り出します。

`authorization_header.split(" ")[1]`としているのは、authorization_headerは、
`Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxfQ.lMVlELJTVaVz8bu3NI1NGdfeXq_UbDBwb4Pa7B0SWO8`のようになっているので、Bearerの後の空白で区切った配列を作成し、その後ろの部分をトークンとして取得しているからです。

secret_keyは先程暗号化したときと同じものを使い、トークンを復号化します。

このとき、復号化できない（トークンが改ざんされているなど）の時に、`JWT::DecodeError`が発生するので、rescueで拾って、unauthorizedレスポンスを返します。

復号化ができれば、ペイロードに`{user_id: 1}`のようにuser_idを入れているので、それを取り出して`@current_user`を返します。

このauthenticateメソッドは、before_actionで使います。
before_actionではrenderされると、次の処理に進まなくなるので、tokenの有効性が確認でき、user_idが一致するユーザーが見つかった場合のみ、`@current_user`にログインしたユーザーがセットされます。

逆に言えば、それ以外の場合は@current_userはnilになりますが、先にrenderしてしまうので、問題ないというわけです。

試しに、userの属性を編集するupdateメソッドを定義してみしょう。このとき、ログインしていなければ編集できないようにします。

```diff ruby:app/controllers/users_controller.rb
class Api::UsersController < ApplicationController

+ before_action :authenticate, only: %i[update]

  def create
    @user = User.new(user_params)
    if @user.save
      render json: {user: {name: @user.name, email: @user.email}}
    else
      render json: {errors: {body: @user.errors}}, status: :unprocessable_entity
    end
  end
  
+ def update
+   if @current_user.update(user_params)
+     render json: {user: {name: @current_user.name}}
+   else 
+     render json: {errors: {body: @current_user.errors}}, status: :unprocessable_entity
+   end
+ end

  private
  def user_params
    params.require(:user).permit(:name, :email, :password)
  end
end

```

```diff ruby:config/routes.rb
Rails.application.routes.draw do
  namespace "api" do
+   resources :users, only: %i[create update]
    post 'login', to: 'authentication#login'
  end
end
```

これで、`http://localhost:3000/api/users/1`に対してPUTメソッドを送ると、authorizationヘッダに正しいトークンが格納されていれば、ユーザー名を変更することができます。

postmanでトークンをヘッダに含めるには、以下の画像のように、
`Auth`→`Bearer Token`を選択し、右側にトークンを貼り付けた状態で、リクエストを送信します。

![authorization_header](/images/api_postman3.png)

以下のように、変更されたユーザー名がレスポンスとして返って来ればOKです。

![user_update](/images/api_postman4.png)

ここまでで、JWT認証の基本が完成しました。

:::message
トークンを設定しないと、401:unauthorizedエラーを返すことも確認しましょう。
:::

## JWTトークンに有効期限を設定

`create_token`で、ペイロードに有効期限をセットします。

```diff ruby:app/controllers/apprication_controller.rb
  def create_token(user_id)
+   payload = {user_id: user_id, exp: (DateTime.current + 14.days).to_i}
    secret_key = Rails.application.credentials.secret_key_base
    token = JWT.encode(payload, secret_key)
    return token
  end
```

`DateTime.current + 14.days).to_i`これで、現在の時刻から2週間後を期限とすることができます。

ちなみに、JWTはこちらのサイトで簡単に中身を確認することができます。
https://jwt.io/#debugger-io

![jwtio](/images/jwtio.png)

この画像のように、expのところにカーソルをあわせると、UNIXタイムを変換して表示してくれます。

```diff ruby:app/controllers/application_controller.rb
  def authenticate
    authorization_header = request.headers[:authorization]
    if !authorization_header
      render_unauthorized
    else
      token = authorization_header.split(" ")[1]
      secret_key = Rails.application.credentials.secret_key_base

      begin
        decoded_token = JWT.decode(token, secret_key)
        @current_user = User.find(decoded_token[0]["user_id"])
      rescue ActiveRecord::RecordNotFound
        render_unauthorized
+     rescue JWT::ExpiredSignature
+       render json: { errors: 'ExpiredSignature' }, status: :unauthorized
      rescue JWT::DecodeError
        render_unauthorized
      end
    end
  end
```


JWT gemは、有効期限超過の場合、`JWT::ExpiredSignature`エラーを返すので、わかりやすいようにauthenticateメソッドに追加しています。

:::message
試しに、有効期限を`DateTime.current + 1.minutes).to_i`などと短めに設定して、ログインによってトークンを発行し、ユーザー情報のアップデートをしてみましょう。
また、有効期限が切れた後に、同じトークンを使って`ExpiredSignature`エラーが起こることも確認しましょう。
:::

これで、JWTに有効期限を設定することができました。

## リフレッシュトークン

JWSによって改ざんを防ぐことはできますが、JWSそのものが流出した場合、なりすましができてしまいます。
そもそも頻繁にネットワークを流れるものなので、盗聴されるリスクは高いです。

そこで、有効期限を短く設定しておくのが通常です。仮に盗聴されてしまったとしても、すぐに有効期限が切れるようにしておけば、被害は最小限に抑えられます。

しかし、ユーザーからしてみれば、すぐに有効期限が切れるのでは、頻繁にログインをし直さなければならないため、面倒です。

そこで「リフレッシュトークン」を使います。

リフレッシュトークンは、有効期限が切れたトークンを再発行するために使われます。

## JWT認証におけるリフレッシュトークンの個人的考え方

そもそもJWT認証のメリットは、サーバー側がステートレス（ログイン中かどうかを判断しない）であること、トークンの有効期限が短いことです。

リフレッシュトークンを使うことで、逆にJWT認証のシンプルさが損なわれ、期限が長いことでセキュリティ上の危険性は増すことになります。

UXのために有効期限を伸ばしたいと考えるのであれば、そもそもJWT認証は候補から外すべきかもしれません。

## まとめ

シンプルなJWT認証の実装方法を解説しました。

もちろん、JWT認証を使えば安全というものではなく、メリット・デメリットを把握した上で実装することが重要です。
