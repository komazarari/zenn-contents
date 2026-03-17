---
title: "OpenClaw を Lightsail でサクっと立ち上げ Spot インスタンスで安く稼動 (ただしたまに止まる)"
emoji: "🦞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Lightsail", "EC2", "Spot", "OpenClaw"]
published: true
---

:::message
本記事は EC2 経験者を想定しています
:::

AWS Lightsail を使うと OpenClaw をわずか数分でセットアップすることができます。この導入の手軽さを享受しつつ、運用を EC2 Spot インスタンスに移すことで安さと EC2 の柔軟性を活かせます。

ただし Spot インスタンスはおおざっぱに言えば「いつ止まるかわからない代わりに安い」というものですので、可用性を求めない用途での選択肢となります。

## 全体の流れ

1. Lightsail で OpenClaw インスタンスを起動
1. Lightsail インスタンスのスナップショットを作成、EC2 にエクスポート
1. EC2 で Spot インスタンスとして起動 (非 Spot も可能)
1. IAM ロール・ポリシーを設定する※

※ OpenClaw に Bedrock 以外の LLM プロバイダーを指定する場合は IAM 設定は必須ではありません

## Lightsail で OpenClaw インスタンスを起動

2026/03 より Lightsail でセットアップ済みの OpenClaw インスタンスを作成できるようになりました。[公式ブログ](https://aws.amazon.com/jp/blogs/news/introducing-openclaw-on-amazon-lightsail-to-run-your-autonomous-private-ai-agents/)や[テックブログ](https://dev.classmethod.jp/articles/amazon-lightsail-openclaw/)などを参考にしていただけると良いかと思います。数クリックで作成できます。

![コンソールからlightsailインスタンス起動](/images/lightsail-openclaw-to-ec2_launch-lightsail.webp)

起動させたら必要に応じ動作確認を行ない、インスタンスを停止します。

## Lightsail から EC2 へのエクスポート

Lightsail のインスタンスを停止したらスナップショット作成と EC2 へのエクスポートを行います。

### スナップショット作成

Lightsail コンソールからインスタンスのスナップショットを作成します。

![lightsailインスタンスのスナップショット作成](/images/lightsail-openclaw-to-ec2_snapshot-lightsail.webp)

### EC2 へエクスポート

スナップショットが作成できたら EC2 へエクスポートします。

![lightsailスナップショットをEC2へエクスポート](/images/lightsail-openclaw-to-ec2_export-snapshot-to-ec2.webp)

エクスポート完了次第 EC2 コンソールの AMI より確認できます。


## IAM ロールの作成

EC2 インスタンスから Bedrock にアクセスするための IAM ロールを作成します (後続手順のインスタンス起動時に作成することも可能です) 。

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

Invoke 系は文字通り LLM の呼び出し用、List/Get は OpenClaw のモデル自動ディスカバリ用です。

## EC2 インスタンス起動

### Spot インスタンスとして起動

Lightsail からもってきた AMI で EC2 を起動します。ここでは Spot インスタンスとして起動させます。
起動させるときにインスタンスに IAM プロファイルを指定し、先ほど作成した IAM ロールを使えるようにしてください。

Spot インスタンスを簡単に運用するために、Spot インスタンスオプションを以下のように指定します。
- リクエストオプションを Persistent (永続) に
- 中断時のふるまいを Stop (停止) に

![Spotインスタンスオプションでpersistent request,interuption behavior を stop に指定](/images/lightsail-openclaw-to-ec2_spot-instance-options.webp)

デフォルトの Spot インスタンスオプションでは Spot キャパシティ不足のときにインスタンスが破棄されてしまいますが、Persistent リクエストと中断時 Stop を指定することで、EBS や EIP の関連づけといったインスタンス状態を保持したままシャットダウンだけされる挙動にできます。キャパシティが復活したらまたインスタンスが起動してくれます。

任意の設定ですが、EBS がインスタンス削除時に消えないようにしたり、EIP を付与しておくのも良いかと思います。

### EC2 インスタンスの IAM 権限を使うよう構成変更

Lightsail は専用の別 AWS アカウントと連携するサービスです。そのため Lightsail 提供の OpenClaw イメージでは、ユーザのアカウントにロールを作成したうえで Lightsail インスタンスから assumeRole して Bedrock を使うような構成となっています。

今回は自アカウントの EC2 で起動するようにしましたので、assumeRole せずに EC2 自身の IAM を使うよう設定ファイルを変更します。

本記事執筆時点のイメージでは、`ubuntu` ユーザの AWS config (`/home/ubuntu/.aws/config`) に `[profile assumed]` が初期化スクリプトで生成され、これを systemd 管理の OpenClaw が環境変数 `AWS_PROFILE` によって読みにいくようになっていました。

```bash
$ grep AWS_PROFILE /opt/aws/open_claw/openclaw.env
AWS_PROFILE=assumed
```

この `AWS_PROFILE` をコメントアウトすることで、デフォルトの認証プロバイダチェーンにより EC2 のインスタンスプロファイルが読まれるようになります。

```bash
$ grep AWS_PROFILE /opt/aws/open_claw/openclaw.env
#AWS_PROFILE=assumed
```

インスタンスに正しく IAM ロールがアタッチされていることが前提になりますので、`sts getCallerIdentity` などで期待したロールになっているかを確認してください。

```bash
$ aws sts get-caller-identity # 例
{
    "UserId": "AROA2xxxxxxxxxxxxx999:i-0a1b2c3d4e5f6g7h8",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/OpenClawInstanceRole/i-0a1b2c3d4e5f6g7h8"
}
```

ここまでで EC2 としての設定は完了です。Lightsail の場合と同様にブラウザと SSH を使って OpenClaw を設定できます。

## コスト比較

同等スペックでの Lightsail と EC2 スポット、参考としてややハイスペックなメモリ重視インスタンスタイプでのコスト比較です（Tokyo リージョン、2026-03 時点）。

| サービス | サイズ/タイプ | vCPU | メモリ | 月額（目安） | 備考 |
| --- | --- | --- | --- | --- | --- |
| Lightsail | $24/月サイズ | 2 | 4 GB | $24 | ストレージ込み |
| EC2 オンデマンド | t3.medium | 2 | 4 GB | 約 $47 | 0.0544*24*30 + EBS  |
| EC2 スポット | t3.medium | 2 | 4 GB | 約 $19 |0.017*24*30 + EBS |
| EC2 スポット | r8i.xlarge | 4 | 32 GB | 約 $82 |0.1035*24*30 + EBS |

EBS は gp3 80GB で月額約 $7.68 ほどです。

Lightsail は停止しても課金が継続しますが、EC2 は起動時のみの課金です。常時稼働させるとそれほど EC2 Spot と Lightsail の価格差は出ませんが、使わないタイミングで止めておく運用もできるので、用途にあわせたコスト最適化が可能です。

私は夜間は基本使わないので、EventBrige Scheduler を使った StopInstance/StartInstance で自動停止させています。必要なときはモバイルの AWS アプリで操作したりもできます。

## まとめ

- Lightsail は手軽に初期セットアップが可能
- スナップショット、エクスポート機能を使うことで Lightsail からより柔軟性の高い EC2 へ移行できる
- EC2 スポットインスタンスは中断されることがあるが、この一時停止を許容できるなら永続リクエストで自動復旧可能

## 参考資料
- [自律型プライベート AI エージェントを実行するための OpenClaw が Amazon Lightsail に導入されました | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/introducing-openclaw-on-amazon-lightsail-to-run-your-autonomous-private-ai-agents/)
- [[アップデート] Amazon Lightsail で事前セットアップ済みの OpenClaw インスタンスが登場しました | DevelopersIO](https://dev.classmethod.jp/articles/amazon-lightsail-openclaw/)
- [Amazon Lightsail インスタンスを Amazon EC2 にエクスポートする](https://docs.aws.amazon.com/ja_jp/lightsail/latest/userguide/amazon-lightsail-exporting-snapshots-to-amazon-ec2.html)
- [スポットインスタンスの中断 - Amazon EC2](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/spot-interruptions.html)
