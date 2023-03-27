---
title: "GitHub Actions OIDC でブランチや Environment ごとに Google Cloud の異なるロールを利用する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud", "GCP", "terraform", "githubactions", "OIDC"]
published: false
---

# GitHub Actions OIDC でブランチや Environment ごとに Google Cloud の異なるロールを利用する
GitHub Actions の OIDC を利用して workflow からキーレスで Google Cloud の権限を使う際の、Workload Identity プールやサービスアカウントの設定について説明します。この際に実行ブランチや Environment など、workflow の条件に応じて異なるサービスアカウントに接続するための設定を例示します。

## さいしょにまとめ

- ロールの異なる複数の Google Cloud サービスアカウントを作成する
- サービスアカウントごとに、Workload Identity プールからの借用条件を設定する
- GitHub から渡される情報を適切に Workload Identity プールの属性にマップする

この記事で扱っている Terraform コードは [GitHub の komazarari/example_githubactions_oidc](https://github.com/komazarari/example_githubactions_oidc) にあります。

## やること
- Workload Identity プールの作成、認証条件の指定、トークンからの属性マッピング
- サービスアカウントの作成、借用(impersonation)の許可と条件指定

### 書かないこと
- GitHub Actions の設定
- Terraform の使い方
- サービスアカウントの権限設定

## やりかた
### Workload Identity プール の作成
以下の Terraform コードは Workload Identity プール `github-pool` とプロバイダー `github-actions` を作成します。
このプールにより、GitHub Actions によって提供される ID を Google Cloud の認証されたプリンシパル^["プリンシパル"とは雑に言えば権利を行使するモノやヒトたちです。こちら: https://cloud.google.com/iam/docs/overview#concepts_related_identity 参照] として扱えるようになります。

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

![](https://storage.googleapis.com/zenn-user-upload/e00301868df6-20230327.png)


`assertion.*` の中身がわかるとイメージしやすいかもしれません。以下にトークンの一部形式を引用します。[GitHub Actions の OIDC トークンのドキュメント](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token)も参照してください。
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

プロバイダーの設定では属性条件 `attribute_condition` を指定することにより、認証範囲を特定のレポジトリに制限しています。ここでは `assertion.repository_owner` を使って GitHub の Organization/ユーザ 名を指定していますが、単一のレポジトリのみにしたり、複合条件にしてより細かく指定することも可能です。属性条件を指定しない場合は全 GitHub の Actions が対象になります。後述のサービスアカウント側の設定で権限借用の範囲を限定しておけば、だれでもどこからでも使われてしまうようなことにはなりませんが、多くの場合は制限しておくほうがよいでしょう。

`attribute_mapping` では、GitHub から渡される情報を Workload Identity プールの属性にマップします。
これは後ほど権限借用の条件に使います。

```
... (再掲)
    "attribute.org_repo_branch"  = "assertion.repository + \"/\" + assertion.ref.extract('refs/heads/{branch}')"
...
```
この `attribute.org_repo_branch` 部分のマッピングは少し奇妙に見えるかもしれませんので補足します。マッピングには Common Expression Language 式が使用できます ([属性のマッピング](https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja#mapping)参照) 。トークンの `assertion.repository` に入ってくる `<org>/<repo>` の文字列と、`assertion.ref` にはいってくる `refs/heads/<branch>` から最後の `<branch>` 部分だけを extract 関数で抽出し、 `/` で結合して`<org>/<repo>/<branch>` の形式の文字列になるようにしています。


### サービスアカウントの作成
以下の Terraform コードは 3 つのサービスアカウント `foo` `bar` `baz` (プレフィックス略) を作成し、Workload Identity プールで認証されたプリンシパルに対して権限借用の条件を設定します。

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

Web コンソールではサービスアカウントの権限(PERMISSIONS)欄に表示されます。
![](https://storage.googleapis.com/zenn-user-upload/1e8537d4014c-20230327.png)

上記の例では、それぞれ以下の意味になるよう許可ポリシーを設定しています。

- `foo` : `<org名>/example_githubactions_oidc` というレポジトリの `main` ブランチで実行される workflow 相当の単一のプリンシパルを許可
- `bar` : `<org名>/example_githubactions_oidc` というレポジトリの `develop` ブランチで実行される workflow 相当のプリンシパル集合を許可
- `baz` : Actions の Environemnt `production` で実行される workflow 相当のプリンシパル集合を許可

ここで指定しているプリンシパルは前述の Workload Identity プールから提供されるものなので、当然ながらプールの subject や 属性のマッピング設定によって文字列が変わってきます。指定方法については[サービスアカウントの権限借用](https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja#impersonation)も参照してください。 

もうすこし詳しく書くと

- `foo` の指定はシンプルに subject が `repo:<org>/<repo>:ref:refs/heads/main` である単一のプリンシパルを許可しています。これはレポジトリの `main` ブランチから実行された workflow に対応しています
  - ただし workflow が Environment を使っておらず、Pull request 起点でない場合に限ります。それぞれの場合でどのような文字列になるかは [subject claims 例](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#example-subject-claims) を参照してください。
  - 単一のプリンシパルなので、`principalSet://` ではなく `principal://` です。
- `bar` は、`<org>/<repo>/<branch>` という形になるよう整形した `attribute.org-repo-branch` を参照し、`<org>/<repo>/develop` を許可しています。これはレポジトリの `develop` ブランチから実行された workflow に対応しています。
  - もちろん、`foo` の場合と同様に `principal://(略..):refs/head/develop` のように指定することでもほぼ同じ意味の指定ができます。
- `baz` は、`attribute.environment` を参照しています。これは Environment `production` で実行された workflow に対応します。
  - この指定にはレポジトリやオーナーの情報が含まれていないことに注意してください。`attribute.environment` が `production` であるプリンシパル全てが許可されます。プールの属性条件で Organization 名を制限しているため、結果的に `<org>` 以下のレポジトリから Environment `production` で実行される全ての workflow に対応します。
  - 一部繰り返しになりますが、仮にプールの属性条件が指定されていないと、このプリンシパル集合は GitHub のあらゆるレポジトリから Environment `production` で実行される全 workflow に対応することに (実質全世界のだれでも借用可能に) なります。

## Workflow での権限借用

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
