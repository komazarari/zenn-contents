---
title: "Lambda@Edge で TypeScript を使うための SAM テンプレート設定"
emoji: "👾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Lambda", "Lambda@Edge", "TypeScript", "SAM", "AWS"]
published: true
---

AWS SAM CLI は TypeScript を使ったテンプレートを出力できる。

AWS からさまざまな用途向けにテンプレートが用意されているのだが、Lambda@Edge 用はいまのとこ (2023/02) 無いようなので、基本の Hello World Example TypeScript テンプレートをもとに、Lambda@Edge で使うための差分について書いておく

## 元にするテンプレートの生成

SAM CLI v1.72.0、2023/02/08 時点で、`sam init` 時の
`Hello World Example` -> `runtime: node18.x` 
 で TypeScript 用のテンプレートが選択できることを確認している。

今後テンプレートの更新でおいおいもっと適切なベースが出てくるかもしれない。AWS のテンプレートのレポ https://github.com/aws/aws-sam-cli-app-templates/blob/master/manifest-v2.json も参考に。



sam init で、TypeScript 用テンプレートを生成する。
```bash
sam init
...
(中略)
...
Select your starter template
        1 - Hello World Example
        2 - Hello World Example TypeScript
Template: 2
...
```
これにより、以下のようなファイルが生成される。

```bash
├── README.md
├── events
│   └── event.json
├── hello-world
│   ├── app.ts
│   ├── jest.config.ts
│   ├── package.json
│   ├── tests
│   │   └── unit
│   │       └── test-handler.test.ts
│   └── tsconfig.json
└── template.yaml
```

## Lambda@Edge 用の差分

### template.yaml

- `Hello World Example` は API Gateway とのセットのテンプレートなので、API Gateway 部分は削っておく^[そもそも init 時の選択には `Hellow World Example` ではなく、API Gateway の付いてない `Standalone function` というのもあって用途としてはコッチが近いのだが、こちらは TypeScript 用が現時点で無かった )]
- Lambda@Edge に使うには関数にバージョンを作成し、バージョン付きの ARN を指定する必要がある。`AutoPublishAlias` を有効にしておくと更新のたびにバージョンを作成してれて便利なので何らかの値を付けておく。以下では `LIVE` としているが何でもよい
- Lambda 関数用 Role の信頼されたエンティティに `edgelambda.amazonaws.com` を追加する
- `SourceMap` が有効だと Lambda 関数に `NODE_OPTIONS=–enable-source-maps` 環境変数が設定されるが、Lambda@Edge では環境変数が使えないので無効にしておく

```diff
--- a/template.yaml
+++ b/template.yaml
@@ -19,18 +19,23 @@ Resources:
       Runtime: nodejs18.x
       Architectures:
         - x86_64
-      Events:
-        HelloWorld:
-          Type: Api # More info about API Event Source: https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#api
-          Properties:
-            Path: /hello
-            Method: get
+      AutoPublishAlias: LIVE
+      AssumeRolePolicyDocument:
+        Version: "2012-10-17"
+        Statement:
+          - Effect: Allow
+            Principal:
+              Service:
+                - lambda.amazonaws.com
+                - edgelambda.amazonaws.com
+            Action:
+              - sts:AssumeRole
     Metadata: # Manage esbuild properties
       BuildMethod: esbuild
       BuildProperties:
         Minify: true
         Target: "es2020"
-        Sourcemap: true
+        Sourcemap: false
         EntryPoints:
         - app.ts

@@ -38,9 +43,6 @@ Outputs:
   # ServerlessRestApi is an implicit API created out of Events key under Serverless::Function
   # Find out more about other implicit resources you can reference within SAM
   # https://github.com/awslabs/serverless-application-model/blob/master/docs/internals/generated_resources.rst#api
-  HelloWorldApi:
-    Description: "API Gateway endpoint URL for Prod stage for Hello World function"
-    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello/"
   HelloWorldFunction:
     Description: "Hello World Lambda Function ARN"
     Value: !GetAtt HelloWorldFunction.Arn
```

### app.ts
以下は Viewer Request を想定した Lambda@Edge 関数の場合の例。

- 型情報は `@types/aws-lambda` に含まれている。Viewer Request 用には `CloudFrontRequestEvent` と `CloudFrontRequestResult` を import
- 戻り値を Lambda@Edge の Viewer Request 用に

```diff
--- a/hello-world/app.ts
+++ b/hello-world/app.ts
@@ -1,30 +1,25 @@
-import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
+import { CloudFrontRequestEvent, CloudFrontRequestResult } from 'aws-lambda';

 /**
  *
- * Event doc: https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-input-format
- * @param {Object} event - API Gateway Lambda Proxy Input Format
+ * @param {Object} event - CloudFront Request event Format
  *
- * Return doc: https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html
- * @returns {Object} object - API Gateway Lambda Proxy Output Format
+ * @returns {Object} object - CloudFront Result Format
  *
  */

-export const lambdaHandler = async (event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> => {
-    try {
-        return {
-            statusCode: 200,
-            body: JSON.stringify({
-                message: 'hello world',
-            }),
-        };
-    } catch (err) {
-        console.log(err);
-        return {
-            statusCode: 500,
-            body: JSON.stringify({
-                message: 'some error happened',
-            }),
-        };
-    }
+export const lambdaHandler = async (event: CloudFrontRequestEvent): Promise<CloudFrontRequestResult> => {
+    return {
+        status: '200',
+        statusDescription: 'OK',
+        headers: {
+            'content-type': [
+                {
+                    key: 'Content-Type',
+                    value: 'text/plain',
+                },
+            ],
+        },
+        body: 'Hello World!',
+    };
 };
```

これでビルド、デプロイされた関数の ARN を CloudFront に紐付けて Lambda@Edge としてつかえるはず。
