---
title: "GitHub Codespaces 入門"
emoji: "☁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "Codespaces", "Cloud"]
published: false
---

# GitHub Codespaces 入門
2022/11 からすべての GitHub ユーザに開放された Codespaces の入門記事です

## その 0: とりあえず使う
とりあえず使ってみるだけなら GitHub アカウントだけあれば大丈夫。GitHub で、自分のものでも自分以外の公開レポジトリでもよいので、適当なレポジトリの Code ボタンから作成できる。すぐにブラウザで VSCode が立ち上がる。

![new codespace ss](https://storage.googleapis.com/zenn-user-upload/96a1b31b99ea-20221128.png)

Codespace はクラウド側に自分用のコンテナ環境を立ち上げて使うことができるサービス。自分が立ち上げた Codespace 環境の一覧は、GitHub 最上段(2022/11現在)にある [Codespaces リンク](https://github.com/codespaces)から一覧できる。

## その 1: 環境設定ファイル devcontainer.json を知る

`.devcontainer/devcontainer.json` のような設定ファイルを配置しておくことで Codespace 環境の詳細を設定できる。

特に重要と思われるポイントを挙げる。
- 環境はコンテナ。公開イメージを指定して使うほか、Dockerfile を指定してビルドしたり、compose.yaml で指定した複雑な環境を構築することもできる
  - 何も指定しなければデフォルトのコンテナが立ち上がる
- ライフサイクルフックがある。コンテナ生成後やエディタ接続後など各ポイントで任意のコマンドを実行できる
- [features](https://containers.dev/features) でコンテナイメージに含まれない機能を後づけできる^[多くはdebian系のベースイメージで利用可能]。
  - 例えば、イメージに `ruby:3.0` を指定しつつ Git だけは開発に必要なので追加したい、といった場合には `"features": { "ghcr.io/devcontainers/features/git:1": {} }` を加えるだけでよい
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

```json:devcontainer.json
  "dockerComposeFile": ["../compose.yaml"],
  "service": "(アタッチ対象にするcompose内のservice)",
  "runServices": ["(起動させるサービス)", "..."],
```
アタッチ対象の `service` は起動しつづけている必要がある。`runServices` は指定しなければ compose 内の全 service が対象になる。

`dockerComposeFile` は複数指定でき、compose 仕様によって後から指定したものが上書きマージされる。配列は追記になるので注意。

**参考**
- [Docker Compose な開発環境にちょい足し3分で作るVSCode devcontainer](https://zenn.dev/saboyutaka/articles/9cffc8d14c6684)
- [Multiple Compose files](https://docs.docker.com/compose/extends/#multiple-compose-files)


**別案**

compose を用意せずにより柔軟にコンテナ操作を行いたい場合は、[docker-in-docker features](https://github.com/devcontainers/features/tree/main/src/docker-in-docker) も活用できる

### プレビルドで起動を高速化する
