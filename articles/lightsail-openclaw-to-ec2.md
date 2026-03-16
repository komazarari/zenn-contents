---
title: "OpenClaw を Lightsail でサクっと立ち上げ Spot インタンスで割安稼動 (ただしたまに止まる)"
emoji: "🦞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Lightsail", "EC2", "Spot", "OpenClaw"]
published: false
---

本記事は EC2 経験者を想定しています。

AWS Lightsail を使うと [OpenClaw をわずか数分でセットアップ](https://aws.amazon.com/jp/blogs/news/introducing-openclaw-on-amazon-lightsail-to-run-your-autonomous-private-ai-agents/)することができます。この導入の手軽さを享受しつつ、運用を EC2 Spot インスタンスに移すことで割安なコストと EC2 の柔軟性を活かせます。

ただし Spot インスタンスはおおざっぱに言えば「いつ止まるかわからない代わりに安い」というものですので、可用性を求めない用途での選択肢と考えてください。

## 全体の流れ

1. Lightsail で OpenClaw インスタンスを起動
1. Lightsail インスタンスのスナップショットを作成、EC2 にエクスポート
1. EC2 で Spot インスタンスとして起動 (非 Spot も可能)
1. IAM ロール・ポリシーを設定する

## Lightsail で OpenClaw インスタンスを起動

2026/03 より Lightsail でセットアップ済みの OpenClaw インスタンスを作成できるようになっています。
[公式ブログ](https://aws.amazon.com/jp/blogs/news/introducing-openclaw-on-amazon-lightsail-to-run-your-autonomous-private-ai-agents/)や[テックブログ](https://dev.classmethod.jp/articles/amazon-lightsail-openclaw/)などを参考にしていただけると良いかと思います。数クリックで作成できます。

![コンソールからlightsailインスタンス起動](/images/lightsail-openclaw-to-ec2_launch-lightsail.webp)

起動させたら必要に応じ動作確認を行ない、インスタンスを停止して OK です。

## Lightsail から EC2 へのエクスポート

Lightsail のインスタンスを停止したら、スナップショットの作成と EC2 へのエクスポートを行います。

### スナップショット作成

Lightsail コンソールからインスタンスのスナップショットを作成します。

![lightsailインスタンスのスナップショット作成](/images/lightsail-openclaw-to-ec2_snapshot-lightsail.webp)

### EC2 へエクスポート

スナップショットが作成できたら EC2 へエクスポートします。

![lightsailスナップショットをEC2へエクスポート](/images/lightsail-openclaw-to-ec2_export-snapshot-to-ec2.webp)

エクスポートが完了したら EC2 コンソールの AMI より確認できます。


## IAM ロールの作成

EC2 インスタンスから Bedrock にアクセスするための IAM ロールを作成します。IAM コンソールでロールを作成しても、後続手順のインスタンス起動時に作成してもどちらでも構いません。 

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowBedrockInvoke",
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Sid": "AllowBedrockList",
            "Effect": "Allow",
            "Action": [
                "bedrock:List*",
                "bedrock:Get*"
            ],
            "Resource": "*"
        }
    ]
}
```

Invoke 系は文字通り LLM モデルを使うため、List/Get は OpenClaw のモデル切り替え用の自動ディスカバリのために付けています。

## EC2 インスタンス起動

### Spot インスタンスとして起動

Lightsail からもってきた AMI で EC2 を起動します。ここでは Spot インスタンスとして起動させます。

起動させるときにインスタンスには IAM プロファイルを指定し、先ほど作成した IAM ロールを使えるようにしてください。

Spot インスタンスを簡単に運用するために、Spot インスタンスオプションを以下のように指定します。

- リクエストオプションを Persistent (永続) に
- 中断時のふるまいを Stop (停止) に

![Spotインスタンスオプションでpersistent request,interuption behavior を stop に指定](/images/lightsail-openclaw-to-ec2_spot-instance-options.webp)

デフォルトの Spot インスタンスオプションでは Spot キャパシティ不足のときにインスタンスが破棄されてしまいますが、Persistent リクエストと中断時の Stop を指定することで、EBS や EIP の関連づけなどインスタンスの状態を保持したままシャットダウンだけされる挙動にできます。キャパシティが復活したらまたインスタンスが起動してくれます。

任意の設定として、データ領域の EBS がインスタンス削除時に消えないようにしたり、EIP を付与しておくのも良いかと思います。

### EC2 インスタンスの IAM 権限を使うよう構成変更

Lightsail は専用の別 AWS アカウントと連携するサービスです。Lightsail が用意している OpenClaw イメージでは、ユーザのアカウントにロールを作成して Lightsail インスタンスから assumeRole して Bedrock を使うような構成となっています。

今回は自アカウントの EC2 で起動するようにしましたので、assumeRole せずに EC2 自身の IAM を使うよう設定ファイルを変更します。

設定は 2026-03 現在のイメージでは、`ubuntu` ユーザの AWS config (`/home/ubuntu/.aws/config`) に `[profile assumed]` が記載してあって、これを systemd 管理の OpenClaw が環境変数設定によって読みにいくようになっていました。


## コスト比較

同等スペックでの Lightsail と EC2 スポットのコスト比較です（東京リージョン、2025年時点の概算）。

| サービス | プラン/タイプ | vCPU | メモリ | 月額（目安） |
| --- | --- | --- | --- | --- |
| Lightsail | $x.xx/月プラン | x | x GB | $x.xx |
| EC2 オンデマンド | t3.small | 2 | 2 GB | 約 $xx |
| EC2 スポット | t3.small | 2 | 2 GB | 約 $x〜$xx |

<!-- 実際の数値を記載 -->

スポットインスタンスはオンデマンド比で最大 90% 割引になるケースもあり、Lightsail と比べても十分に競争力のある価格になることが多いです。ただし、スポット価格は需要によって変動することに注意してください。

:::message
常時稼働が必要でなく、停止・起動を許容できる用途であれば、スポットをさらに活用（使わない時間は停止）してコストを下げることもできます。
:::

## まとめ

- Lightsail は初期セットアップが手軽で、プロトタイプや動作確認に最適
- スナップショットのエクスポート機能で、Lightsail から EC2 への移行が比較的簡単にできる
- EC2 スポットインスタンスを使うことで Lightsail より低コストでの運用が可能
- スポット中断への対策として、データの外部化と中断通知のハンドリングを実装しておくと安心

## 参考資料
- [自律型プライベート AI エージェントを実行するための OpenClaw が Amazon Lightsail に導入されました | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/introducing-openclaw-on-amazon-lightsail-to-run-your-autonomous-private-ai-agents/)
- [Amazon Lightsail インスタンスを Amazon EC2 にエクスポートする](https://docs.aws.amazon.com/ja_jp/lightsail/latest/userguide/amazon-lightsail-exporting-snapshots-to-amazon-ec2.html)
- [スポットインスタンスの中断 - Amazon EC2](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/spot-interruptions.html)
