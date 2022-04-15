---
title: "kustomize の commonLabels が Headless Service のセレクタに干渉してしまうのを回避する"
emoji: "‼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes"]
published: false
---

## 何が問題なのか

Headless Service は `selector` の有無により、自動的に対応する Endpoints を構成するかどうかの挙動が違うが、kustormize の commonLabels を使っていると `selector` が挿入されて意図しない挙動になることがある。

つまり、自前で Endpoints を作って渡してあげたいのに、`selector` が挿入されることでその `selector` に基づいたセットで上書きされてしまう。

https://github.com/kubernetes-sigs/kustomize/issues/249 など。

## 対応策

前述の issue にもあるように、multiple base でやるか、jsonPatch6902 で挿入される `selector` を削除すればいい

```yaml:kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

commonLabels:
  app.kubernetes.io/name: myapp

#...(中略)

patchesJson6902:
- target:
    version: v1
    kind: Service
    name: myservice
  # prevent to add selector to headless service without selector
  patch: |-
    - op: remove
      path: /spec/selector
```