---
title: "GitHub Codespaces 入門"
emoji: "☁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "Codespaces", "Cloud"]
published: false
---

# GitHub Codespaces 入門
GitHub Codespaces はクラウド上で専用の開発環境を利用できるサービスです。
2022/11 からすべての GitHub ユーザが無料で一定時間使用できるようになりましたので簡単に使い方を紹介します。

## その 0: とりあえず使う
とりあえず使ってみるだけなら GitHub アカウントだけあれば大丈夫。GitHub で、自分のものでも自分以外の公開レポジトリでもよいので、適当なレポジトリの Code ボタンから作成できる。すぐにブラウザで VSCode が立ち上がる。

![new codespace ss](https://storage.googleapis.com/zenn-user-upload/96a1b31b99ea-20221128.png)

Codespace はクラウド側に自分用のコンテナ環境を立ち上げて使うことができるサービス。自分が立ち上げた Codespace 環境の一覧は、(2022/11現在)GitHub 最上段にある [Codespaces リンク](https://github.com/codespaces)から一覧できる。

## その 1: 環境設定ファイル devcontainer.json を知る

`.devcontainer/devcontainer.json` のような設定ファイルを配置しておくことで Codespace 環境の詳細を設定できる。

ポイントになる設定項目をピックアップすると
- 環境はコンテナ。公開イメージを指定して使うほか、Dockerfile を指定してビルドしたり、compose.yaml で複雑な環境を構築することもできる
  - 何も指定しなければデフォルトのコンテナが立ち上がる
- ライフサイクルフックがある。コンテナ生成後やエディタ接続後など各ポイントで任意のコマンドを実行できる
- [features](https://containers.dev/features) でコンテナイメージに含まれない機能を後づけできる^[基本的にはdebian系のベースイメージであれば利用可能。busybox/alpine系向けも探せばあるのかもしれないが…]。
  - 例えば、イメージに `ruby:3.1-buster` を指定しつつ Git だけは開発に必要なので追加したい、といった場合には `"features": { "ghcr.io/devcontainers/features/git:1": {} }` を加えるだけでよい
- インストールする VSCode 拡張機能を指定できる。どのユーザに対しても同じ拡張機能が準備された状態にできる。

[Codespaces テンプレート](https://github.com/codespaces/templates)があるので、それらを参考にするとわかりやすい。
- https://github.com/github/codespaces-rails/blob/main/.devcontainer/devcontainer.json
- https://github.com/github/codespaces-react/blob/main/.devcontainer/devcontainer.json
- https://github.com/github/codespaces-express/blob/main/.devcontainer/devcontainer.json

devcontainer 自体はもともと VSCode 拡張の設定だったのが、今は VSCode から切り離されて独立した仕様になっている。Codespaces 用の設定ファイルもコレに倣う。
- https://containers.dev/
- https://github.com/devcontainers/spec

なお、devcontainer 設定の不備などにより構築に失敗した場合はデフォルトのコンテナ環境にフォールバックして起動される。

## その 2: 環境をもっとカスタマイズする

### 複数のコンテナを使う
例えばアプリケーションサーバと開発 DB のような、メインの開発対象コンテナ+依存する別サービスがあるような場合。このような場合は compose (docker-compose) で設定すると環境設定しやすい。

compose 環境を普通に設定し、devcontainer.json から参照する。

```json:devcontainer.json 内
  ...
  "dockerComposeFile": ["../compose.yaml"],
  "service": "(アタッチ対象にするcompose内のservice)",
  "runServices": ["(起動させるサービス)", "..."],
  ...
```
アタッチ対象の `service` は起動しつづけている必要がある。

`runServices` は起動させるserviceを明示的に選別したいときに指定する。例えばコマンド実行専用や中間定義のserviceといった起動不要のものがある場合にそれらを除外し常駐させるものだけを書けばOK。`runServices` を指定しない場合はデフォルトで compose 内の全 service が起動対象になる。

`dockerComposeFile` は複数指定でき、compose 仕様によって後から指定したものが上書きマージされる。値が配列の場合は後から指定した要素がどんどん追加されていくので注意。

**参考**
- [Docker Compose な開発環境にちょい足し3分で作るVSCode devcontainer](https://zenn.dev/saboyutaka/articles/9cffc8d14c6684)
- [Multiple Compose files](https://docs.docker.com/compose/extends/#multiple-compose-files)


**複数コンテナの別案**

compose を用意せずにより柔軟にコンテナ操作を行いたい場合は、[docker-in-docker features](https://github.com/devcontainers/features/tree/main/src/docker-in-docker) も活用できる

### プリビルドで起動を高速化する

Codespace の起動に時間がかかってくる場合は、プリビルドを検討する。
https://docs.github.com/en/codespaces/prebuilding-your-codespaces/about-github-codespaces-prebuilds によると、Codespace 作成に2分以上かかるならプリビルドを使うと良さそうである。

プリビルドは、イメージのビルドを含むライフサイクルの一部をあらかじめ済ませておける機能である。Codespace 起動までを簡単に書くと

- レポジトリの clone (ホストVM)
- `initializeCommand` (ホストVM)
- イメージの取得/ビルド (ホストVM)
- コンテナ起動
- `onCreateCommand` (コンテナ内、以降コンテナ内)
- `updateContentCommand`
- `postCreateCommand`
- `postStartCommand`
- `postAttachCommand`

のような順番で実行される。プリビルドでは `updateContentCommand` までのステップを事前に実行したものを保存しておける。

**参考**
- [VSCode拡張機能 Remote Containers におけるpostCreateCommandなどの実行タイミングについて](https://vlike-vlife.netlify.app/posts/vscode_remote_container_command)
- [Dev Container metadata reference # Lifecycle scripts](https://containers.dev/implementors/json_reference/#lifecycle-scripts)

---
プリビルドを設定するとき、デフォルトでは全リージョン向けにビルドを作成するようになっている。必要なければ対象リージョンを減らしておくとストレージ枠の節約になる。
![prebuildconfig](https://storage.googleapis.com/zenn-user-upload/cf2b1bdd7810-20221129.png)

### dotfiles を自動インストールする
dotfiles 用レポジトリを作成しておくと、Codespaces 環境へ個人設定を自動インストールさせることもできる。個人の `Settings -> Codespaces -> Dotfiles -> [] Automatically install dotfiles` から設定する。(VSCode Remote Containers にある dotfiles の機能と同じっぽい?)

レポジトリ名は `dotfiles` の必要はなく任意の名前でよい。レポジトリ中に `install.sh` や `setup.sh` といった名前のスクリプトを置いておくと自動で実行してくれる。詳細は [Personalizing GitHub Codespaces for your account # Dotfiles](https://docs.github.com/en/codespaces/customizing-your-codespace/personalizing-github-codespaces-for-your-account#dotfiles) を参照。

## その 3: 外側の環境へのアクセス
### 機密情報を扱う

Codespace 内で秘密鍵やアクセストークンのような機密情報が必要な場合は、Secret を追加して使う。個人の [`Settings -> Codespaces -> Codespaces secrets`](https://github.com/settings/codespaces) から追加できる。 Secret ごとに、アクセスできるレポジトリを明示的に指定する必要がある。(1つづつポチポチ追加する必要があるようだ)

追加した Secret は Codespace 内から環境変数としてアクセスできる。

### 別のレポジトリにアクセスする

Codespace を作成したレポジトリ以外の他のレポジトリにアクセスしたい場合は、devcontainer 設定でアクセス権を付与すると Codespace 内でトークンを使ってアクセスできるようになる。トークンは Codespace 内の環境変数として設定される^[トークンを使うには GitHub CLI を使うと簡単。そのまま `gh repo clone` などでアクセス可能]。

https://docs.github.com/ja/codespaces/managing-your-codespaces/managing-repository-access-for-your-codespaces

```json:devcontainer.json 内
  ...
  "customizations": {
    "codespaces": {
      "repositories": {
        "my_org/my_repo": {
          "permissions": "write-all"
        }
      }
    }
  }
  ...
```

もちろん、自分がアクセス権を持っている必要がある。レポジトリアクセス設定のある Codespace を新規作成するときは、レポアクセスを求める認可画面が表示される。

### プライベートネットワークと疎通させたい
Codespace は通常 Azure VM のネットワークで稼働するので、プライベートネットワークへの直接疎通はできない。
GitHub CLI の拡張機能 [`gh-net`](https://github.com/github/gh-net#codespaces-network-bridge) をつかうと、ローカル PC をブリッジしてプライベートネットワークへのアクセスができるらしい、が、自分の環境では動作しなかった。2022/11 現在まだプレビュー版なので、将来的には Codespace から社内ネットワークに制限されたリソースへのアクセス、などができるかもしれない。

ほかの方法としては、VPN 使ってみる、などが挙げられている。
https://docs.github.com/en/codespaces/developing-in-codespaces/connecting-to-a-private-network