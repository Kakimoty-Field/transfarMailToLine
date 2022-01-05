### 更新履歴
- 2022/01/05 Lambda を作成するリージョンについて追記

# 目次
<!-- code_chunk_output -->

- [目次](#目次)
- [作るもの](#作るもの)
  - [前提条件](#前提条件)
- [背景](#背景)
  - [やりたいこと](#やりたいこと)
  - [課題](#課題)
  - [解決方法](#解決方法)
  - [この仕組みのメリット](#この仕組みのメリット)
- [実装](#実装)
  - [LINE](#line)
  - [Amazon Route53 (ドメイン取得)](#amazon-route53-ドメイン取得)
    - [ドメイン名の選択](#ドメイン名の選択)
    - [登録者の連絡先入力と購入](#登録者の連絡先入力と購入)
  - [Amazon SES](#amazon-ses)
    - [リージョンの選択](#リージョンの選択)
    - [Domain Identity 作成](#domain-identity-作成)
    - [メール受信設定とS3連携](#メール受信設定とs3連携)
      - [受信メールアドレス登録](#受信メールアドレス登録)
      - [S3 への保存設定](#s3-への保存設定)
      - [動作確認](#動作確認)
  - [Amazon S3](#amazon-s3)
    - [ライフサイクルルール作成](#ライフサイクルルール作成)
  - [AWS Lambda](#aws-lambda)
    - [モジュールレイヤーの作成](#モジュールレイヤーの作成)
      - [LINE Messaging API SDK for nodejs](#line-messaging-api-sdk-for-nodejs)
      - [MAILPARSER](#mailparser)
      - [結果確認](#結果確認)
    - [Lambda 関数作成](#lambda-関数作成)
    - [ソースコードデプロイ](#ソースコードデプロイ)
    - [環境変数設定](#環境変数設定)
    - [トリガー追加](#トリガー追加)
    - [テスト](#テスト)
    - [応用](#応用)

<!-- /code_chunk_output -->


# 作るもの
構成図は以下の通りです。

構成図では、メールサービスが **Gmail** になっていますが、転送設定できるメールサービスであればどんなメールサービスでも LINE に転送できます。

![構成図](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/000.png)

1. LINE に通知したいメールサービスで、AmazonSES に対してメール転送設定します
1. AmazonSES のメールアドレスは Amazon Route 53 で解決されます。
1. AmazonSES で受信したメールは Amazon S3 バゲットに格納されます。
1. S3 トリガー呼び出しで、LINE メッセージを送信する AWS Lambda 関数を呼び出します。

## 前提条件
- 独自ドメインを取得します (場合によってはお金がかかる)
- DNS は Amazon Route53 を利用します
- Amazon SES から 直接 AWS Lambda をコールせずに、一度 Amazon S3 にメールデータを格納します。(SES から Lambda を呼び出すと、メール本文データが取得できなかったため)
- Lambda 関数のランタイムは `Node.js` です。
- LINE bot は友だち登録します。
- LINE に通知されるメールは、友だち登録している全員に送信されます。

# 背景

## やりたいこと
受信したメールのうち、特定のメールを LINE に転送したい。

## 課題
- 転送したいメールは Gmail
- Gmail はメールを受信したときに発火する トリガー の仕組みが無い（？）
- ポーリングはしたくない

## 解決方法
Amazon SES にはメール受信時のトリガーがあるので、Amazon SES にメールを転送して Lambda から LINE に転送する。

## この仕組みのメリット
Gmail だけでなく、メール転送設定できるメールサービスであれば **メールを LINE に転送する** という汎用的な仕組みが構築できる。

# 実装

## LINE

[LINE Developers](https://developers.line.biz/ja/) で、**Messaging API** のチャンネルを作成します。

チャンネルを作成したら、下記項目をコピーしておきます。

- Basic settings メニューの **Channel secret**
- Messaging API Settings メニューの **Channel access token (long-lived)**

また、**Messaging API settings** メニューの **QR code** を利用して、このチャンネルと友だちになっておきます。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/line000.png)

## Amazon Route53 (ドメイン取得)
(※すでに独自ドメインを持っている場合は、このステップは飛ばします)

### ドメイン名の選択
AWSマネージメントコンソール から **Amazon Route53** を選択し「ドメインの登録」ボタンをクリックしウィザードを開始します。  

**ドメイン名の選択** 画面で、利用したいドメイン名の入力とトップレベルドメインを選択します。
![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-route53/100.png)

**利用可能**であれば、「カートに入れる」ボタンをクリックし、画面最下部の「続行」ボタンをクリックします。

### 登録者の連絡先入力と購入
**１ ドメインのお問い合わせ詳細** 画面で、ドメインの登録者・管理者の情報を正しく入力します。トップレベルドメインによっては、ここで入力された情報が公開されないので安心です。

すべての情報が入力し終わったら画面最下部の「続行」ボタンをクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-route53/110.png)

**連絡先の詳細確認** 画面で、前画面で入力した情報の内容を確認します。また、「１年毎のドメイン自動更新」を有効にするかどうかを設定できるので、適切に設定します。

確認が完了したら画面最下部の「注文を完了」ボタンをクリックします。
![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-route53/120.png)

**AWSクレジット** を保有している場合、下記のダイアログが表示されます。ドメイン登録にかかる費用はAWSクレジットは利用できず、実費が請求されます。内容を確認したうえで「注文を完了」ボタンをクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-route53/130.png)

Route 53 ダッシュボードの **通知** 欄に、ドメイン登録の進捗状況が表示されます。ドメインが利用可能になるまで３０分前後かかるかと思います。

## Amazon SES
(※SES の設定は2021年5月上旬に試したため、旧コンソールに誘導されました。旧画面のスクショでの手順となります。)

### リージョンの選択
**AmazonSES でメールを受信できるリージョンは限定されています。** 

下記リージョンのいずれかを選択して作業します。

| リージョン名 | リージョン   |
| --- | --- |
| US East (N. Virginia)| us-east-1 |
| US West (Oregon) | us-west-2 |
| Europe (Ireland) | eu-west-1 |

[公式ドキュメント(Amazon Simple Email Service endpoints and quotas)](https://docs.aws.amazon.com/general/latest/gr/ses.html)

### Domain Identity 作成
Amazon SES で 自分のドメインを利用できるように Identity を作成します。画面左ペインの **Identity Management** 内の **Domains** をクリックします。（図内①）

そのあと、画面上部の「Verify a New Domain」ボタンをクリックします。(図内②)

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/100.png)

**Verify a New Domain** ダイアログが表示されるので、メールアドレスとして利用するドメインを入力して「Verify This Domain」ボタンをクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/110.png)

Route 53 を利用して `MXレコード` を登録するかどうか確認して「Use Route 53」ボタンをクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/120.png)

しばらく待つと、登録した `Domain Identity` の Verification Status が **verified** になります。

### メール受信設定とS3連携

#### 受信メールアドレス登録
Amazon SES でメールを受信し、受信したメールを Amazon S3 に保存するルールを作成します。

画面左ペインの **Email Receiving** 内の **Rule Sets** をクリックします。（図内①）

そのあと、画面上部の「Create a Receipt Rule」ボタンをクリックします。(図内②)
![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/130.png)

メールボックスに相当するものを作成します。ここで何も入力しないと、サブドメインを含めたありとあらゆるユーザに対してのメールを受信することになりますので、転送先として適切なメールアドレスを作成しておきます。

下記例では、`test-mail@ドメイン名` というメールアドレスを有効にしています。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/140.png)

メールアドレスを入力したら画面下部の「Next Step」ボタンをクリックします。

#### S3 への保存設定
**Actions** 画面で、受信したメールをどのように取り扱うかを設定します。

**Add action** メニューの `<Select an action type>` で `S3` を選択します。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/150.png)

**Amazon S3** の設定をする画面に切り替わるので **S3 bucket** メニュをクリックし `Create S3 bucket` を選択します。
(既存のバケットを利用したい場合、ドロップダウンリストの下部に利用可能なバケットがリストされているので、そちらを選択してください。)

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/160.png)

`Create S3 bucket` を選択すると**Create New Bucket** ダイアログが表示されるので、世界でユニークとなる名前を付けて「Create Bucket」ボタンをクリックします。

:::note info
Amazon SES を利用しているリージョンに S3バゲットが作成されます
:::

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/170.png)



続いて **Object key prefix** に、受信したメールを格納するフォルダを入力します。
入力がおわったら画面下部の「Next Stop」ボタンをクリックします。
![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/180.png)

**Rule Details**画面で、**Rule name** を入力して、「Next Step」ボタンをクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/190.png)

**Review**画面では、いままで入力した設定内容を確認します。確認が終わったら「Create Rule」ボタンをクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/200.png)

#### 動作確認
正しくメールを受信し、S3にメールが保存されるかどうかを確認します。

[受信メールアドレス登録](#受信メールアドレス登録) で入力しSESで有効にしたメールアドレスにメールを送信します。

これまでの設定が正しい場合、メールを送信すると[S3 への保存設定](#s3-への保存設定)で設定したバケット内のフォルダにオブジェクトが作成されます。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/aws-ses/210.png)

## Amazon S3
この仕組みを運用していくとバケット内にオブジェクトが増え続けます。メールをＬＩＮＥに通知したらオブジェクトが不要となるので、自動でオブジェクトが削除されるように設定してコスト削減を図ります。

### ライフサイクルルール作成
Amazon S3 の該当バケットを選択し、**管理**タブをクリックしたあと「ライフサイクルルールを作成する」ボタンをクリックします。
![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/S3/100.png)

下記の設定をして、オブジェクトが作成された翌日に削除されるようにします。

1. **ライフサイクルルール名**は任意の文字を入力します。
1. **プレフィックス**は [S3 への保存設定](#s3-への保存設定) で入力した **Object key prefix** を入力します。
1. **ライフサイクルルールのアクション**は `オブジェクトの現行バージョン有効期限が切れる`にチェックします。
1. **オブジェクトの現行バージョンの有効期限が切れる**に `1` を入力します。
1. 「保存」ボタンをクリックします。


![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/S3/110.png)

## AWS Lambda
### モジュールレイヤーの作成

AWS Lambda の Node.js ランタイムで npm モジュールを利用するため、レイヤーを作成します。
[ドキュメント-Lambda レイヤーの作成と共有](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/configuration-layers.html)

**※以下は AWS CloudShell で作業します。**
[ドキュメント-CLI aws lambda publish-layer-version](https://docs.aws.amazon.com/cli/latest/reference/lambda/publish-layer-version.html)

#### LINE Messaging API SDK for nodejs
[github - line-bot-sdk](https://github.com/line/line-bot-sdk-nodejs)

```
#
# AWS ドキュメントに沿ったパス作成
$ mkdir nodejs
$ cd nodejs

#
# パッケージインストール
$ npm install @line/bot-sdk --save

#
# レイヤー用 Zip ファイル作成
$ cd ..
$ zip -r layer1.zip ./nodejs

#
# レイヤー作成
aws lambda publish-layer-version \
 --layer-name line-bot-sdk \
 --description "Line Bot for Node.js" \
 --zip-file fileb://./layer1.zip \
 --compatible-runtimes nodejs12.x nodejs14.x

# {結果のJSON が出力される}

#
# 後片付け
$ rm -rf nodejs/
$ rm layer1.zip
```

#### MAILPARSER
[OfficialPage - MAILPARSER](https://nodemailer.com/extras/mailparser/)

```
#
# AWS ドキュメントに沿ったパス作成
$ mkdir nodejs
$ cd nodejs

#
# パッケージインストール
$ npm install mailparser --save

#
# レイヤー用 Zip ファイル作成
$ cd ..
$ zip -r layer2.zip ./nodejs

#
# レイヤー作成
aws lambda publish-layer-version \
 --layer-name mailparser \
 --description "mailparser for Node.js" \
 --zip-file fileb://./layer2.zip \
 --compatible-runtimes nodejs12.x nodejs14.x

# {結果のJSON が出力される}

#
# 後片付け
$ rm -rf nodejs/
$ rm layer1.zip
```

#### 結果確認
AWS マネージドコンソールで、作成されたレイヤーを確認します。

**AWS Lambda** コンソールの左ペインから **その他のリソース**→**レイヤー**をクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/230.png)

２つのレイヤーが作成されていることを確認します。

### Lambda 関数作成

:::note warn
lambda は、S3 と同じリージョンで作成する必要があります。
:::

**AWS Lambda** ダッシュボード右上の「関数の作成」ボタンをクリックします。
![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/300.png)


**関数の作成** 画面では、以下の項目を設定します。

- １から作成 を選択します。
- 関数名は任意の文字列を入力します。
- ランタイムは `Node.js 14.x` を選択します。

設定が完了したら、画面下部の「関数の作成」ボタンをクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/310.png)

**lambda 関数設定画面**の最下部、 **レイヤー** グループの「レイヤーの追加」ボタンをクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/320.png)

**レイヤーを追加**画面では、`カスタムレイヤー`を選択し、ドロップダウンリストから `line-bot` と `mailperser` を追加します。(２つのレイヤーを追加する際は、1つ目のレイヤーを選んだあと、「追加」ボタンをクリックして改めて2つ目のレイヤーを追加します。)

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/330.png)

### ソースコードデプロイ

下記ソースコードでS3に格納されたメールデータをLINEに通知することができます。

**※注意※**
`lineClient.broadcast()` メソッドは `Promise` を返す非同期処理となっています。`await` を付けずに Lambda 関数を抜けてしまうと送信エラーになります。

(ローカル環境では、`await`無しでもうまくいったソースコードが、なぜか Lambda で動かずにしばらく悩みました。)

```
console.log('Loading function');
const aws = require('aws-sdk');
const s3 = new aws.S3({ apiVersion: '2006-03-01' });

const simpleParser = require('mailparser').simpleParser;
const line_bot_sdk = require('@line/bot-sdk');

// LINE 接続情報（ハンドラ外で定義
var lineClient = new line_bot_sdk.Client(
{
    channelAccessToken : process.env.LINE_CHANNEL_ACCESS_TOKEN,
    channelSecret : process.env.LINE_CHANNEL_SECRET
});

exports.handler = async (event, context) => {
    //console.log('Received event:', JSON.stringify(event, null, 2));

    const bucket = event.Records[0].s3.bucket.name;
    const key = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, ' '));
    const params = {
        Bucket: bucket,
        Key: key,
    };
    try {
        const { Body, ContentType } = await s3.getObject(params).promise();
        const mailBody = await simpleParser(Body);
        
        // Line 送信
        await lineClient.broadcast (
        {
            type: 'text',
            text : `件名 : ${mailBody.subject} \n` +
                   `内容 : ${mailBody.text}` ,
        });
        return ContentType;
        
    } catch (err) {
        console.log(err);
        const message = `Error getting object ${key} from bucket ${bucket}. Make sure they exist and your bucket is in the same region as this function.`;
        console.log(message);
        throw new Error(message);
    }
};

```

### 環境変数設定
LINE にメッセージを送信するためのアクセストークンとチャンネルシークレットを環境変数に設定します。
画面中段部分の **設定**タブをクリックし **環境変数** メニューをクリックします。

環境変数入力欄には、下記の変数を設定します。

| キー | 値 |
| --- | --- |
| LINE_CHANNEL_ACCESS_TOKEN | アクセストークン |
| LINE_CHANNEL_SECRET | チャンネルシークレット |

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/100.png)

### トリガー追加

Amazon S3 にオブジェクトが作成されたら Lambda 関数が実行されるように設定します。

画面上部の **関数の概要**から「＋トリガーを追加」ボタンをクリックします。
![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/200.png)

**トリガーを追加**画面の、**トリガーを設定**ドロップダウンボックスに `S3` と入力すると リスト内にＳ３が表示されるので選択します。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/210.png)

**トリガーの設定** 画面では、以下の項目を設定します。

- バケットには[SESで受信したメールを保存するバケット](#s3-への保存設定)を設定します。
- イベントタイプは `PUT` を選択します。
- プレフィックス・オプションは [Object key prefix](#s3-への保存設定)を設定します。
- 再帰呼び出し の注意ボックスに表示されている内容を読み、確認チェックボックスにチェックします。

設定が完了したら、画面最下部の「追加」ボタンをクリックします。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/220.png)

### テスト

[SES でのメール受信動作確認](#動作確認) で作成されたオブジェクトを、名前を変えてS3バケットに保存してみましょう。 Lambda 関数が正しく実行されると、LINE に通知が送信されます。

![](https://raw.githubusercontent.com/Kakimoty-Field/transfarMailToLine/master/img/lambda/900.png)

### 応用
Lambda 関数ソースコード内でメール件名や本文をチェックして、ルールに当てはまるメールだけをLINE に通知するという機能を作ることも可能かと思います。
