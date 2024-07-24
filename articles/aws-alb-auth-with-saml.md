---
title: "AWS ALB と Entra ID (旧 Azure AD) を SAML 認証で連携する"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "ALB", "SAML", "entraid", "azuread"]
published: false
---

AWS ALB には認証機能があり、バックのターゲットへリクエストを流す前に認証を行うようルールを構成することができます。ここでは Microsoft Entra ID (旧 Azure AD) と連携して SAML 認証を行う設定について説明します。
:::message
記事内で作成した各種クラウドリソースは全て削除済みです
:::

## 構成

ALB は認証方法として OpenID Connect (OIDC) または Amazon Cognito をサポートしています。ALB で直接 SAML を扱うことはできないので、ALB からは AWS Cognito を指定し、Cognito 側で Entra ID との SAML 連携を行います。


あくまで SAML を処理するのは Cognito であり、ALB は Cognito からユーザ情報を受け取るという点に留意してください^[SAML 部分では EntraID(IdP)-Cognito(SP) であり、ALB + Cognito 部分は OIDC 風に Cognito(IdP)-ALB(RP) のようになる]。

## Cognito リソースの準備

最初に一部の Cognitor リソースを作成し、Entra ID の SAML 設定時に必要になる名前や ID を確定させておきます。

### Cognito User Pool 作成
Cognito ユーザープールを作成します。Web コンソールの GUI で作成してもよいのですが、選択項目が多めなので、ここについてはシンプルに作成できる AWS CLI を使って作成します。

AWS CLI を使う環境が整っていない場合は、Web コンソール上部にある Cloud Shell が簡単でおすすめです。

以下のコマンドでユーザープールを作成します。`my-user-pool` を適当な名前で置き換えてください。このコマンドでは Entra ID 外ユーザのサインアップを防ぐために `--admin-create-user-config '{"AllowAdminCreateUserOnly":true}'` を指定しています。
```
aws cognito-idp create-user-pool \
  --pool-name my-user-pool \
  --admin-create-user-config '{"AllowAdminCreateUserOnly":true}'
```

出力が以下のようになるので、`Id` の値を控えておきます。この場合は `ap-northeast-1_3dmYPi9Cn` です。
```json
{
    "UserPool": {
        "Id": "ap-northeast-1_3dmYPi9Cn",
        "Name": "my-user-pool",
        "Policies": {
...
    }
  }
}
```

### Cognito ドメイン作成

Cognito のホステッド UI を使うためのドメインを作成します。このドメインは SAML 認証後の応答の宛先 (Reply URL) として使われます。

独自ドメインを使うこともできますが、簡単のため Cognito のサブドメインを使って作成します。作成されるドメインは `https://<yourDomainPrefix>.auth.<aws-region>.amazoncognito.com` の形になるので、グローバルユニークな名前になるようにコマンドの `--domain <yourDomainPrefix>` 部分を置き換えてください。

```
aws cognito-idp create-user-pool-domain \
  --domain <yourDomainPrefix> \
  --user-pool-id <yourUserPoolID>
```

ここでは以下のように実行したものとします。もしドメインが既に使われていてエラーになる場合は適宜変更してください。
```bash
aws cognito-idp create-user-pool-domain \
  --domain my-cognito-auth-test \
  --user-pool-id ap-northeast-1_3dmYPi9Cn \
  --region ap-northeast-1
```

このとき生成されるドメインは次のようになります。これも控えておきます。
```
https://my-cognito-auth-test.auth.ap-northeast-1.amazoncognito.com
```

なお、Web UI からドメイン作成を行う場合は、`Amazon Cognito > ユーザープール > [対象のユーザープール]` 内にあるタブの `アプリケーションの統合` から作成できます。

## Entra ID (Azure AD) SAML 設定
次に Azure Entra ID 側を設定します。

エンタープライズアプリケーションから、`独自のアプリケーションの作成` に進みます。任意の名前でアプリケーションを作成します。

![entraid-appの作成](/images/aws-alb-auth-with-saml_create-entraid-app.png)

シングルサインオンの設定では SAML を選択し設定を進めます。

### SAML SSO セットアップ
入力が必要になるのは
- 識別子 (エンティティ ID)
- 応答 URL (Reply URL, Assertion Consumer Service URL)

です。先ほど作成した Cognito の情報を使って入力していきます。


識別子 (エンティティ ID) に入力する値は、Cognito のユーザープール ID を含む `urn:amazon:cognito:sp:<user pool id>` の形式になります。

今回作成したユーザープール ID  `ap-northeast-1_3dmYPi9Cn` の例では以下のようになります。

```
urn:amazon:cognito:sp:ap-northeast-1_3dmYPi9Cn
```

応答 URL には、Cognito の `https://<ホステッド UI のドメイン>/saml2/idpresponse` を入力します。例えば、先ほど作成したドメインの例であれば以下です。
```
https://my-cognito-auth-test.auth.ap-northeast-1.amazoncognito.com/saml2/idpresponse
```

![SAML設定](/images/aws-alb-auth-with-saml_saml-settings.png)

基本的な SAML 構成が入力できたら、下部証明書の中にあるフェデレーションメタデータ URL をコピーして控えておきます。

設定保存時にテストをするかを聞かれるかもしれませんが、この構成ではテストできないためスキップで問題ありません。

#### (省略可) クレームの追加
ユーザーの属性やロールの割り当てによって認可情報を渡したい場合は、属性とクレームより任意の情報を追加するこができます。

例えば、エンタープライズアプリケーション内で割り当てたロールを渡したい場合は `user.assignedroles` をクレームに含めます。
クレーム名を `http://my.examples.com/claims/roles` としたならば、以下の図のようになります。

![クレームの追加](/images/aws-alb-auth-with-saml_add-claim.png)

ここで設定したクレーム名は Cognito でユーザ属性にマッピングする必要があります。

## ALB のドメインを確定

アクセスの起点および認証後に戻ってくる URL となる ALB のドメインを確定します。HTTPS でアクセスできる必要があるので、適切に証明書を設定可能なドメインを使ってください。

:::message
本記事では、`my-alb-domain.example.com` というドメイン名を使って説明します。
:::

## Cognito 設定
作成したユーザープールが Entra ID を外部 ID プロバイダとして使えるように設定します。
### Entra ID を ID プロバイダーとして追加
アイデンティティプロバイダーを追加します。
![CognitoプールにIdPを追加](/images/aws-alb-auth-with-saml_add-idp-to-cognito-pool.png)
SAML を選択し、先ほど控えたフェデレーションメタデータ URL を入力します。
![IdPをSAMLを使って設定](/images/aws-alb-auth-with-saml_saml-as-idp-settings.png)
必要に応じて、属性のマッピングを設定してください。後から追加、変更も可能です。

:::details マッピング例
|ユーザープール属性|SAML属性|
| --- | --- |
|email|http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress|
|custom:roles|http://my.examples.com/claims/roles|

※ `custom:roles` のようなカスタム属性は先にユーザープールに定義しておく必要があります。プールの `サインアップエクスペリエンス` から設定します。
:::

### アプリケーションクライアントを作成
`アプリケーションの統合` よりアプリケーションクライアントを作成します。これは ALB が Cognito から情報取得するためのクライアントです。

`秘密クライアント` を選択し、`許可されているコールバック URL` に `https://<ALB ドメイン>/oauth2/idpresponse` (本記事の例では `https://my-alb-domain.example.com/oauth2/idpresponse`) を追加します。

 ![Cognitoプールに秘密クライアントを追加](/images/aws-alb-auth-with-saml_add-app-secret-client.png)

![許可コールバックにALBドメインを追加](/images/aws-alb-auth-with-saml_add-allowed-callbacks.png)


ここまでで Entra ID と Cognito の基本設定は完了です。

## ALB から Cognito への認証設定

### テスト用 Lambda ターゲット
テスト用に適当な Lambda を作成し ALB からの認証済みリクエストを受け付けるようにします。以下は ALB から渡されるヘッダをそのまま表示する関数の例です。あくまで本記事の説明用なのでご注意ください。

```javascript
export const handler = async (event, context) => {
    const headers = event.headers;

    let content = '';
    for (let key in headers) {
        content += `${key} : ${headers[key]}\n`;
    }
    const response = {
        statusCode: 200,
        headers: {"content-type": "text/plain"},
        body: content,
    };
    return response;
};
```

この Lambda 関数を作成しターゲットグループとして登録しておきます。

### ALB 認証ルール設定

ALB のルールアクションで、認証に Cognito を使うよう設定します。認証後にルーティングされるターゲットグループとして先程の Lambda のものを指定します。

![ALBに認証ルールを設定しCognitoを指定](/images/aws-alb-auth-with-saml_alb-auth-with-cognito.png)

ユーザープール、ドメイン、アプリケーションクライアントがここまで作成したものになっているか確認してください。

### 動作確認
準備ができたら、プライベートブラウジングで開発ツールを開きながら ALB にアクセスしてみます。うまく行っていれば概ね以下のような流れでリクエストが流れるはずです。
- ALB `https://my-alb-domain.example.com/` にアクセス
- Cognito のログイン画面 `https://my-cognito-auth-test.auth.ap-northeast-1.amazoncognito.com/` にリダイレクト
- Entra ID のログイン画面 `https://login.microsoftonline.com` にリダイレクト
- Microsoft アカウントのログイン処理が続く
- Cognito の応答 URL `https://my-cognito-auth-test.auth.ap-northeast-1.amazoncognito.com/saml2/idpresponse` に戻ってくる
- ALB のコールバック URL `https://my-alb-domain.example.com/oauth2/idpresponse` にリダイレクト
- 元の URL `https://my-alb-domain.example.com/` にリダイレクト
- Lambda で用意したコンテンツが表示される

ブラウザには先程のテスト用 Lambda により ALB から渡されるヘッダが表示されるはずです。

```
accept : text/html,...
accept-encoding : gzip, deflate, br, zstd
...
x-amzn-oidc-accesstoken : eyJraWQi......
x-amzn-oidc-data : eyJ0eXAiOiJKV1......
x-amzn-oidc-identity : ********-****-****-****-************
...
```
`x-amzn-oidc-data` が Entra ID の SAML レスポンスを反映した内容になっています。[公式ドキュメント](https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/application/listener-authenticate-users.html)いわく JWT ということですが、2024/07/20現在、標準に沿わないパディングが含まれた形式になっています。JWT ライブラリによってはデコードできないことがあるかと思いますのでご注意ください。

これをデコードすると Entra ID の SAML 設定で指定したクレームが含まれていることが確認できます。デコードはテスト目的であれば https://jwt.io/ などのサービスを使うと簡単です。

ローカルでデコードする場合は、公式ドキュメント中の python の例や [npm の jwt-decode](https://www.npmjs.com/package/jwt-decode)、[rubygem jwt (検証 false にする必要あり)](https://rubygems.org/gems/jwt)、あるいは coreutils に含まれる `basenc` コマンドなどが使えます。


```bash
echo eyJ0eXAiOiJKV1...... | tr '.' '\n' | head -n2 | basenc -d --base64url
```

また、 `x-amzn-oidc-accesstoken` は Cognito から発行されたアクセストークンです。これを使って Cognito からユーザ情報を取得することもできます^[取れる情報は scope による。Cognito 内の属性を取るにはクライアントに profile scope を許可し ALB 側の認証設定でも指定する]。

実際のアプリケーションではこれらのヘッダを使って SAML 認証されたユーザー情報を扱うことができます。

## まとめ
肝心の `x-amzn-oidc-data` 部分が非標準 JWT になっているという扱いにくさはあるものの、ALB と Cognito を使えばフルマネージドで SAML を受けることができます。ALB の豊富な機能と合わせ、バックエンドのターゲットをシンプルに保ちつつ、認証の選択肢を増やせるかと思います。


## 参考資料・関連リンク
- [How to set up Amazon Cognito for federated authentication using Azure AD | AWS Security Blog](https://aws.amazon.com/jp/blogs/security/how-to-set-up-amazon-cognito-for-federated-authentication-using-azure-ad/)
- [Application Load Balancer を使用してユーザーを認証する - Elastic Load Balancing](https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/application/listener-authenticate-users.html)
- [RFC 7515 - JSON Web Signature (JWS)](https://datatracker.ietf.org/doc/html/rfc7515)
- [JSON Web Tokens - jwt.io](https://jwt.io/)
- [ALB + Cognito認証で付与されるユーザー情報をEC2サイドから眺めてみる | DevelopersIO](https://dev.classmethod.jp/articles/http-headers-added-by-alb-and-cognito-are-explained/)
