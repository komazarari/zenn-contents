---
title: "GitHub Actions OIDC でブランチや Environment によって異なるロールを利用する (Google Cloud)"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud", "GCP", "terraform", "githubactions", "OIDC"]
published: true
---

GitHub Actions OIDC と Google Cloud の Workload Identity を利用して、workflow から秘密鍵配置なしで Google Cloud にアクセスできます。

このとき workflow の実行ブランチや Environment に応じて異なるロールを利用する方法を説明します。

## さいしょにまとめ

- ロールの異なる複数の Google Cloud サービスアカウントを作成する
- サービスアカウントごとに、Workload Identity プールからの借用条件を設定する
- GitHub から渡される情報を適切に Workload Identity プールの属性にマップする

この記事で扱っている Terraform コードは [GitHub の komazarari/example_githubactions_oidc](https://github.com/komazarari/example_githubactions_oidc) にあります。

## やること
- Workload Identity プールの作成、認証条件の指定、トークンからの属性マッピング
- サービスアカウントの作成、借用(impersonation)の許可と条件指定

### 詳しく書かないこと
- GitHub Actions の設定
- Terraform の使い方
- サービスアカウントの権限設定

## やりかた
### Workload Identity プール の作成
外部 ID を Google Cloud で認識するための Workload Identity プールを作成します。


以下の [Terraform コード](https://github.com/komazarari/example_githubactions_oidc/blob/main/googlecloud/workload_identity.tf)は Workload Identity プール `github-pool` とプロバイダー `github-actions` を作成します。このプールにより、GitHub Actions によって提供される ID を Google Cloud 側でプリンシパル^["プリンシパル"とは雑に言えば権利を行使するモノやヒトたちです。ロールを仮面や帽子に例えるなら、それをかぶる顔がプリンシパルと言えるかと思います。こちら: https://cloud.google.com/iam/docs/overview#concepts_related_identity 参照] として扱えるようになります。

```hcl
resource "google_iam_workload_identity_pool" "github" {
  project = var.gcp_project

  workload_identity_pool_id = "github-pool"
  display_name              = "github-pool"
  description               = "Identity pool for GitHub Actions"
}

resource "google_iam_workload_identity_pool_provider" "github_oidc" {
  project = var.gcp_project

  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-actions"

  display_name        = "GitHub OIDC"
  description         = "GitHub OIDC identity pool provider for GitHub Actions in Your Org"
  disabled            = false
  attribute_condition = "\"${var.org_name}\" == assertion.repository_owner"

  // https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token
  attribute_mapping = {
    "google.subject"             = "assertion.sub"
    "attribute.repository_owner" = "assertion.repository_owner"
    "attribute.ref"              = "assertion.ref"
    "attribute.environment"      = "assertion.environment"
    "attribute.org_repo_branch"  = "assertion.repository + \"/\" + assertion.ref.extract('refs/heads/{branch}')"
  }

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}
```

作成されたプロバイダーは Web コンソールでは以下のように表示されます。

![ScreenShot: Attribute Mapping and Attibute Conditions of the Workload Identity Provider](https://storage.googleapis.com/zenn-user-upload/e00301868df6-20230327.png)


`assertion.*` は GitHub から渡されるトークンに相当します。中身がわかるとよりイメージしやすいかもしれません。以下にトークンの一部形式を引用します。[GitHub Actions の OIDC トークンのドキュメント](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token)も参照してください。
```
{
  "jti": "example-id",
  "sub": "repo:octo-org/octo-repo:environment:prod",
  "environment": "prod",
  "aud": "https://github.com/octo-org",
  "ref": "refs/heads/main",
  "sha": "example-sha",
  "repository": "octo-org/octo-repo",
  "repository_owner": "octo-org",
  "actor_id": "12",
  "repository_visibility": "private",
  "repository_id": "74",
  "repository_owner_id": "65",
  "run_id": "example-run-id",
  "run_number": "10",
  "run_attempt": "2",
  "actor": "octocat",
  "workflow": "example-workflow",
  "head_ref": "",
  "base_ref": "",
  "event_name": "workflow_dispatch",
  "ref_type": "branch",
  "job_workflow_ref": "octo-org/octo-automation/.github/workflows/oidc.yml@refs/heads/main",
  "iss": "https://token.actions.githubusercontent.com",
  "nbf": 1632492967,
  "exp": 1632493867,
  "iat": 1632493567
}
```

プロバイダーの設定では属性条件 `attribute_condition` を指定することにより、認証範囲を特定のレポジトリに制限しています。ここでは `assertion.repository_owner` を参照し特定の GitHub の Organization/ユーザ である場合に制限しています。単一のレポジトリのみにしたり、複合条件にしてより細かく指定することも可能です。

属性条件を指定しない場合はすべての GitHub Actions workflow が対象になります。後述のサービスアカウント側の設定で権限借用の範囲を限定しておけば、だれでもどこからでも使われてしまうようなことにはなりませんが、多くの場合はプール側でも制限しておくほうがよいでしょう。

`attribute_mapping` では、GitHub から渡される情報を Workload Identity プールの属性にマップします。これは後ほど権限借用の条件に使います。


以下の `attribute.org_repo_branch` 部分は少し他と毛色が違って複雑に見えるかもしれません。
```
... (再掲)
    "attribute.org_repo_branch"  = "assertion.repository + \"/\" + assertion.ref.extract('refs/heads/{branch}')"
...
```
マッピングは `assertion.*` の値を対応させるだけではなく、Common Expression Language 式が使用できます ([属性のマッピング](https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja#mapping)参照) 。ここでは、`<org>/<repo>/<branch>` という形の属性を作りたかったので、`assertion.repository` に入ってくる `<org>/<repo>` の文字列と、`assertion.ref` にはいってくる `refs/heads/<branch>` から最後の `<branch>` 部分だけを CEL 関数の extract() で抽出し、 `/` で結合して目的の形にしています。


### サービスアカウントの作成
Workload Identity から借用させるためサービスアカウントを作成します。

以下の Terraform コードは 3 つのサービスアカウント `foo`、`bar`、`baz` (プレフィックス略) を作成し、Workload Identity プールで認証された特定のプリンシパルに対して権限借用を許可します。

```hcl
// foo サービスアカウントを作成
resource "google_service_account" "github_actions_foo" {
  account_id   = "github-actions-foo"
  display_name = "GitHub Actions Foo"
}

// bar サービスアカウントを作成
resource "google_service_account" "github_actions_bar" {
  account_id   = "github-actions-bar"
  display_name = "GitHub Actions Bar"
}

// baz サービスアカウントを作成
resource "google_service_account" "github_actions_baz" {
  account_id   = "github-actions-baz"
  display_name = "GitHub Actions Baz"
}

// ...中略...

// foo の権限借用を許可するプリンシパルを指定
resource "google_service_account_iam_binding" "github_actions_foo_wi_binding" {
  service_account_id = google_service_account.github_actions_foo.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "principal://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/subject/repo:${var.org_name}/example_githubactions_oidc:ref:refs/heads/main",
  ]
}

// bar の権限借用を許可するプリンシパルを指定
resource "google_service_account_iam_binding" "github_actions_bar_wi_binding" {
  service_account_id = google_service_account.github_actions_bar.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.org-repo-branch/${var.org_name}/example_githubactions_oidc/develop",
  ]
}

// baz の権限借用を許可するプリンシパルを指定
resource "google_service_account_iam_binding" "github_actions_baz_wi_binding" {
  service_account_id = google_service_account.github_actions_baz.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.environment/production",
  ]
}
```

`"google_service_account_iam_binding"` ではサービスアカウントごとに [許可ポリシー](https://cloud.google.com/iam/docs/overview?hl=ja#cloud-iam-policy)を設定しています。このポリシーで許可されたプリンシパルのみが権限借用を行うことができます。

これは Web コンソールではサービスアカウントの権限(PERMISSIONS)欄に表示されます。
![ScreenShot: Permitted principals of a ServiceAccount](https://storage.googleapis.com/zenn-user-upload/1e8537d4014c-20230327.png)

サービスアカウントそれぞれの設定について少し詳しく見ていきます。

#### 例: `foo` サービスアカウント - ブランチ指定

`foo` の許可ポリシーはシンプルに単一プリンシパルを指定しています。変数を展開した具体例だと以下のような形です。

```
principal://iam.googleapis.com/projects/123456789012/locations/global/workloadIdentityPools/github-pool/subject/repo:komazarari/example_githubactions_oidc:ref:refs/heads/main
```
マッピングを踏まえると、これは `komazarari/example_githubactions_oidc` レポジトリの `main` ブランチから実行された workflow に対応します。

ただし、workflow が Environment を使っておらず、Pull request 起点でない場合に限ります。このような場合の subject 文字列は [subject claims 例](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#example-subject-claims) を参照してください。

他の `bar`、`baz` と違って単一プリンシパルなので、`principalSet://~` ではなく `principal://~` です。

#### 例: `bar` サービスアカウント - 別の方法でブランチ指定
`bar` の許可ポリシーはマッピングのところで定義した `org_repo_branch` 属性を参照しています。具体例だと以下のような形です。

```
principalSet://iam.googleapis.com/projects/123456789012/locations/global/workloadIdentityPools/github-pool/attribute.org_repo_branch/komazarari/example_githubactions_oidc/develop
```

`org_repo_branch` 属性の値にあたるのが `komazarari/example_githubactions_oidc/develop` 部分です。結果として、これは `komazarari/example_githubactions_oidc` レポジトリの `develop` ブランチから実行された workflow に対応します。

`foo` と比較すると、こちらの指定の場合は Environment の使用有無や Pull request 起点かで条件が変わりません。

この属性を持つプリンシパルの集合という意味になるので、`principalSet://～` で記述します。

同じことを実現するために必ずしも属性を参照しなければいけないということではありません。例えば、`google.subject` へのマップを `assertion` の `repo` と `ref` を組み合わせた文字列を使うことでも近いことができます。その場合は単一のプリンシパルになるので `principal://~/subject/~` のような形になります。


#### 例: `baz` サービスアカウント - Environment 指定
`baz` の許可ポリシーは `environment` 属性を参照しています。具体例だと以下のような形です。

```
principalSet://iam.googleapis.com/projects/123456789012/locations/global/workloadIdentityPools/github-pool/attribute.environment/production
```

この指定にはレポジトリやオーナーの情報が含まれていないことに注意してください。`attribute.environment` が `production` であるプリンシパル全てが許可されます。本記事の設定ではプールの属性条件で Organization 名を制限しているため、結果的に `<org>` 以下のレポジトリから Environment `production` で実行される全ての workflow が対応します。

一部繰り返しになりますが、仮にプールの属性条件が指定されていないと、このプリンシパル集合は GitHub のあらゆるレポジトリから Environment `production` で実行される workflow に対応することに (実質全世界のだれでも借用可能に) なります。

### Workflow での権限借用

詳しくは言及しませんが、https://github.com/google-github-actions/auth を利用できます。

## あらためてまとめ

workflow の条件によって権限を使い分けるには:

- ロールの異なる複数の Google Cloud サービスアカウントを作成する
  - 例えば本番リリース用サービスアカウント、定期チェックに使うサービスアカウント、PR テスト用サービスアカウントなど、workflow の求める権限ごとに分けて作成する
- サービスアカウントごとに、Workload Identity プールからの借用条件を設定する
  - Workload Identity ユーザロールの許可ポリシーを設定する。ポリシーにはプリンシパル(集合)を指定する
  - ここで使いたい条件をもとに、プールでの属性マッピングを考える
- GitHub から渡される情報を適切に Workload Identity プールの属性にマップする
  - プロバイダーの属性条件を指定し、必要最小限の範囲で GitHub Actions を認証する
  - トークンの中身を理解し、自分の使いたいようにマップする
  - マップには CEL の式が使え、文字列加工ができる

-------

## 関連
- IAM の概要 https://cloud.google.com/iam/docs/overview
- デプロイ パイプラインによるサービス アカウントの権限借用を許可する https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines?hl=ja#allow_the_deployment_pipeline_to_impersonate_the_service_account
- サービス アカウントの権限借用 https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja#impersonation
- 属性のマッピング https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja#mapping
- GitHub Docs / Example subject claims https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#example-subject-claims
