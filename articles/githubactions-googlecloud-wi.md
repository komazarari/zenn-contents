---
title: "GitHub Actions OIDC でブランチや Environment ごとに Google Cloud の異なるロールを利用する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud", "GCP", "terraform", "githubactions", "OIDC"]
published: false
---

# まとめ

- ロールの異なる複数のサービスアカウントを作成する
- サービスアカウントごとに、Workload Identity からの借用条件を設定する
- GitHub から渡される情報を適切に Workload Identity プールの属性にマップする


# かきたいこと
- 実現したいこと
  - workflow が実行されるブランチごと、または workflow の Environment ごとに、異なる権限を付与したい


- terraform で説明します


## Workload Identity プール の作成
以下の Terraform コードは Workload Identity プール `github-pool` とプロバイダー `github-actions` を作成します。
このプールにより、GitHub Actions によって提供される ID を Google Cloud の認証されたプリンシパル^["プリンシパル"とは雑に言えば権利を行使するモノやヒトたちです。こちら https://cloud.google.com/iam/docs/overview#concepts_related_identity 参照]として扱えるようになります。

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

`attribute_condition` (属性条件) を指定することにより、このプールでは認証範囲を特定のレポジトリに制限しています。ここでは Organization/User 名を指定していますが、単一のレポジトリにしたり、複合条件にしてより細かく指定することも可能です。これを指定しない場合は全 GitHub の Actions が対象になります。後述のサービスアカウント側の設定で権限借用の範囲を限定しておけば、どこからでも使われてしまうことにはなりませんが、ほとんどの場合は制限しておくほうがよいでしょう。

`attribute_mapping` では、GitHub から渡される情報を Workload Identity プールの属性にマップします。[GitHub Actions の OIDC トークンのドキュメント](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token)も参照してください。以下にトークンの一部形式を引用します。
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
ここから、権限借用の条件で使いたい属性をマップします。`attribute.org_repo_branch` のマッピングは少し奇妙に見えるかもしれないので補足すると、`assertion.repository` に入っている `<org>/<repo>` の文字列と、extract 関数で `assertion.ref` から `refs/heads/<branch>` を抽出した文字列を `/` で結合して`<org>/<repo>/<branch>` の文字列になるようにしています。後述のサービスアカウント側の権限借用の条件で使います。


## サービスアカウントの作成
以下の Terraform コードは 3 つのサービスアカウント `foo` `bar` `baz` (プレフィックス略) を作成し、前述の Workload Identity プールで認証されるプリンシパルに対して権限借用の条件を設定します。

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

resource "google_service_account_iam_binding" "github_actions_foo_wi_binding" {
  service_account_id = google_service_account.github_actions_foo.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "principal://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/subject/repo:${var.org_name}/example_githubactions_oidc:ref:refs/heads/main",
  ]
}

resource "google_service_account_iam_binding" "github_actions_bar_wi_binding" {
  service_account_id = google_service_account.github_actions_bar.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.org-repo-branch/${var.org_name}/example_githubactions_oidc/develop",
  ]
}

resource "google_service_account_iam_binding" "github_actions_baz_wi_binding" {
  service_account_id = google_service_account.github_actions_baz.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.environment/production",
  ]
}
```

`"google_service_account_iam_binding"` ではサービスアカウントごとに [許可ポリシー](https://cloud.google.com/iam/docs/overview?hl=ja#cloud-iam-policy)を設定しています。このポリシーで許可されたプリンシパルのみが権限借用を行うことができます。

上記の例では、それぞれ以下の意味になるよう許可ポリシーを設定しています。

- `foo` : `<org名>/example_githubactions_oidc` というレポジトリの `main` ブランチで実行される workflow 相当の単一のプリンシパルを許可
- `bar` : `<org名>/example_githubactions_oidc` というレポジトリの `develop` ブランチで実行される workflow 相当のプリンシパル集合を許可
- `baz` : Environemnt `production` で実行される workflow 相当のプリンシパル集合を許可

ここで指定しているプリンシパルは前述の Workload Identity プールから提供されるものなので、当然ながらプールの subject / 属性のマッピングによって文字列が変わります。マッピングには Common Expression Language 式を使うことができるので、`attribute.org-repo-branch` で指定しているような多少複雑なこともできます([属性のマッピング](https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja#mapping)参照)。


`foo` の指定はシンプルに単一のプリンシパルを許可しています。`repo:<org>/<repo>:ref:refs/heads/main` は `main` ブランチから実行された workflow に対応するプリンシパルを表します (workflow が Environment を使っていない場合)。

`bar` は、`<org>/<repo>/<branch>` という形になるよう整形した `attribute.org-repo-branch` を参照しています。もちろん、`foo` の場合と同様に `principal://(略..):refs/head/develop` のように指定することでもほぼ同じ意味の指定ができます。

`baz` は、`attribute.environment` を参照しています。これは Environment `production` で実行された workflow に対応するプリンシパル集合を表します。ここのプリンシパル指定にはレポジトリやオーナーの情報が含まれていないことに注意してください。`attribute.environment` が `production` であるプリンシパル全てが許可されます。プールの属性条件で Organization 名を制限しているため、結果的に `<org>` 以下すべてのレポジトリから Environment `production` で実行される全ての workflow に対応します。一部繰り返しになりますが、仮にプールの属性条件が指定されていなければ、このプリンシパル集合は GitHub のあらゆるレポジトリから Environment `production` で実行される全 workflow に対応します。

-------

## 関連
- IAM の概要 https://cloud.google.com/iam/docs/overview

- デプロイ パイプラインによるサービス アカウントの権限借用を許可する https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines?hl=ja#allow_the_deployment_pipeline_to_impersonate_the_service_account

- サービス アカウントの権限借用 https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja#impersonation
- 属性のマッピング https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja#mapping