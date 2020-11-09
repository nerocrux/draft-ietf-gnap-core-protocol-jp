# 0. GNAP

https://www.ietf.org/archive/id/draft-ietf-gnap-core-protocol-00.html

# 1. Introduction

* GNAP は Grant Negotiation and Authorization Protocol の略。
* 権限をソフトウェアに移譲するためのメカニズムを定義する。この権限の移譲は、特定セットのAPIにアクセスするためのものだけでなく、情報をソフトウェアに直接に渡す仕組みも含まれる。
* 似たようなユースケースを解決するために、OAuth 2.0 や OIDC がすでにありますが、GNAP は OAuth 2.0 の拡張ではないし、互換性でもありません。GNAP は OAuth 2.0 が解決しづらい、あるいは明確に解決できてないユースケースのソリューションを模索します。

## 1.1 ロール（Roles）

* Authorization Server (AS): 認可サーバ。ROに対するdelegationを管理し、RCにトークンを発行します。Grant エンドポイントを持ち、JSON payload の HTTP POST リクエストを受け付けます。その他に、Interaction エンドポイントと User Code エンドポイントも持っています。
* Resource Client (RC / "client"): ASからトークンを取得し、RSにトークンを利用します。ユニークキーを持ち、ASから識別され、特定の認可ポリシーが適用されます。
* Resource Server (RS / "API"): RCからトークンを受付、ROの代理として、許可されたリソースの提供を行います。
* Resource Owner (RO): RCからRSへのリクエストを許可する側で、ASとインタラクティブします。
* Requesting Party (RQ / "user"): RCを操作するユーザで、RCとインタラクティブします。

RQとRSは基本的に同じ"人"ですが、ロール的には異なります： ROはリソースの所有者で、RQはリソースを要求する人。[CIBA](https://openid.net/tag/ciba/)のようなユースケースだと、ROとRQは異なる人物の可能性もあるため、このように定義を分離したと筆者が理解しています。

GNAPはロールを定義するだけで、そのロールはどうやって実現されるかは問いません。一つのサーバは複数のロールを担うことも可能になっています。例えば、とあるサービスはRSとして動作すると同時に、RCとしてdownstreamなサービスにリクエストを送ることも可能になります。

各ロールは一つのモノリシックなサービス上にデプロイされる必要がなく、複数のコンポネントから構成され分散サービスにデプロイされることも可能です。例えばASはいろんなことやるし、集める identity claims は様々なサービスに散らばっている可能性が高いので、該当するサービスは一つにまとめる必要がなく、合わせてASと呼べば良いです。

## 1.2 構成要素（Elements）

* Access Token: RCに対する権限移譲のためのトークンで、ASがRCに対して発行し、RSが検証します。
* Grant: RO同意の元に、RCがRSに対するアクセス権限の要求をASが移譲するプロセスです。
* Key: 所持者とリクエストを紐づく暗号要素。アクセストークンやRCインスタンスとも紐付けます。
* Resource: RSが提供する保護されたAPI。RCからアクセスされます。
* Subject Information: RCからRSに対して呼び出すことなく、ASからRCに直接に返すROに関する情報。これもGrantプロセスの一部である（ROからの権限移譲が必要）。

## 1.3 シーケンス（Sequences）

すべてのロールは以下の図のように接続する必要がなく、同じエンティティが複数のロールを担当し、その間の通信が省かれることもありえます。
また、全てのステップを必ず全部実行されるわけではないです。例えばRCがsubject informationを取得するだけであれば、RSを呼び出す必要がないので、それに関連するすべてのステップが実施されなくても良いです。

![1.png](resources/35C4A8D86D33512E5A5F9774C2BDC452.png =715x484)

OAuth2の主なユースケースはウェブ上のリダイレクトベースだと思われますが、GNAPはそれだけではなく更に多くのユースケースを対応できるよをアピールするように、いくつかのユースケースをまず提示しました。

### 1.3.1 リダイレクトベースのインタラクション（Redirect-based interaction）

![2.png](resources/BF8CFACE4FCB96A1156DD4FF858D7521.png =592x354)

リダイレクトベースのインタラクションはなんとなく一般的なOAuth2のユースケース（Auth Code Grant）に似ていて、リダイレクトベースです。具体的なリクエスト内容は後述になりますが、全体的な流れは以下のようになります。

* RO/RQは何かしらのサービスを利用し始め、RCにアクセスします。RCはRSにアクセスしたいので、認証認可を開始します（1）。
* RCはユーザを特定のURLに遷移させる能力を持っているため、その事実を認可リクエストとともにASに伝えます（2）。
* ASはインタラクションが必要と判断し、インタラクションに必要なURLとCallback時検証に必要な情報をレスポンスでRCに返却します（3）。
* RCはこういった情報を保管し、ユーザをインタラクション先のURLに遷移させます（4）。
* ユーザはその遷移先でASとやり取りして、認証（5）と認可（6）を行います。
* ASとのやり取りが完了後、ユーザはRCに戻されます（7）。
* RCは権限取得に必要な継続の作業を行い、ASと通信します（8）。その際、（3）で受信した「Callback時検証に必要な情報」を利用します。ASはこういった情報を検証し、リクエスト元は（2）の認可要求RCとは同一人物であることを確認します。
* ASがRCに対して権限を付与します（9）

### 1.3.2 ユーザコードインタラクション（User-code interaction）

![3.png](resources/5A5C82C8A7C25D22993CF2F16A2B1097.png =586x402)

ユーザコードインタラクションは、表示装置があるけどユーザインタラクションし難い端末（IoTデバイスなど）をターゲットとするユースケースで、ユーザをRC上でリダイレクトさせる代わりに、
* ディスプレイ上に○桁のコードをユーザに提示します。
* ユーザが能動的にASの特定のエンドポイントにアクセスし、`認証→コード入力→認可`の順番で操作を行い、RCに権限を付与します。

これはなんとなくOAuth2のDevice Flowに対応している形だと考えています。インタラクションモードは、RCが（1）のリクエストでASに提示します。

### 1.3.3 非同期認可（Asynchronous Authorization）

![6.png](resources/6A52EA55E83F4F28FA9B3CF7F3B80DBA.png =581x276)

非同期認可のユースケースでは、RQとROが異なる主体になります。ROは直接にRCとやり取りしません。Draftには言及しませんでしたが、RCとやり取りするのはRQだと理解すればわかりやすいでしょう。
RCが権限を要求するとき（1）にインタラクションモードを指定しないことで、`RCとRO`はやり取りできないことをASが判断します。そうするとASがGrant済んだかどうかのみをRCに返却し（2）、RCは継続的にGrant情報を問い合せます（Polling）。
その間にASは何かしらの方法でROと連絡を取り（プッシュ通知などと想像する）、ROがASと直接にやり取りし、認証認可を行います（3）（4）（5）。
そうするとRCからのPollingが特定のタイミングで成功になり、アクセス権限をRCが取得できるようになります（9）。

これは[Sign in with Google prompt](https://support.google.com/accounts/answer/7026266?co=GENIE.Platform%3DiOS&hl=en)のようなdecoupled authorizationの仕組みだと理解しています。

### 1.3.4 ソフトウェアオンリー認可（Software-only Authorization）

![4.png](resources/9B7E41EBA51C42A0FFFDDB23B0E69400.png =444x103)

ソフトウェアオンリー認可は、ROが参加しない認可で、ASがRCに対して権限付与するシンプルなユースケースです。
RO参加していないので、発行しているトークンもユーザコンテキストなしのもので、OAuth2でいうClient Credentials Grantに相当するだと理解しています。

### 1.3.5 期限切れのアクセストークンの更新

![5.png](resources/9EF67B053CF44A17BE96F96123AA2F41.png =505x238)

これはOAuth2でもおなじみのgrant_type=refresh_tokenに相当するものだと考えていて、RCが有効期限が刷新されたアクセストークンを再取得するユースケースです。
ただ違うのは、(2)でASからRCにトークンを返すと同時に、`token management URI`のメタ情報も一緒に返却しています。(5)のところでRCはこのURIにアクセスし、適切なキーやアクセストークンを提示することで、新しいトークンを取得します。

# 2. アクセス権のリクエスト

リクエストはRCからJSON形式で送られます。サンプルは以下になります。

```
{
    "resources": [
        {
            "type": "photo-api",
            "actions": [
                "read",
                "write",
                "dolphin"
            ],
            "locations": [
                "https://server.example.net/",
                "https://resource.local/other"
            ],
            "datatypes": [
                "metadata",
                "images"
            ]
        },
        "dolphin-metadata"
    ],
    "client": {
      "display": {
        "name": "My Client Display Name",
        "uri": "https://example.net/client"
      },
      "key": {
        "proof": "jwsd",
        "jwk": {
                    "kty": "RSA",
                    "e": "AQAB",
                    "kid": "xyz-1",
                    "alg": "RS256",
                    "n": "kOB5rR4Jv0GMeL...."
        }
      }
    },
    "interact": {
        "redirect": true,
        "callback": {
            "method": "redirect",
            "uri": "https://client.example.net/return/123455",
            "nonce": "LKLTI25DK82FX4T4QFZC"
        }
    },
    "capabilities": ["ext1", "ext2"],
    "subject": {
        "sub_ids": ["iss-sub", "email"],
        "assertions": ["id_token"]
    }
}
```

リクエスト内のfieldを一つずつ見て行きます。

## 2.1 リソースの要求

RCがAPIにアクセスするために一つか複数のアクセストークンを要求する場合、リクエストに `resources` フィールとを追加する必要があります。
`resources` は JSON array になっていて、各オブジェクトに以下の項目が含まれています。

|name|type|note|
| --- | --- | --- |
| type | string | 要求するリソースの種類<br/>ASはこの値が「何を指すか」を理解し、衝突なく定義します。<br/>一般的なAPIならURIを利用するのを推薦します。<br/>OAuth2 の [RAR](https://tools.ietf.org/html/draft-lodderstedt-oauth-rar-03) の仕様に合わせる予定があるが、どうやるかがまだ未定らしいです。 |
| actions | string array | 操作種類を示します。<br/>例として `read`, `write` などがあげられています。 |
| locations | string array | RS の所在を示す URI です。 |
| datatypes | string array | RS の API にある RC に提供できるデータの種類です。 |
| identifier | string | RS 上にある特定のリソースを示す識別子です。 |

リクエストの中に、全てのフィールドを含める必要がなく、ほしいリソースに合わせて設定します。リクエストが承認されると、アクセストークンには要求するリソースにアクセスできる権限が含まれます。

例

```
    "resources": [
        {
            "type": "photo-api",
            "actions": [
                "read",
                "write",
                "dolphin"
            ],
            "locations": [
                "https://server.example.net/",
                "https://resource.local/other"
            ],
            "datatypes": [
                "metadata",
                "images"
            ]
        },
        {
            "type": "financial-transaction",
            "actions": [
                "withdraw"
            ],
            "identifier": "account-14-32-32-3",
            "currency": "USD"
        }
    ]
```

また、AS / RS が理解できれば、JSON object ではなく、シンプルに文字列だけを送っても良いです。各文字列は具体的な意味を持つオブジェクトと紐付いています。これを`reference`と呼んでいます。

例
```
    "resources": [
        "read", "dolphin-metadata", "some other thing"
    ]
```

そして、reference と object が混在するリクエストもありえます。

```
    "resources": [
        {
            "type": "photo-api",
            "actions": [
                "read",
                "write",
                "dolphin"
            ],
            "locations": [
                "https://server.example.net/",
                "https://resource.local/other"
            ],
            "datatypes": [
                "metadata",
                "images"
            ]
        },
        "read",
        "dolphin-metadata",
        {
            "type": "financial-transaction",
            "actions": [
                "withdraw"
            ],
            "identifier": "account-14-32-32-3",
            "currency": "USD"
        },
        "some other thing"
    ]
```

一つのリクエストの中で、複数のアクセストークンを要求することもでき、それぞれ異なるリソースにアクセス権限を持つように表現できます。
その場合、resources は JSON array ではなく、JSON object になり、RCがトークン識別子`token1`, `token2`を勝手に指定することも可能になります。

```
    "resources": {
        "token1": [
          {
              "type": "photo-api",
              "actions": [
                  "read",
                  "write",
                  "dolphin"
              ],
              "locations": [
                  "https://server.example.net/",
                  "https://resource.local/other"
              ],
              "datatypes": [
                  "metadata",
                  "images"
              ]
          },
          "dolphin-metadata"
      ],
      "token2": [
            {
                "type": "walrus-access",
                "actions": [
                    "foo",
                    "bar"
                ],
                "locations": [
                    "https://resource.other/"
                ],
                "datatypes": [
                    "data",
                    "pictures",
                    "walrus whiskers"
                ]
            }
        ]
    }
```

以上定義されているパラメータ以外に、追加でいくつかのフラグをRCからASに送ることができます。

* `multi_token`: GNAPはトークンローテーション機能（後述）を定義しています。それを利用する際に、複数のトークンを同時に有効させる（古いトークンを即時無効させない）のは、このフラフです。
* `split_token`: RC が複数のトークンをハンドリングする能力を持っている、または指定しているトークン数と異なる数のトークンが返ってきたときに正しくハンドリングできることを AS に伝えるためのフラグです。そうすると AS は自分のポリシーでトークンを分割し、 RC に返すことができます。AS/RSのdeployment問題を解決するため云々と書いていて、RCの実装が複雑になるからどうかとコメントに書かれてますが、正直この機能が必要な理由がイマイチよくわからないです。
* `bind_token`: 発行された AS は、RC が利用している鍵に紐付いてほしいと AS に伝えるためのフラグです。Token Binding や Proof of Possession 的な仕組みかと理解しています。

この章は、resources について説明しました。resources は OAuth2 の `scope` の詳細版と理解すればよいかと思います。reference はわりと scope そのもののイメージがあります。
ここまで読んで、もうだいぶ疲れてました。

## 2.2 ユーザ情報の要求

RC が RO の情報を要求する場合、`subject` フィールドをリクエストに入れて AS に送信します。
現状、オプショナルで subject に以下のフィールドを追加することが可能となります。

| name | type | note |
| --- | --- | --- |
| sub_ids | string array | ユーザ識別子の種類。<br/>例えば`iss-sub`や`email`などが設定可能です。<br/>この識別子は、ROを識別するためだけのもので、例えばemailを利用するであれば、このemailに実際メールを送りユーザに届くという保証がありません（このような用途を想定していません）。 |
| assertions | string array | アサーションのフォーマットを指定します。<br/>OIDC流なら`id_token`で、SAMLなら`saml2`などを想定。 |

AS は いくつかの手段を通して、RO のアイデンティティ情報を提供するか否かを決定できます。

* RO とインタラクション
* AS 自体のポリシー
* RC からのアサーション情報

## 2.3 RC を識別する

以前のリクエストの継続でい限り、RC は必ず自分のアイデンティティを AS に提示する必要があり、それは `client` フィールド、かつリクエストに対する署名（後述）によって実現します。

以前のリクエストの続きのリクエストであれば、このフィールドを送信しないケースもあります。

`client` という JSON object は以下のフィールドを含めます。

| name | type | note |
| --- | --- | --- |
| key | object | リクエストに使われた公開鍵。 |
| class_id | string | RCのインスタンスの識別子。 |
| display | object | インタラクション時や認可時などに、AS が RO に提示するための追加情報。 |

例
```
"client": {
    "key": {
        "proof": "httpsig",
        "jwk": {
                    "kty": "RSA",
                    "e": "AQAB",
                    "kid": "xyz-1",
                    "alg": "RS256",
                    "n": "kOB5rR4Jv0GMeLaY6_It_r3ORwdf8ci_JtffXyaSx8xY..."
        },
        "cert": "MIIEHDCCAwSgAwIBAgIBATANBgkqhkiG9w0BAQsFA..."
    },
    "class_id": "web-server-1234",
    "display": {
        "name": "My Client Display Name",
        "uri": "https://example.net/client"
    }
}
```

RCが提示している公開鍵を所持していることを必ず証明しなければなりません。その仕組は後述になります。

RCの公開鍵が事前にASに登録していれば、RCからのリクエストをASが検証します。そして同じ公開鍵によって署名されたリクエストは同じRCインスタンスから送られたとみなしてよいです。
逆にRCの公開鍵がASに登録していなければ、ASは自分のポリシーやほかの仕組みによって、リクエストを受け入れるか拒否するかを決められます。

クライアントのアテステイション手法について、今後さらなる仕様に追加される可能性があります。

### 2.3.1 RCインスタンスの識別

もしRCインスタンスの鍵情報を何かしらの識別子を通してASが取得できるであれば、client フィールドにそのインスタンス識別子を送信すればよいです。インスタンス識別子は`instance_id`フィールドに入れます。

例
```
"client": {
    "instance_id": "client-541-ab"
}
```
あるいは
```
"client": "client-541-ab"

```

### 2.3.2 RC鍵の識別

`key`フィールドに公開鍵が含まれており、リクエストを署名するには必ずその鍵を利用する必要があります。
`key`フィールドに以下のサブフィールドが含まれます。
連続関係のあるリクエストは必ず同じ鍵（あるいは最新のローテートされた鍵）を利用する必要があります。

| name | type | note |
| --- | --- | --- |
| proof | string | RCがASに鍵を提示する方式を示すパラメータ。必須。後述。 |
| jwk | object | 公開鍵本体。[JSON Web Key](https://openid-foundation-japan.github.io/rfc7517.ja.html)形式になります。 |
| cert | string | リクエストを署名時に使った証明書。PEM形式。 |
| cert#256 | string | [RFC8705](https://www.rfc-editor.org/rfc/rfc8705.html)で定義されているX.509 Certificate SHA-256 Thumbprint。 |

例
```
    "key": {
        "proof": "jwsd",
        "jwk": {
                    "kty": "RSA",
                    "e": "AQAB",
                    "kid": "xyz-1",
                    "alg": "RS256",
                    "n": "kOB5rR4Jv0GMeLaY6_It_r3ORwdf8ci_JtffXyaSx8xY..."
        },
        "cert": "MIIEHDCCAwSgAwIBAgIBATANBgkqhkiG9w0BAQsFA..."
    }
```

### 2.3.3 RCディスプレイ情報の提供

RCがROに追加で提示したい情報があれば、`display`フィールドに該当情報を入れてASに送ります。

| name | type | note |
| --- | --- | --- |
| name | string | RCの表示名 |
| uri | string | ユーザに表示するRCのURI |
| logo_uri | string | RCを表す画像のURI |

AS はこれらの情報を RO に提示します。これらの情報は"表示"のためのもので、RC のアイデンティティを証明するための材料には利用できません。RC によって AS がこれらの情報を表示しなくてもよいです。

### 2.3.4 RC認証

RCを認証するには、公開鍵暗号方式を利用します。RCがASに提示する鍵は、ASが識別可能で、特定インスタンスのRCと紐付いていれば、その鍵を利用して十分にRCを認証できます。

この仕組は一連のリクエストがすべて同じRCからのものであることをASが検証できるようになります。

ASが事前に鍵を知らずにRCからのリクエストを許可するパターンもありえます。例えばRCはSPAあるいはモバイルアプリの場合、RCがたくさん独立のインスタンスを持ち、それぞれ自分の鍵ペアを生成します。その場合一連のリクエストが同じRCからのものであることを検証できるものの、その鍵はASが把握していないため、該当リクエストに対してある程度制限をかけることもありえます。例えば「以前登録したことのあるRCだけがこのリソースにアクセスする権限を要求できる」のようなポリシーを策定してもよいです。

## 2.4 ユーザを識別する

RC上にすでにユーザのアイデンティティを識別するための識別子やアサーションがあるなら、それらの情報を`user`フィールドに入れ、ASに送りつけることが可能になります。

| name| type | note |
| --- | --- | --- |
| sub_ids | object array | RQ subject識別子。 |
| assertions | object | アサーション。現状「ユーザ情報要求時」と同様に、`id_token`や`saml2`をサポート。 |

例
```
"user": {
   "sub_ids": [ {
     "subject_type": "email",
     "email": "user@example.com"
   } ],
   "assertions": {
     "id_token": "eyj..."
   }
}
```

この情報は、ASに対する「どのROの権限を許可するか」のヒント(※1)で、ASはこれを「このROがいまRQとしてRCを操作している」と理解しそのまま受け入れるべきではないです。またASはアサーションを検証すべきです。


また、インタラクションが発生時に、RQとROは異なる人物であれば、ASはこのリクエストを拒否すべきです(※2)。もし同じ人物であれば、ASは自分のポリシーによってRQとのインタラクションをスキップしても良いです。

※1 OIDCのid_token_hintのような用途で、silent authorizationみたいなROとのインタラクションを抑えるためのものだと理解すればよい、か？
※2 ようはインタラクションを抑えるのは、自分のリソースを操作するときのみになります。CIBAのようなユースケースはまぁ確かにインタラクションをスキップしたら成り立たないかもしれません。

また、参照によってユーザを識別する情報をASに提示することも可能になります。ASが識別できれば良いです。

```
"user": "XUT2MFM1XBIKJKSDU8QM"
```

## 2.5 ユーザとインタラクティブする

ROの権限をRCに移譲するため、多くの場合ASはROとインタラクションを行います。
多くの場合、ROはRQとは同じ人物で、同じデバイス上のリダイレクトや、アプリを起動するなどの方式で、そのインタラクションを開始できます。
ROはRQと同じ人物でない場合（ROとRQが使っているデバイスは異なるなど）、RCはASに情報を提供し、ROの端末にインタラクションを開始させたり、RCがROの（非同期な）同意を待つのも考えられます。あるいはRCがASからの「ROが同意済み」の通知を受け取るのも考えられます。

これらのインタラクションモードの中からRCは「自分ができるのはなにか」を`interact`フィールドに設定し、ASに送信します。複数設定も可能です。
ASは自分のポリシーによって、任意（any, all or none）のインタラクションモードに答えて良いです。
この仕様では、以下の5つのインタラクションモードを定義しています。

1. `redirect`: RCがRQを任意のURLにリダイレクトできることを示します。
1. `app`: RCはRQのデバイス上のアプリケーションを起動できることを示します。
1. `callback`: ROがASとのやり取りが終わった後に、RCがASからのコールバックを受け取られることを示します。
1. `user_code`: RCがRQに短いコードを提示できることを示し、そのコードをAS側の特定のURL上に利用できます。
1. `ui_locales`: インタラクションのプロセスの中、RQの好きな言語をASに提示します。

例1. リダイレクトOK、コールバック受け付けられる
```
    "interact": {
        "redirect": true,
        "callback": {
            "method": "redirect",
            "uri": "https://client.example.net/return/123455",
            "nonce": "LKLTI25DK82FX4T4QFZC"
        }
    }
```

例2. 255文字までのURLへのリダイレクトOK、ユーザコード表示OK、コールバック受け付けられない
```
    "interact": {
        "redirect": 255,
        "user_code": true
    }
```

RCが複数のインタラクションモードをユーザに提供したいケースも考えられるので、これらのオブジェクトを組み合わせることも可能になっています。

例3.
```
"interact": [
  {
    "redirect": true,
    "callback": {
       "method": "redirect",
       "uri": "https://client.example.net/return/123455",
       "nonce": "LKLTI25DK82FX4T4QFZC"
    }
  },
  {
    "redirect": 255,
    "user_code": true
  }
]
```

### 2.5.1 リダイレクトする

RCがRQを任意のURLへのリダイレクトが可能であれば、`redirect`フィールドに`true`を設定します。

```
"interact": {
   "redirect": true
}
```

RCが何を使ってそのURLを開くのはスコープ外ですが、だいたいの場合はHTTPリダイレクトになり、RCがWebブラウザを利用します。
RCが短縮されたURLが好む場合、URLの文字数制限を入れてASに送信します。ASはURLを短縮するすべがなければエラーを返さなければなりません。
（ASの実装が大変やな...）

```
"interact": {
   "redirect": 255
}
```

### 2.5.2 アプリケーションを起動する

RCがURLを通してアプリを起動できるであれば、`app`フィールドに`true`を入れます。

```
"interact": {
   "app": true
}
```

### 2.5.3 インタラクション後にコールバックを受け取る

ROインタラクション完了後の情報をASからのコールバックで受け取る能力がRCにある場合、RCが`callback`フィールドを送ります。その中に以下のパラメータが含まれます。

| name | type | note |
| --- | --- | --- |
| uri | string / REQUIRED | callback 先を示す URI。 |
| nonce | string / REQUIRED | hash計測用のユニーク値。callback URLに含まれ、callbackの有効性をRCが検証時に利用します。 |
| metehod | string / REQUIRED | ASからRCへcallbackする方式。`redirect`か`push`が選べる。 |
| hash_method | string / OPTIONAL | hash計算アルゴリズム。`sha3`か`sha2`が選べます。デフォはsha3。 |

OAuth2/OIDCになんとなくわかる方はご存知だと思いますが、このコールバックは主な攻撃ポイントの一つだと考えられます。そのため、OAuth2に似たように、この仕様にもそこそこ厳しい制限が定められています。

uri
* RC がホストする、またはアクセスできる場所
* fragment 含めてはいけない
* HTTPS か、RCのlocalhost上か、appのURI scheme になる必要がある
* front channel 通信（通常はリダイレクト方式）の場合、毎回必ずユニークなURIを利用しなければならない
* リダイレクト前に、リダイレクト先のURIをROに明示的に提示すべき

nonce
* 推測不可能、暗号的セキュア
* RC生成で、リクエストごとにユニーク

例
```
"interact": {
    "callback": {
       "method": "redirect",
       "uri": "https://client.example.net/return/123455",
       "nonce": "LKLTI25DK82FX4T4QFZC"
    }
}
```

#### redirect 方式

method フィールドが `redirect` になっている場合、RC が RO のブラウザ上で HTTP GET のリクエストを期待します。

この method は通常、RO と RQ が同一人物である場合に利用されます（同意する人ROと要求する人RQが同一端末利用するため）。そのため、substitution攻撃（おそらくROの代わりに攻撃者が不正に同意を行うような攻撃モデル？）を防ぐために、RC は RQ の有効なコンテキスト（おそらくログインセッションなどの形になるかと）がリクエストにあることを期待します。

#### push 方式

method フィールドが `push` になっている場合、RC は AS からの HTTP POST のリクエストを期待します。

AS から直接にリクエストが届くため、RC は RO のコンテキストがリクエスト上にいないことを期待します。

## 2.5.4 短いユーザコードを表示する

RC が短い、人間入力可能なコードを表示する能力があれば、`user_code`フィールドに true を設定し送信します。このコードは、ASが持つ固定のURLに入力されます。

```
"interact": {
    "user_code": true
}
```

### 2.5.5 期待ロケールを提示する

RC が RQ のロケールや言語好みを知っていれば、AS に `ui_locales` フィールド(string array)を提示することで、インタラクティブ時に利用する言語を要求できます。AS がどれもサポートしなかったらデフォルトの言語を利用すると良いです。

```
"interact": {
    "ui_locales": ["en-US", "fr-CA"]
}
```

### 2.5.6 インタラクションモードの拡張

乞うご期待。

## 2.6 RC の能力を申告する

RC がほかの能力を持っていれば、`capabilities` フィールド（string array）を通して申告可能です。
選択できる extension は今後レジストリに定義されますが、まだできてません。

```
"capabilities": ["ext1", "ext2"]
```

## 2.7 既存の Grant リクエストを参照する

以前許可されたリクエストがあれば、RC がそのリクエストの識別子を`existing_grant`フィールド（string）に入れて AS に送信できます。

```
"existing_grant": "80UPRY5NM33OMUKMKSKU"
```

この仕組みを提案された理由は、step-up authorization や downscoped authorization をサポートしたいためです。
※ 新しい完全なるリクエストではなく、既存のリクエストを参照しつつ、差分部のみリクエストすると理解しています。

## 2.8 OpenID Connect のクレームを要求する

RC と AS どっちも[OIDC のクレームクエリ言語](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#ClaimsParameter)をサポートしていれば、RC がそれを利用し `claims`フィールド（object）でASに送信できます。

```
        "claims": {
                "id_token" : {
                    "email"          : { "essential" : true },
                    "email_verified" : { "essential" : true }
                },
                "userinfo" : {
                    "name"           : { "essential" : true },
                    "picture"        : null
                }
        }
```

AS は OIDC と同樣にこのクエリを処理します。
`id_token` は要求しているクレームが ID Token の形式で RC に返されるを意味し、`userinfo` はアクセストークンに UserInfo エンドポイントにアクセスする権限が含まれることを意味します。

ちなみに

* ここらへんの仕様がまだ迷いが多いらしく、claims フィールドを設けるではなく、例えば `id_token` を `subject` に、`userinfo` を `resources` に入れてもよいでは？とも考えられます。
* 作者は今後OIDFがここらへんの仕様を策定し、GNAPの小さいアイデンティティレイヤーとして定義することを期待しているようです。

## 2.9 Grant リクエストを拡張する

2.6 でも言及したように、リクエストオブジェクトが今後拡張可能になり、（策定中の）レジストリに登録します。
ここらへんはまだ何もない状態です。

# 3. Grant レスポンス

RC のリクエストに対して、AS は JSON Object をレスポンスとして返却します。レスポンスに含まれる可能性あるフィールドは以下です。


| name | note |
| ---- | ---- |
| continue | これらのパラメータを使って、RC が継続的なリクエストを送ることが可能になります。 |
| access_token | RO の代わりになって RS にアクセスするための（一件のみの）アクセストークン。 |
| multiple_access_token | 複数件名前がついているアクセストークン。 |
| interact | 定義されているインタラクションを行う必要であることを示す。 |
| subject | ASが知っているROのクレーム情報。 |
| instance_id | RCインスタンスの識別子。今後さらなるリクエストを送るときにRC自分自身を識別するために利用可能。 |
| user_handle | 同樣に、RQの識別子で、今後リクエスト送るときにRCがRQを特定するために利用可能。 |
| error | エラーコード。 |

例1.
```
{
    "interact": {
        "redirect": "https://server.example.com/interact/4CF492MLVMSW9MKMXKHQ",
        "callback": "MBDOFXG4Y5CVJCX821LH"
    },
    "continue": {
        "access_token": {
            "value": "80UPRY5NM33OMUKMKSKU",
            "key": true
        },
        "uri": "https://server.example.com/tx"
    }
}
```

例2. 
```
{
    "access_token": {
        "value": "OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0",
        "key": false,
        "manage": "https://server.example.com/token/PRY5NM33OM4TB8N6BW7OZB8CDFONP219RP1L"
    },
    "subject": {
        "sub_ids": [ {
           "subject_type": "email",
           "email": "user@example.com",
        } ]
    }
}
```

## 3.1 リクエストの継続

AS が RC から追加のリクエストを受け付ける場合、`continue`フィールドを返します。そのフィールドに以下のパラメータが含まれます。

* `uri` (REQUIRED): 後継リクエストの送りつけ先。リクエストごとで変えてもいいし、AS が access_token を返し後継リクエストに access_token を含める場合、この uri が固定でも良い。
* `wait` (RECOMMENDED): RC がこのレスポンスをもらってから次の後継リクエストを投げるまで待つべき時間。
* `access_token` (RECOMMENDED): 後継リクエストに使われるアクセストークン。このトークンは bearer であってはならなず、RC の鍵に紐づく必要がある。このアクセストークンは AS 外の RS に利用してはならない。

例
```
{
    "continue": {
        "access_token": {
            "value": "80UPRY5NM33OMUKMKSKU",
            "key": true
        },
        "uri": "https://server.example.com/continue",
        "wait": 60
    }
}
```

RC は後継リクエストを署名する必要がありますし、access_token がレスポンスに付与された場合、この access_token を後継リクエストに含める必要があります。

## 3.2 アクセストークン

AS が `access_token` か `multiple_access_token` フィールド経由で、一件を複数件のアクセストークンを RC に発行します。
両フィールドどっちも RC に返してはなりません。

※ ちなみに作者はこういう形でフィールドを２つに分けるやり方が好きじゃないので、もうちょっといい方法ないかを考えているそうです。単純に string array で １件でも複数件でも返せるようにしちゃダメなのかな？

### 3.2.1 シングルアクセストークン

`access_token` フィールドに以下のパラメータが含まれます。

| name | type | note |
| ---- | ---- | ---- |
| value | string / REQUIRED | アクセストークンの値。<br/>HTTPヘッダにて追加エンコーディングなしで転送可能で、ASCIII文字列限定にすべき（SHOULD）。|
| manage | string / OPTIONAL | management URI。これによってRCがアクセストークンを管理可能になる（後述）。|
| resources | object array / RECOMMENDED | アクセストークンの権を示す。|
| expires_in | int / OPTIONAL | あと何秒でトークンが失効するか。 |
| key | boolean or object or string / REQUIRED | トークンに紐づく鍵。<br/>値が`true`になっているなら、その鍵はRCが提示したものになる。<br/>`false`になっているなら、このトークンはbearerトークンになる。<br/>それ以外のケースでは、この値はobjectかstringになり、RCの秘密鍵に紐づく公開鍵になる。 |

例1. management URI が含まれた access_token レスポンス
```
    "access_token": {
        "value": "OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0",
        "key": false,
        "manage": "https://server.example.com/token/PRY5NM33OM4TB8N6BW7OZB8CDFONP219RP1L",
        "resources": [
            {
                "type": "photo-api",
                "actions": [
                    "read",
                    "write",
                    "dolphin"
                ],
                "locations": [
                    "https://server.example.net/",
                    "https://resource.local/other"
                ],
                "datatypes": [
                    "metadata",
                    "images"
                ]
            },
            "read", "dolphin-metadata"
        ]
    }
```

例2. detached JWS binding（後述）方式鍵を利用するケース
```
    "access_token": {
        "value": "OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0",
        "key": true,
        "resources": [
            "finance", "medical"
        ]
    }
```

### 3.2.2 複数アクセストークン

RC が複数のアクセストークンを一つのリクエストで要求できるので（リクエスト内`multi_token`を指定）、正しく一件以上のトークン発行できた場合、AS が `multiple_access_tokens` フィールドでそれらのトークンを返します。
RC がトークン名を指定できるので、AS がトークンをそれに合わせてレスポンスを返します。

例
```
    "multiple_access_tokens": {
        "token1": {
            "value": "OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0",
            "key": false,
            "manage": "https://server.example.com/token/PRY5NM33OM4TB8N6BW7OZB8CDFONP219RP1L"
        },
        "token2": {
            "value": "UFGLO2FDAFG7VGZZPJ3IZEMN21EVU71FHCARP4J1",
            "key": false
        }
    }
```

RC が multi_token を要求していれば、AS は必ず multiple_access_token でトークンを返します。実際1件しか発行しなくても。同様に RC が single トークン（split_token ついてない限り）を要求する場合、AS は必ず access_token でトークンを返します。

## 3.3 インタラクションモード

RC が RO とインタラクションを行う能力を持っていて、AS もインタラクションを行う必要があると判断すれば、レスポンスに `interact` フィールドを入れて RC に返します。
レスポンスに以下のパラメータ（インタラクションモード）が含まれ、どれを利用するかは RC 次第です。順番などもありません。

リクエストに似たように、以下の値が定義されています。

* `redirect`
* `app`
* `callback`
* `user_code`

AS は RC のリクエストに指定していないモードを返してはいけません。そして、ASが一つのレスポンスの中に、一つの interact 要求に対して、一回しか応答しないようにすべきです（シークレットやユニークな情報がレスポンスに含まれるため。複数のレスポンスを入れるとRCがどう反応すべきかがわからなくなるからかと）。

### 3.3.1 任意URLへリダイレクトする

RCがリダイレクト可能 & ASがこのモードをサポートするであれば、ASが`redirect`フィールドを返します。
その中に、RQのリダイレクト先のURLが含まれていて、このURLはリクエストごとユニークになります。

```
    "interact": {
        "redirect": "https://interact.example.com/4CF492MLVMSW9MKMXKHQ"
    }
```

このURLは、アクセス権をリクエストするためのURLとは全く異なるものにしても良いです。そうすることでASにとってユーザインタラクション機能をバックエンドセキュリティ機能を分離でき、都合がいいだろうと考えられています。
ASは「権限移譲サーバ（backend）」と「ユーザインタラクションサーバ（user-facing）」の２つのロールに分かれられます。

受信後、RC該当URLを変更せずにRQにそこに遷移させ、ASとやり取りさせます。遷移の形は定義しませんが、通常は以下の形になります。

* ブラウザ上のHTTPリダイレクトを行う
* スキャン可能なコードを表示する
* インタラクティブ可能な機器上に該当URLを表示する

### 3.3.2 アプリケーションを起動する

RC はアプリURLを指定する & ASがこのモードをサポートする場合、RQを遷移するURLが含まれる`app`フィールドがついている、以下のようなレスポンスを返します。

```
    "interact": {
        "app": "https://app.example.com/launch?tx=4CF492MLV"
    }
```

RCはこのURLを利用してアプリケーションを起動します。

### 3.3.3 インタラクション後にRC URLへコールバックを送信する

RC はインタラクション後のコールバックを受信可能と示し、AS がサポートしていれば、`nonce` を含む `callback` フィールドをレスポンスで返します。このnonceはRCがコールバックを検証するために利用されます。詳しくは4.4.1で説明します。

```
    "interact": {
        "callback": "MBDOFXG4Y5CVJCX821LH"
    }
```

RO は AS 上でのインタラクションを完了したあとに、AS が RC のコールバックURLに指定されたメソッドで呼び出します。

コールバックを受信するまで、RCが後継リクエストを送信してはなりません。

### 3.3.4 ユーザコードを表示する

RC はユーザコードを表示できるを示し、AS がサポートしていれば、AS が`user_code`フィールドをレスポンスに含めます。

```
    "interact": {
        "user_code": {
            "code": "A1BC-3DFF",
            "url": "https://srv.ex/device"
        }
    }
```

* `code` (REQUIRED): ユニークなショートコード。ユーザがそれをASに入力できる。CASE INSENSITIVE。入力しやすい形でなければならない（数字やアルファベットなど）。有効期限が短い（数分程度）。長さ8文字以下を推奨する。
* `url` (RECOMMENDED): RC から RO をリダイレクトする AS の URL。コード入力用。URLが変わってはいけない。

RC はこのコードを画面に表示するなり、読み上げたりして、なにかしらの方法でユーザに提示しなければなりません。このコードは使い切りのクレデンシャル扱いで、AS が RC からのリクエストを特定するために利用されます。RO がこのコードを入力したあとに、AS が該当するリクエストを特定できなかったら、ユーザにエラーを表示します。複数回検証失敗した（間違ったコードが入力された）場合、AS が何かしらのエラーをユーザに表示すべきです。

このモードは、RCがRQを該当URLにリダイレクトすることを期待していません、代わりに、別の端末上でRQがASにアクセスし、コードを入力することを期待しています。
もしRCがRQに対して、任意のURLを提示可能（例えば二次元バーコードをスキャンさせるなど）であれば、`user_code`モードではなく、`redirect`モードを利用すべきです。

### 3.3.5 インタラクションモードレスポンスの拡張

新しいインタラクションモードレスポンスをレジストリを通して拡張可能です。
現状はまだ何もないようです。

## 3.4 ユーザ情報の返却

RC が RO に関する情報を要求した & AS が認可した場合、`subject`フィールドをレスポンスに含めます。

```
"subject": {
   "sub_ids": [ {
     "subject_type": "email",
     "email": "user@example.com",
   } ],
   "assertions": {
     "id_token": "eyj..."
   }
}
```

* `sub_ids`: RO の識別子。
* `assertions`: アサーション情報を含めたオブジェクト。
* `updated_at`: アカウント情報が最後に更新された時刻。この時刻を見て、RCが更新されたプロフィールを取得するかどうかを判断してもよい。

AS は RO と RQ が同じ人物であることを確認してから`subject` を返さなければならない。確認の方法は 4章 にて説明します。

返却された識別子は AS 上で RO を識別するためにユニークである必要がある。この識別子は issuer と subject の組み合わせで、opaque な識別子でもよいです（グローバルでユニークな識別子も設けるかを作者が考えているらしい）。

前も言ったように、この識別子は仮にemailや電話番号を利用するとしても、これは正しいユーザ情報であることを示すわけではなく、あくまでもユーザを識別するためのものです。
正しくユーザ情報を取得するには、この識別子を利用してASが提供するIdentify APIのようなAPIにアクセスするなども考えられますが、GNAPの仕様外です。
また、プライバシーに関する配慮も必要です。

## 3.5 Dynamically-bound Reference Handles の返却

RC はリクエストを送るときに、いつくかの部分では値かリファランスのどっちかで送れ、値ではなくリファランスを送ることでリクエストを最適化できます。
クライアントインスタンスのアイデンティティや、要求するリソースなどは、リファランスで要求することによって、リクエスト内容が静的な値になり、RC 開発者はこの値をもっとコンパクト、もっと効率的にコードに記載できます。

必要であれば、同じソフトウェアと複数のインタラクションを行うときに、AS はリファランス値を動的に生成しRCに返します。このリファランス（ここはhandleと呼ぶ）は将来のリクエストに利用できます。
こういったリファランス値は文字列で、秘匿性があります。（推測不可能、センシティブ情報含まれない、RCにとってはopaqueである）

* `instance_id`: クライアントを示す文字列。RCインスタンスのリファランスとして利用可能（2.3.1）。
* `user_handle`: 現ユーザを示す文字列。ユーザのリファランスとして利用可能（2.4.1）。

例: user, RC instance の handle x 2 とアクセストークンを同時に発行する
```
{
    "user_handle": "XUT2MFM1XBIKJKSDU8QM",
    "instance_id": "7C7C4AZ9KHRS6X63AJAO",
    "access_token": {
        "value": "OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0",
        "key": false
    }
}
```

※ この仕組みは、dynamic registration のインラインバージョンとして利用可能で、リクエストに先立ちclient registrationを行う必要すらないです。instance_id も [RFC 7592](https://tools.ietf.org/html/rfc7592#page-5) の dynamic client management のように利用できそうです。

## 3.6 エラーレスポンス

AS がいかなる理由でレスポンスを返せないと決めていれば、エラーレスポンスを返します。

```
{

  "error": "user_denied"

}
```

以下のエラーコードをいま定義しています。またその値も拡張可能です。

* `user_denied`: RO がリクエストを拒否した
* `too_fast`: RC が wait で定められた待ち時間を守らなかった
* `unknown_request`: 不明な進行中のアクセスリクエスト

## 3.7 レスポンスの拡張

Grant レスポンスのフィールドを追加することが可能です。

# 4. AS 上でのインタラクション

RO が AS とインタラクションを行うとき、AS は適切なアクションを取ると良いです。
例えば
* ユーザを RO として認証する（この人はRQである可能性もある）
* ROから同意情報を収集する
* リクエストのパラメータをROに修正させる（例えば、同意したくないリソースを外す、アカウントを指定してもらう、など）
* ROに対して、認可に伴う可能なリスクを警告する

## 4.1 リダイレクト URI ベースでインタラクションを行う

RO が `redirect` モードで AS にリダイレクトされた場合、AS は RO とブラウザ上でやり取りして、同意を収集できます。
RC が interaction_url にパラメータを追加することないので、AS はこのRCに対して何をgrantするかの情報を保持する必要があります。
grant 情報がなければ、RCにリダイレクトバックするではなく（MUST NOT）、エラーを表示します。

intraction_url は HTTP GET でブラウザでアクセスでき、HTTPS である必要があります。ユーザが別のブラウザで開いてもよいです。

この方式は基本的に RO と RQ が同じ party であることを想定しています。

## 4.2 ユーザコードベースでインタラクションを行う

`user_code` モードでインタラクションを行う場合、AS は RO の web ブラウザを利用して、ユーザコードを回収し、ユーザを認証し、ユーザ同意を収集します。
`redirect` モード同様に、user_code の URL も静的なものなので、AS はその URL に紐付いている grant 情報を保持する必要があります。情報がない場合のルールも`redirect`モードと同じです。
また、user_code の URL も webブラウザから HTTP GET でアクセスでき、HTTPS によって保護される必要があります。

## 4.3 アプリケーションURIベースでインタラクションを行う

`app`モードによってアプリケーションを起動できた後に、AS はそのアプリケーションを通して RO とやり取りを行います。
同じように認証を行い、同意を収集します。
具体的な認証・同意収集の仕組みは本仕様の範囲外になります。

※ このアプリケーションはモバイル端末上のGoogleのSmart Lock、Auth0 の Guardian みたいなものを想定しているだろうと想像しています。

## 4.4 インタラクションの後始末

RO とのインタラクションが終わった後に、AS は `callback` の情報通りに、RC に戻ったりして RC に処理を継続させます。`callback` がなければ AS は RO に指示して RC に戻らせるべきです。
同意拒否など、エラー時にも同じです。

インタラクション完了時に、AS は現在のインタラクションと進行中のリクエストに紐づくリファランスを生成します。このリファランス値は推測不可能で、ワンタイム性を持ちます（MUST）。
また、RC と AS の nonce 値とリファランス値をもとに、Hash値を生成し、リファランス値と一緒に RC に返します。RC はこの Hash 値を利用して返却されたクエリパラメータの値を検証します。

リファランスとHashの返却方法は、インタラクションモードに依存します。

### 4.4.1 Callback URI にブラウザリダイレクト

インタラクションモードは `redirect` の場合、AS は RO をブラウザ上で RC の callback URL にリダイレクトバックすることで、RC に「インタラクションが完了した」ことを通知します。
Hash 値を計算し RC に返すことで、リクエストの改ざんを防げます。

* `hash` (REQUIRED): インタラクションHash値。詳細な計算方法は後述。
* `interact_ref` (REQUIRED): インタラクションリファランス値。

例
```
https://client.example.net/return/123455
  ?hash=p28jsq0Y2KK3WS__a42tavNC64ldGTBroywsWxT4md_jZQ1R2HZT8BOWYHcLmObM7XHPAdJzTZMtKBsaraJ64A
  &interact_ref=4IFWWIKYBC2PQ6U56NL1
```

RC は必ず Hash 値の有効性を検証します。検証成功後、RC は後継リクエストを AS に送信します（後述）。その際、interact_refを付け加えます。

### 4.4.2 直接な HTTP リクエストコールバック

`callback` インタラクションモードを指定する場合、AS は RC に HTTP POST によってリクエストを送信し、インタラクション完了したことを通知します。
メッセージボディは JSON 形式になります。中身は、`redirect`モードの場合とは基本的に一緒です。

例
```
POST /push/554321 HTTP/1.1
Host: client.example.net
Content-Type: application/json

{
  "hash": "p28jsq0Y2KK3WS__a42tavNC64ldGTBroywsWxT4md_jZQ1R2HZT8BOWYHcLmObM7XHPAdJzTZMtKBsaraJ64A",
  "interact_ref": "4IFWWIKYBC2PQ6U56NL1"
}
```

同様に、RC は Hash を検証し、成功すれば AS に後継リクエストを送信します。

### 4.4.3 インタラクションHash値の計算

RC へのコールバックリクエストに含まれる `hash` 値は、front channel レスポンスを進行中のリクエストにひも付きます。この仕組は、RC からセッション固定/インジェクションなどの攻撃から守ります。

Hash を計算するには、RC が AS に送る `interact` に含まれる `nonce` の値（2.5.3）と、AS が grant レスポンスに入れた nonce 値（`callback`フィールド）（3.3.3）と、`interact_ref` を利用します。
この値たちを改行でつなぎます。paddingやwhitespaceが含まれません。末尾にも改行を入れません。

例
```
VJLO6A4CAYLBXHTR0KRO
MBDOFXG4Y5CVJCX821LH
4IFWWIKYBC2PQ6U56NL1
```

そして、この値を RC が指定する `hash_method` （2.5.3、デフォはsha3）でハッシュ化します。
将来的にベストプラクティスをもとに、最も適切なアルゴリズム（plain hash, keyed MAC, HMAC, etc...）を提案しますが、現時点では早期での開発のためとりあえず SHA512 と SHA3 を定義します。

* `SHA3-512`: 入力を `512-bit SHA3 アルゴリズム` でハッシュ化し、URLセーフなBase64（paddingなし）でエンコーディングします。
* `SHA2-512`: 入力を `512-bit SHA2 アルゴリズム` でハッシュ化し、URLセーフなBase64（paddingなし）でエンコーディングします。

# 5. Grant リクエストの継続

AS と RC のやり取りは複数回に渡って行うのを想定しています。これインタラクションを円滑するためであり、AS - RC 間でパラメータについての交渉を継続的に行わせるためにも利用できます。
リクエストを継続させるために、AS はレスポンスに Access Token や後継リクエストの送信先など後継リクエストに必要な情報を含めた `continue` フィールドを返します。

RC が後継リクエストを送信時に、進行中のリクエストに紐づく鍵を利用して、リクエストを署名します。この鍵は最初のリクエストに含まれるか、ローテート後の最新のものになります。

例：Detached JWS によって署名された HTTP POST リクエスト
```
POST /continue/80UPRY5NM33OMUKMKSKU HTTP/1.1
Host: server.example.com
Detached-JWS: ejy0...
```

`continue`フィールドに`access_token`が含まれた場合、RC は必ずアクセストークンをリクエストに含めなければなりません。アクセストークンは RC の鍵にひも付きます。

例：Detached JWS 署名の上、アクセストークンを含めた HTTP POST リクエスト
```
POST /continue HTTP/1.1
Host: server.example.com
Content-type: application/json
Authorization: GNAP 80UPRY5NM33OMUKMKSKU
Detached-JWS: ejy0...

{
  "interact_ref": "4IFWWIKYBC2PQ6U56NL1"
}
```

AS はこのリクエストから、「これはどのリクエストの継続か」を識別しなければならないです。通常、２つのやり方が考えられます。
* 常に1 RCインスタンスに同時に一つのリクエストしか進行できないようにする
* ユニークな URL を発行し、RC にアクセスしてもらう

この継続の仕組みを利用して、RC は以下のようなことができるようになります
* インタラクションのために、追加情報を AS に提示する
* 最初のリクエストの情報を修正する
* リクエスト現在の状態を取得する

3.1 に記載されたように、継続レスポンスに`wait`フィールドが含まれた場合、RCはこの間隔をおいてから AS にアクセスしなければなりません。`wait` がなくても RC はレスポンスを受けてから `5秒` 以上を待ってから次のリクエストを送信すべきです。

後継リクエストを受信してから、更に後継リクエストを許容する場合、AS が新しい`continue`フィールドをレスポンスに追加します。以前の継続レスポンスに`access_token`を発行した場合、その次の継続レスポンスにも新しい`access_token`を発行しなければなりません。
`continue`フィールドがなければ、RC は後継リクエストを送ってはなりません。

## 5.1 インタラクションが完了後での継続リクエスト

前に説明したように、AS が RC の`callback`リクエストに対して返すレスポンスに、`interact_ref`パラメータを含めます。
継続URIにアクセスする際に、RC は受信した`interact_ref`パラメータを含めなければなりません。

```
POST /continue/80UPRY5NM33OMUKMKSKU HTTP/1.1
Host: server.example.com
Content-type: application/json
Detached-JWS: ejy0...

{
  "interact_ref": "4IFWWIKYBC2PQ6U56NL1"
}
```

interact_ref はワンタイム性があり、再利用してはなりません。再利用された場合、AS はエラーを返すべきだし、進行中のリクエストを破棄すべきです。

前に説明したように、このリクエストに対するレスポンスには、新たに発行されたAccess Tokenを含めたり、Subject Claim を含めたりすることが可能であり、さらに`continue`フィールドを含めることも考えられます。
以下の例では、新たにAccess TokenやSubject Claimが発行されましたが、continue がないので、継続的にリクエストを送信できません。

```
{
    "access_token": {
        "value": "OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0",
        "key": false,
        "manage": "https://server.example.com/token/PRY5NM33OM4TB8N6BW7OZB8CDFONP219RP1L"
    },
    "subject": {
        "sub_ids": [ {
           "subject_type": "email",
           "email": "user@example.com",
        } ]
    }
}
```

## 5.2 インタラクションが一時中断時での継続リクエスト

RC が `callback` フィールドを付け加えない場合、RC がポーリング方式で AS から RO の認可結果を取得しなければなりません。
その場合、RC は ボディのないリクエストを HTTP POST を継続URIに送信します。

```
POST /continue HTTP/1.1
Host: server.example.com
Content-type: application/json
Authorization: GNAP 80UPRY5NM33OMUKMKSKU
Detached-JWS: ejy0...
```

AS からのレスポンスは5.1と同様に、AccessToken、Subject Claim、continue が含まれる可能性があります。
例えば今度の例は、リクエストがまだ成功せず、継続リクエストを要求する場合、 continue フィールドを返すとします。前説明したように、ポーリングを優しくしてとRCに指示する意味で、待ち間隔の`wait`フィールドを含めるべきです。

```
{
    "continue": {
        "access_token": {
            "value": "33OMUKMKSKU80UPRY5NM",
            "key": true
        },
        "uri": "https://server.example.com/continue",
        "wait": 30
    }
}
```

リクエストが成功したら、5.1で示す例と似たようなレスポンスをASが返します。

## 5.3 既存のリクエストの変更

トークンやSubject Claimがすでに発行されたかどうかに関わらず、RC が進行中のリクエストを変更したくなる可能性があります。その場合、RC は HTTP PATCH リクエストを継続URIに変更したいフィールドを送信します。送信していないフィールドは変更されません。

`resources`や`subject`への変更も可能ですが、AS の要求やポリシーに引っかかる可能性があります。例えば追加の権限を要求する場合、AS が追加のインタラクションを RO に要求し、同意を収集する可能性があります。限定的な権限を要求する場合、AS の判断で権限を即時に RC に付与することもありえます。
※ アクセストークンをローテートするためにこの機能を使うべきじゃないので、既存のリクエストに一致するかどうかをチェックする？みたいなことも検討しているらしいです。

RC は`interact`フィールドをリクエストに追加し、変更を要求することが可能で、意味や利用方法は新しいリクエストを送るときと同じです。受信後にASは既存のinteract情報を上書きします。

同様に、RC は`user`フィールドを送信できます。新しいアサーションやRQの情報をASに伝えます。
※ ここで送信したユーザのアイデンティティはAS上で認可時のROと異なる場合、問題を引き起こす可能性がありそうだけど、複垢の場合にはなんとなく役に立ちそう。

ただし、このリクエストの中には`client`フィールドを追加してはなりません。
※ユーザが知らないときにRCを入れ替えるのはあれなんだからね。ただ鍵のローテーションにはこのフィールドを利用するのはただしそう。

リクエストの変更はすでに発行済みのアクセストークン（の権限など）を変更しなければなりません。ただし、継続的リクエストは常に新しいトークンを発行するため、既存のトークンを無効化にすることが可能です。

リクエストの変更がASに許可された場合、直ちにアクセストークンを発行したり、Subject Claimを発行したりすることができます。同様に、`continue`や`interact`フィールドを返すことも可能です。

#### 例

最初のリクエスト

```
POST /tx HTTP/1.1
Host: server.example.com
Content-type: application/json
Detached-JWS: ejy0...

{
    "resources": [
        "read", "write"
    ],
    "interact": {
        "redirect": true,
        "callback": {
            "method": "redirect",
            "uri": "https://client.example.net/return/123455",
            "nonce": "LKLTI25DK82FX4T4QFZC"
        }
    },
    "client": "987YHGRT56789IOLK"
}
```

許可済みでアクセストークンが発行されたレスポンス

```
{
    "continue": {
        "access_token": {
            "value": "80UPRY5NM33OMUKMKSKU",
            "key": true
        },
        "uri": "https://server.example.com/continue",
        "wait": 30
    },
    "access_token": ...
}
```

RCが`write`権限が不要になり、継続的なリクエストで、その権限を取り消します

```
PATCH /continue HTTP/1.1
Host: server.example.com
Content-type: application/json
Authorization: GNAP 80UPRY5NM33OMUKMKSKU
Detached-JWS: ejy0...

{
    "resources": [
        "read"
    ]
    ...
}
```

AS は既存の`resources`を上書きし、read権限のみついているアクセストークンを新たに発行します。古いwrite権限がついているトークンをRevokeしてもよいです。

```
{
    "continue": {
        "access_token": {
            "value": "M33OMUK80UPRY5NMKSKU",
            "key": true
        },
        "uri": "https://server.example.com/continue",
        "wait": 30
    },
    "access_token": ...
}
```

#### 例2

RC は最初に read only の権限を要求しました。

```
POST /tx HTTP/1.1
Host: server.example.com
Content-type: application/json
Detached-JWS: ejy0...

{
    "resources": [
        "read"
    ],
    "interact": {
        "redirect": true,
        "callback": {
            "method": "redirect",
            "uri": "https://client.example.net/return/123455",
            "nonce": "LKLTI25DK82FX4T4QFZC"
        }
    },
    "client": "987YHGRT56789IOLK"
}
```

```
{
    "continue": {
        "access_token": {
            "value": "80UPRY5NM33OMUKMKSKU",
            "key": true
        },
        "uri": "https://server.example.com/continue",
        "wait": 30
    },
    "access_token": ...
}
```

その後、追加の write 権限を要求します。強い権限を要求するので、追加のインタラクションを見越して、RC は `interact` フィールドも送信します。
この interact は最初に送信した interact の nonce & uri の値とは異なります。なぜならこれらの値はワンタイム性が要求されています。

```
PATCH /continue HTTP/1.1
Host: server.example.com
Content-type: application/json
Authorization: GNAP 80UPRY5NM33OMUKMKSKU
Detached-JWS: ejy0...

{
    "resources": [
        "read", "write"
    ],
    "interact": {
        "redirect": true,
        "callback": {
            "method": "redirect",
            "uri": "https://client.example.net/return/654321",
            "nonce": "K82FX4T4LKLTI25DQFZC"
        }
    }
}
```

このリクエストを受信した AS は最初のリクエストを受信したときと同じように RO から同意をもらったりしてもよいし、そのままトークンを発行しても良いです。また、元トークンの権限が弱いので、AS がそれを revoke しないと選択しても良いです（ただし write 権限を発行済のアクセストークンに追加付与することはできません）。

## 5.4 現在進行中の Grant リクエストの状態の取得

RC は現在進行中のリクエストの状態を知りたい場合、継続URIに HTTP GET にリクエストを送ります。
このリクエストはすでに送信された Grant リクエストのパラメータを変更できません。

AS はレスポンスの中に、`3. Grant レスポンス` に記載されているレスポンス項目をすべてこのリクエストのレスポンスに含めてよいです。
例えばすでに発行済のアクセストークンやSubject Claim、インタラクションモデルなど。ただし、新たにトークンを発行できません。
また、`continue`フィールドを返却しても良いです。

例
```
GET /continue HTTP/1.1
Host: server.example.com
Content-type: application/json
Authorization: GNAP 80UPRY5NM33OMUKMKSKU
Detached-JWS: ejy0...
```

※ トークンをそのまま返していいのか？を含めて、この仕様に対するたくさんの質問を答えなければなりませんが、一旦これで。
※ HTTP Method を揃える意味でもこの仕組みを提供したい、かも。

## 5.5 Grant リクエストの取消

もし RC が Grant リクエストを取消したいなら、HTTP DELETE リクエストを継続URIに送信します。

```
DELETE /continue HTTP/1.1
Host: server.example.com
Content-type: application/json
Authorization: GNAP 80UPRY5NM33OMUKMKSKU
Detached-JWS: ejy0...
```

成功の場合、AS は 202 で応答します。AS は取り消されたGrantリクエストに関連するすべてのアクセストークンをrevokeしなければなりません。

# 6. トークン管理

3.2.1 で記載されたように、トークンレスポンスに`manage`パラメータが含まれた場合、RC はこのパラメータに含まれているURLを通してアクセストークンを管理できます。

アクセストークンを提示時に、RCはトークンの所持証明を提示しなければなりません。
アクセストークンが sender-constrained の場合、リクエストは適切にトークンとバインドして送信しなければなりません（詳細は Section 7 にて）。
アクセストークンが bearer トークンの場合、最初のリクエストに提示した鍵の所持証明を提示しなければなりません（詳細は Section 8 にて）。
AS は必ず証明を検証し、トークンの提示方式次第でトークンは正当なASかRCに紐づくことを検証しなければなりません。

この仕様では、以下の2種類の管理内容を定義しています、

## 6.1 トークンのローテーション

RC は HTTP POST でトークン管理URIにアクセスします。トークンを適切なヘッダにセットし、適切な鍵でリクエストを署名します。

```
POST /token/PRY5NM33OM4TB8N6BW7OZB8CDFONP219RP1L HTTP/1.1
Host: server.example.com
Authorization: GNAP OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0
Detached-JWS: eyj0....
```

AS はトークンと署名を検証します。
トークンの有効期限が切れていても、AS はローテーションを続行すべきで、なぜならトークンをローテートする理由はトークンの有効期限が切れていてRCがそのトークンをリフレッシュしたい可能性が高いからです。ただし、トークンがrevokeされたときにはローテートを続行してはなりません。

提示されたアクセストークンと署名の有効性が検証されたあとに、AS は該当管理APIに紐づくアクセストークンを失効させるべきです。`multi_token`フラグが存在しなければ、新しいアクセストークンを含めたレスポンスを返します。

新たに発行したアクセストークンの値は、既存のアクセストークンの値と一致してはならないです。レスポンスに新しい管理APIを発行してもいいです。その際に、新しいアクセストークンはその新しい管理APIを利用します。

```
{
    "access_token": {
        "value": "FP6A8H6HY37MH13CK76LBZ6Y1UADG6VEUPEER5H2",
        "key": false,
        "manage": "https://server.example.com/token/PRY5NM33OM4TB8N6BW7OZB8CDFONP219RP1L",
        "resources": [
            {
                "type": "photo-api",
                "actions": [
                    "read",
                    "write",
                    "dolphin"
                ],
                "locations": [
                    "https://server.example.net/",
                    "https://resource.local/other"
                ],
                "datatypes": [
                    "metadata",
                    "images"
                ]
            },
            "read", "dolphin-metadata"
        ]
    }
}
```

## 6.2 トークンの取り消し（Revoke）

ユーザが RC に権限付与を差し止めたい、あるいは RC が自分がアンインストールされたと検出した、などのときに、RC が能動的にアクセストークンをrevokeしたい場合、RC は HTTP DELETE をトークン管理APIにリクエストを送信し、 AS に特定のトークンを失効させると指示できます。

リクエストにアクセストークンを含め、適切な鍵を利用してリクエストを署名する必要があります。

```
DELETE /token/PRY5NM33OM4TB8N6BW7OZB8CDFONP219RP1L HTTP/1.1
Host: server.example.com
Authorization: GNAP OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0
Detached-JWS: eyj0....
```

AS は指定されているアクセストークンを失効させ、HTTP 204 レスポンスを返します。

```
204 No Content
```

AS は任意のタイミングで任意の理由でアクセストークンを失効させられますが、トークン管理APIの機能は RC のために提供しています。
トークンがすでに有効期限切れや他のきっかけでrevokeされていても、AS はトークン取り消しのリクエストを有効なものとして取り扱わなければなりません（エラーを返さない意味かな？）

# 7. アクセストークンの利用

RC が RS にアクセストークントークンを送る方式は、アクセストークンレスポンス（3.2.1）にある `key` と `proof` の値に依存します。
`key` の値が `false` の場合、アクセストークンは `bearer` トークンになり、認可ヘッダにトークンを入れて送信します。

```
Authorization: Bearer OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0
```

[RFC6750](https://www.rfc-editor.org/rfc/rfc6750.txt)が定義しているもう２つの送信方法 `Form-Encoded Body Parameter` と `URI Query Parameter` は利用してはなりません。

もし `key` の値が `true` であれば、アクセストークンは RS に送信する際に、最初の Grant リクエストを送信する際に利用している同じ鍵と同じリクエスト署名方式を利用して、リクエストを署名します。

もし `key` の値はオブジェクトになるとしたら、`key` にある `proof` には、トークンの所持を証明する仕組みが定義されています。
アクセストークンは HTTP 認可スキーム `GNAP` を利用し、後述（Section 8）の鍵証明と一緒に送信されます。
例えば以下のリクエストは、`jwsd` 縛りなアクセストークンは以下の格好で RS に送信されます。

```
Authorization: GNAP OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0
Detached-JWS: eyj0....
```

# 8. キーバインディング

RC から AS や RS に提示した鍵は必ず提示時のリクエスト中で検証されなければなりません。バインディングの方式は`key.proof`パラメータ（Section 2.3.2）で示されます。そのパラメータの値は以下の6種類です。

* `jwsd`: Detached JWS signature ヘッダ
* `jws`: Attached JWS ペイロード
* `mtls`: Mutual TLS 証明書
* `dpop`: OAuth Demonstration of Proof-of-Possession ヘッダ
* `httpsig`: HTTP Signing 署名ヘッダ
* `oauthpop`: OAuth PoP 認証ヘッダ

この仕様で使われたキーバインディングのメソッドは、AS がセキュアにリクエストを検証するために、必ずリクエストのすべてのパートをカバーしなければなりません。
* 呼び出されたURI
* HTTPメソッド
* HTTPヘッダとその値
* HTTPメッセージボディ

リクエスト受信者は署名済みのメッセージの全パートを検証し、リクエストが改ざんされていないことを確認しなければなりません。

GNAP を利用する際、これらのキーバインディングの仕組みは、RC が最初のリクエストで提示した鍵が後継リクエスト中にでも送信者が所有することを検証する能力を AS に付与します。
この要件を円滑にするため、最初のリクエスト（Section 2.3.2）に含まれたすべての鍵が、後継リクエスト（Section 5）やトークンマネジメントリクエスト（Section 6）の中で検証される必要があります。

※ 進行中の Grant リクエストがあるとしても、RC 側で安全に鍵をローテートする仕組みが必要です（まだない）。

## 8.1 Detached JWS

この方式を採用する場合`proof`フィールドに`jwsd`を記載します。JWS署名([RFC7515](https://tools.ietf.org/html/rfc7515))は以下の手順で生成します。

* JWS ヘッダーにある `kid` フィールドはこの RC のこのリクエストにひも付きます。
* `alg` フィールドはJWSに利用されるアルゴリズムを記載し、`none` に指定してはなりません。
* `b64` フィールドは `false` にしなければなりません。
* `crit` フィールドは少なくとも `b64` フィールドを含めなければなりません（[RFC7797](https://tools.ietf.org/html/rfc7797)）

更に、追加で以下のフィールドをヘッダーに入れなければなりません。

* `htm`: ASCII大文字の文字列。このリクエストに使われた HTTP Method 名。
* `htu`: すべてのパスとクエリパラメータを含めたこのリクエストの HTTP URI。
* `ts`: このリクエストが送信されたタイムスタンプ（integer 秒）。
* `at_hash`: リクエストをアクセストークンにバインドする場合のアクセストークンのハッシュ値。ハッシュアルゴリズムは`alg`ヘッダーに記載されているハッシュアルゴリズムに一致し、ハッシュ計算結果の左半分をとってbase64urlでエンコードし、結果が`at_hash`の値になる。例えば`alg`の値が`RS256`の場合、アクセストークンを`SHA-256`でハッシュ値を計算し、左からの`128bit`をとって`base64url`エンコードを行う。

JWS の payload はリクエストボディをシリアライズしたもので、detached JWS ([RFC7797](https://tools.ietf.org/html/rfc7797))によって署名されます。

RC はこの署名をDetached-JWS HTTPヘッダにて AS に提示します。

```
POST /tx HTTP/1.1
Host: server.example.com
Content-Type: application/json
Detached-JWS: eyJiNjQiOmZhbHNlLCJhbGciOiJSUzI1NiIsImtpZCI6Inh5ei0xIn0.
  .Y287HMtaY0EegEjoTd_04a4GC6qV48GgVbGKOhHdJnDtD0VuUlVjLfwne8AuUY3U7e8
  9zUWwXLnAYK_BiS84M8EsrFvmv8yDLWzqveeIpcN5_ysveQnYt9Dqi32w6IOtAywkNUD
  ZeJEdc3z5s9Ei8qrYFN2fxcu28YS4e8e_cHTK57003WJu-wFn2TJUmAbHuqvUsyTb-nz
  YOKxuCKlqQItJF7E-cwSb_xULu-3f77BEU_vGbNYo5ZBa2B7UHO-kWNMSgbW2yeNNLbL
  C18Kv80GF22Y7SbZt0e2TwnR2Aa2zksuUbntQ5c7a1-gxtnXzuIKa34OekrnyqE1hmVW
  peQ

{
    "resources": [
        "dolphin-metadata"
    ],
    "interact": {
        "redirect": true,
        "callback": {
            "method": "redirect",
            "uri": "https://client.foo",
            "nonce": "VJLO6A4CAYLBXHTR0KRO"
        }
    },
    "client": {
      "proof": "jwsd",
      "key": {
        "jwk": {
                    "kty": "RSA",
                    "e": "AQAB",
                    "kid": "xyz-1",
                    "alg": "RS256",
                    "n": "kOB5rR4Jv0GMeLaY6_It_r3ORwdf8ci_JtffXyaSx8
xYJCNaOKNJn_Oz0YhdHbXTeWO5AoyspDWJbN5w_7bdWDxgpD-y6jnD1u9YhBOCWObNPF
vpkTM8LC7SdXGRKx2k8Me2r_GssYlyRpqvpBlY5-ejCywKRBfctRcnhTTGNztbbDBUyD
SWmFMVCHe5mXT4cL0BwrZC6S-uu-LAx06aKwQOPwYOGOslK8WPm1yGdkaA1uF_FpS6LS
63WYPHi_Ap2B7_8Wbw4ttzbMS_doJvuDagW8A1Ip3fXFAHtRAcKw7rdI4_Xln66hJxFe
kpdfWdiPQddQ6Y1cK2U3obvUg7w"
        }
      }
      "display": {
        "name": "My Client Display Name",
        "uri": "https://example.net/client"
      },
    }
}
```

HTTP GET, OPTIONS, DELETE などリクエストにメッセージボディがない場合、JWS は空の payload を利用して署名を計算します。

AS あるいは RS が Detached-JWS ヘッダを受信したら、必ずそれを detached JWS オブジェクトにパースします。HTTP ボディは変更なしで JWS の検証に payload として利用できます。


// Detached JWS とは？ --> https://tools.ietf.org/html/rfc7515#appendix-F
![スクリーンショット 2020-11-08 16.05.39.png](resources/99F5179D9115C3F9D6AB828BE70FB7F4.png =1569x848)

## 8.2 Attached JWS

この方式を採用する場合、`proof`フィールドに`jws`を記載します。JWS 署名オブジェクトは以下のように生成されます：

* JWS ヘッダーにある `kid` フィールドはこの RC のこのリクエストにひも付きます。
* `alg` フィールドはJWSに利用されるアルゴリズムを記載し、`none` に指定してはなりません。

更に、追加で以下のフィールドをヘッダーに入れなければなりません。

* `htm`: ASCII大文字の文字列。このリクエストに使われた HTTP Method 名。
* `htu`: すべてのパスとクエリパラメータを含めたこのリクエストの HTTP URI。
* `ts`: このリクエストが送信されたタイムスタンプ（integer 秒）。
* `at_hash`: リクエストをアクセストークンにバインドする場合のアクセストークンのハッシュ値。ハッシュアルゴリズムは`alg`ヘッダーに記載されているハッシュアルゴリズムに一致し、ハッシュ計算結果の左半分をとってbase64urlでエンコードし、結果が`at_hash`の値になる。例えば`alg`の値が`RS256`の場合、アクセストークンを`SHA-256`でハッシュ値を計算し、左からの`128bit`をとって`base64url`エンコードを行う。

※ b64, crit 以外に Detached JWS とは同じ

JWS の payload 部分はJSONリクエストボディで、JWS に仕様によって署名、シリアライズされます（[RFC7515](https://tools.ietf.org/html/rfc7515)）。

RC はこの JWS をリクエストボディに入れ、`content-type: application/jose` で AS に送信します。AS は必ずJWSをリクエストボディとして利用しなければなりません。

```
POST /tx HTTP/1.1
Host: server.example.com
Content-Type: application/jose

eyJhbGciOiJSUzI1NiIsImtpZCI6IktBZ05wV2JSeXk5T
WYycmlrbDQ5OExUaE1ydmtiWldIVlNRT0JDNFZIVTQiLC
JodG0iOiJwb3N0IiwiaHR1IjoiL3R4IiwidHMiOjE2MDM
4MDA3ODN9.eyJjYXBhYmlsaXRpZXMiOltdLCJjbGllbnQ
iOnsia2V5Ijp7Imp3ayI6eyJrdHkiOiJSU0EiLCJlIjoi
QVFBQiIsImtpZCI6IktBZ05wV2JSeXk5TWYycmlrbDQ5O
ExUaE1ydmtiWldIVlNRT0JDNFZIVTQiLCJuIjoibGxXbU
hGOFhBMktOTGRteE9QM2t4RDlPWTc2cDBTcjM3amZoejk
0YTkzeG0yRk5xb1NQY1JaQVBkMGxxRFM4TjNVaWE1M2RC
MjNaNTlPd1k0YnBNX1ZmOEdKdnZwdExXbnhvMVB5aG1Qc
i1lY2RTQ1JRZFRjX1pjTUY0aFJWNDhxcWx2dUQwbXF0Y0
RiSWtTQkR2Y2NKbVpId2ZUcERIaW5UOHR0dmNWUDhWa0F
NQXE0a1ZhenhPcE1vSVJzb3lFcF9lQ2U1cFN3cUhvMGRh
Q1dOS1ItRXBLbTZOaU90ZWRGNE91bXQ4TkxLVFZqZllnR
khlQkRkQ2JyckVUZDR2Qk13RHRBbmpQcjNDVkN3d3gyYk
FRVDZTbHhGSjNmajJoaHlJcHE3cGM4clppYjVqTnlYS3d
mQnVrVFZZWm96a3NodC1Mb2h5QVNhS3BZVHA4THROWi13
In0sInByb29mIjoiandzIn0sIm5hbWUiOiJNeSBGaXN0I
ENsaWVudCIsInVyaSI6Imh0dHA6Ly9sb2NhbGhvc3QvY2
xpZW50L2NsaWVudElEIn0sImludGVyYWN0Ijp7ImNhbGx
iYWNrIjp7Im1ldGhvZCI6InJlZGlyZWN0Iiwibm9uY2Ui
OiJkOTAyMTM4ODRiODQwOTIwNTM4YjVjNTEiLCJ1cmkiO
iJodHRwOi8vbG9jYWxob3N0L2NsaWVudC9yZXF1ZXN0LW
RvbmUifSwicmVkaXJlY3QiOnRydWV9LCJyZXNvdXJjZXM
iOnsiYWN0aW9ucyI6WyJyZWFkIiwicHJpbnQiXSwibG9j
YXRpb25zIjpbImh0dHA6Ly9sb2NhbGhvc3QvcGhvdG9zI
l0sInR5cGUiOiJwaG90by1hcGkifSwic3ViamVjdCI6ey
JzdWJfaWRzIjpbImlzcy1zdWIiLCJlbWFpbCJdfX0.LUy
Z8_fERmxbYARq8kBYMwzcd8GnCAKAlo2ZSYLRRNAYWPrp
2XGLJOvg97WK1idf_LB08OJmLVsCXxCvn9mgaAkYNL_Zj
HcusBvY1mNo0E1sdTEr31CVKfC-6WrZCscb8YqE4Ayhh0
Te8kzSng3OkLdy7xN4xeKuHzpF7yGsM52JZ0cBcTo6WrY
EfGdr08AWQJ59ht72n3jTsmYNy9A6I4Wrvfgj3TNxmwYo
jpBAicfjnzA1UVcNm9F_xiSz1_y2tdH7j5rVqBMQife-k
9Ewk95vr3lurthenliYSNiUinVfoW1ybnaIBcTtP1_YCx
g_h1y-B5uZEvYNGCuoCqa6IQ
```

デコード後の JWS ヘッダーは

```
{
  "alg": "RS256",
  "kid": "KAgNpWbRyy9Mf2rikl498LThMrvkbZWHVSQOBC4VHU4",
  "htm": "post",
  "htu": "/tx",
  "ts": 1603800783
}
```

ペイロードは

```
{
  "capabilities": [],
  "client": {
    "key": {
      "jwk": {
        "kty": "RSA",
        "e": "AQAB",
        "kid": "KAgNpWbRyy9Mf2rikl498LThMrvkbZWHVSQOBC4VHU4",
        "n": "llWmHF8XA2KNLdmxOP3kxD9OY76p0Sr37jfhz94a93xm2FNqoSPcRZAPd0lqDS8N3Uia53dB23Z59OwY4bpM_Vf8GJvvptLWnxo1PyhmPr-ecdSCRQdTc_ZcMF4hRV48qqlvuD0mqtcDbIkSBDvccJmZHwfTpDHinT8ttvcVP8VkAMAq4kVazxOpMoIRsoyEp_eCe5pSwqHo0daCWNKR-EpKm6NiOtedF4Oumt8NLKTVjfYgFHeBDdCbrrETd4vBMwDtAnjPr3CVCwwx2bAQT6SlxFJ3fj2hhyIpq7pc8rZib5jNyXKwfBukTVYZozksht-LohyASaKpYTp8LtNZ-w"
      },
      "proof": "jws"
    },
    "name": "My Fist Client",
    "uri": "http://localhost/client/clientID"
  },
  "interact": {
    "callback": {
      "method": "redirect",
      "nonce": "d90213884b840920538b5c51",
      "uri": "http://localhost/client/request-done"
    },
    "redirect": true
  },
  "resources": {
    "actions": [
      "read",
      "print"
    ],
    "locations": [
      "http://localhost/photos"
    ],
    "type": "photo-api"
  },
  "subject": {
    "sub_ids": [
      "iss-sub",
      "email"
    ]
  }
}
```

Detached JWS と同様に、メッセージボディが空の場合、JWS 署名は空のペイロードを使って計算します。

## 8.3 Mutual TLS

この方式を利用するには、`proof`フィールドに`mtls`を記載します。
RC はクライアント証明書を AS あるいは RS とTLSネゴシエーションするときに提示します。
[RFC8705 Section 3](https://tools.ietf.org/html/rfc8705#section-3)の記載通りに、AS あるいは RS は RC が提示したクライアント証明書のフィンガープリントをTLSネゴシエーション時に送信されたフィンガープリントと比較します。

```
POST /tx HTTP/1.1
Host: server.example.com
Content-Type: application/json
SSL_CLIENT_CERT: MIIEHDCCAwSgAwIBAgIBATANBgkqhkiG9w0BAQsFADCBmjE3MDUGA1UEAwwuQmVz
 cG9rZSBFbmdpbmVlcmluZyBSb290IENlcnRpZmljYXRlIEF1dGhvcml0eTELMAkG
 A1UECAwCTUExCzAJBgNVBAYTAlVTMRkwFwYJKoZIhvcNAQkBFgpjYUBic3BrLmlv
 MRwwGgYDVQQKDBNCZXNwb2tlIEVuZ2luZWVyaW5nMQwwCgYDVQQLDANNVEkwHhcN
 MTkwNDEwMjE0MDI5WhcNMjQwNDA4MjE0MDI5WjB8MRIwEAYDVQQDDAlsb2NhbGhv
 c3QxCzAJBgNVBAgMAk1BMQswCQYDVQQGEwJVUzEgMB4GCSqGSIb3DQEJARYRdGxz
 Y2xpZW50QGJzcGsuaW8xHDAaBgNVBAoME0Jlc3Bva2UgRW5naW5lZXJpbmcxDDAK
 BgNVBAsMA01USTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMmaXQHb
 s/wc1RpsQ6Orzf6rN+q2ijaZbQxD8oi+XaaN0P/gnE13JqQduvdq77OmJ4bQLokq
 sd0BexnI07Njsl8nkDDYpe8rNve5TjyUDCfbwgS7U1CluYenXmNQbaYNDOmCdHww
 UjV4kKREg6DGAx22Oq7+VHPTeeFgyw4kQgWRSfDENWY3KUXJlb/vKR6lQ+aOJytk
 vj8kVZQtWupPbvwoJe0na/ISNAOhL74w20DWWoDKoNltXsEtflNljVoi5nqsmZQc
 jfjt6LO0T7O1OX3Cwu2xWx8KZ3n/2ocuRqKEJHqUGfeDtuQNt6Jz79v/OTr8puLW
 aD+uyk6NbtGjoQsCAwEAAaOBiTCBhjAJBgNVHRMEAjAAMAsGA1UdDwQEAwIF4DBs
 BgNVHREEZTBjgglsb2NhbGhvc3SCD3Rsc2NsaWVudC5sb2NhbIcEwKgBBIERdGxz
 Y2xpZW50QGJzcGsuaW+GF2h0dHA6Ly90bHNjbGllbnQubG9jYWwvhhNzc2g6dGxz
 Y2xpZW50LmxvY2FsMA0GCSqGSIb3DQEBCwUAA4IBAQCKKv8WlLrT4Z5NazaUrYtl
 TF+2v0tvZBQ7qzJQjlOqAcvxry/d2zyhiRCRS/v318YCJBEv4Iq2W3I3JMMyAYEe
 2573HzT7rH3xQP12yZyRQnetdiVM1Z1KaXwfrPDLs72hUeELtxIcfZ0M085jLboX
 hufHI6kqm3NCyCCTihe2ck5RmCc5l2KBO/vAHF0ihhFOOOby1v6qbPHQcxAU6rEb
 907/p6BW/LV1NCgYB1QtFSfGxowqb9FRIMD2kvMSmO0EMxgwZ6k6spa+jk0IsI3k
 lwLW9b+Tfn/daUbIDctxeJneq2anQyU2znBgQl6KILDSF4eaOqlBut/KNZHHazJh

{
    "resources": [
        "dolphin-metadata"
    ],
    "interact": {
        "redirect": true,
        "callback": {
            "method": "redirect",
            "uri": "https://client.foo",
            "nonce": "VJLO6A4CAYLBXHTR0KRO"
        }
    },
    "client": {
      "display": {
        "name": "My Client Display Name",
        "uri": "https://example.net/client"
      },
      "key": {
        "proof": "mtls",
        "cert": "MIIEHDCCAwSgAwIBAgIBATANBgkqhkiG9w0BAQsFADCBmjE3
MDUGA1UEAwwuQmVzcG9rZSBFbmdpbmVlcmluZyBSb290IENlcnRpZmljYXRlIEF1d
Ghvcml0eTELMAkGA1UECAwCTUExCzAJBgNVBAYTAlVTMRkwFwYJKoZIhvcNAQkBFg
pjYUBic3BrLmlvMRwwGgYDVQQKDBNCZXNwb2tlIEVuZ2luZWVyaW5nMQwwCgYDVQQ
LDANNVEkwHhcNMTkwNDEwMjE0MDI5WhcNMjQwNDA4MjE0MDI5WjB8MRIwEAYDVQQD
DAlsb2NhbGhvc3QxCzAJBgNVBAgMAk1BMQswCQYDVQQGEwJVUzEgMB4GCSqGSIb3D
QEJARYRdGxzY2xpZW50QGJzcGsuaW8xHDAaBgNVBAoME0Jlc3Bva2UgRW5naW5lZX
JpbmcxDDAKBgNVBAsMA01USTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggE
BAMmaXQHbs/wc1RpsQ6Orzf6rN+q2ijaZbQxD8oi+XaaN0P/gnE13JqQduvdq77Om
J4bQLokqsd0BexnI07Njsl8nkDDYpe8rNve5TjyUDCfbwgS7U1CluYenXmNQbaYND
OmCdHwwUjV4kKREg6DGAx22Oq7+VHPTeeFgyw4kQgWRSfDENWY3KUXJlb/vKR6lQ+
aOJytkvj8kVZQtWupPbvwoJe0na/ISNAOhL74w20DWWoDKoNltXsEtflNljVoi5nq
smZQcjfjt6LO0T7O1OX3Cwu2xWx8KZ3n/2ocuRqKEJHqUGfeDtuQNt6Jz79v/OTr8
puLWaD+uyk6NbtGjoQsCAwEAAaOBiTCBhjAJBgNVHRMEAjAAMAsGA1UdDwQEAwIF4
DBsBgNVHREEZTBjgglsb2NhbGhvc3SCD3Rsc2NsaWVudC5sb2NhbIcEwKgBBIERdG
xzY2xpZW50QGJzcGsuaW+GF2h0dHA6Ly90bHNjbGllbnQubG9jYWwvhhNzc2g6dGx
zY2xpZW50LmxvY2FsMA0GCSqGSIb3DQEBCwUAA4IBAQCKKv8WlLrT4Z5NazaUrYtl
TF+2v0tvZBQ7qzJQjlOqAcvxry/d2zyhiRCRS/v318YCJBEv4Iq2W3I3JMMyAYEe2
573HzT7rH3xQP12yZyRQnetdiVM1Z1KaXwfrPDLs72hUeELtxIcfZ0M085jLboXhu
fHI6kqm3NCyCCTihe2ck5RmCc5l2KBO/vAHF0ihhFOOOby1v6qbPHQcxAU6rEb907
/p6BW/LV1NCgYB1QtFSfGxowqb9FRIMD2kvMSmO0EMxgwZ6k6spa+jk0IsI3klwLW
9b+Tfn/daUbIDctxeJneq2anQyU2znBgQl6KILDSF4eaOqlBut/KNZHHazJh"
    }
}
```

※ サーバの実装はmTLSを利用している場合、その部分（ネットワークレイヤー）の実装は特に変わることありません。しかし AS / RS を実装時に、アプリケーションレイヤーの実装でリクエストに含まれる証明書の検証を実施する必要があります。

## 8.4 Demonstration of Proof-of-Possession (DPoP)

この方式を利用する場合、`proof`フィールドに`dpop`を記載します。
RC は [draft-ietf-oauth-dpop-01 Section 2](https://tools.ietf.org/html/draft-ietf-oauth-dpop-01) の説明通りに DPoP 署名ヘッダーを生成します。DPoP ボディには必ずリクエストボディのダイジェスト（[RFC3230](https://tools.ietf.org/html/rfc3230)）を含める必要があります。

```
POST /tx HTTP/1.1
Host: server.example.com
Content-Type: application/json
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IlJTMjU2IiwiandrIjp7Imt0eSI6Il
JTQSIsImUiOiJBUUFCIiwia2lkIjoieHl6LWNsaWVudCIsImFsZyI6IlJTMjU2Iiwibi
I6Inp3Q1RfM2J4LWdsYmJIcmhlWXBZcFJXaVk5SS1uRWFNUnBablJySWpDczZiX2VteV
RrQmtEREVqU3lzaTM4T0M3M2hqMS1XZ3hjUGRLTkdaeUlvSDNRWmVuMU1LeXloUXBMSk
cxLW9MTkxxbTdwWFh0ZFl6U2RDOU8zLW9peXk4eWtPNFlVeU5aclJSZlBjaWhkUUNiT1
9PQzhRdWdtZzlyZ05ET1NxcHBkYU5lYXMxb3Y5UHhZdnhxcnoxLThIYTdna0QwMFlFQ1
hIYUIwNXVNYVVhZEhxLU9fV0l2WVhpY2c2STVqNlM0NFZOVTY1VkJ3dS1BbHluVHhRZE
1BV1AzYll4VlZ5NnAzLTdlVEpva3ZqWVRGcWdEVkRaOGxVWGJyNXlDVG5SaG5oSmd2Zj
NWakRfbWFsTmU4LXRPcUs1T1NEbEhUeTZnRDlOcWRHQ20tUG0zUSJ9fQ.eyJodHRwX21
ldGhvZCI6IlBPU1QiLCJodHRwX3VyaSI6Imh0dHA6XC9cL2hvc3QuZG9ja2VyLmludGV
ybmFsOjk4MzRcL2FwaVwvYXNcL3RyYW5zYWN0aW9uIiwiaWF0IjoxNTcyNjQyNjEzLCJ
qdGkiOiJIam9IcmpnbTJ5QjR4N2pBNXl5RyJ9.aUhftvfw2NoW3M7durkopReTvONng1
fOzbWjAlKNSLL0qIwDgfG39XUyNvwQ23OBIwe6IuvTQ2UBBPklPAfJhDTKd8KHEAfidN
B-LzUOzhDetLg30yLFzIpcEBMLCjb0TEsmXadvxuNkEzFRL-Q-QCg0AXSF1h57eAqZV8
SYF4CQK9OUV6fIWwxLDd3cVTx83MgyCNnvFlG_HDyim1Xx-rxV4ePd1vgDeRubFb6QWj
iKEO7vj1APv32dsux67gZYiUpjm0wEZprjlG0a07R984KLeK1XPjXgViEwEdlirUmpVy
T9tyEYqGrTfm5uautELgMls9sgSyE929woZ59elg

{
    "resources": [
        "dolphin-metadata"
    ],
    "interact": {
        "redirect": true,
        "callback": {
            "method": "redirect",
            "uri": "https://client.foo",
            "nonce": "VJLO6A4CAYLBXHTR0KRO"
        }
    },
    "client": {
      "display": {
        "name": "My Client Display Name",
        "uri": "https://example.net/client"
      },
      "proof": "dpop",
      "key": {
        "jwk": {
                    "kty": "RSA",
                    "e": "AQAB",
                    "kid": "xyz-1",
                    "alg": "RS256",
                    "n": "kOB5rR4Jv0GMeLaY6_It_r3ORwdf8ci_JtffXyaSx8xYJ
CCNaOKNJn_Oz0YhdHbXTeWO5AoyspDWJbN5w_7bdWDxgpD-y6jnD1u9YhBOCWObNPFvpkTM
8LC7SdXGRKx2k8Me2r_GssYlyRpqvpBlY5-ejCywKRBfctRcnhTTGNztbbDBUyDSWmFMVCH
e5mXT4cL0BwrZC6S-uu-LAx06aKwQOPwYOGOslK8WPm1yGdkaA1uF_FpS6LS63WYPHi_Ap2
B7_8Wbw4ttzbMS_doJvuDagW8A1Ip3fXFAHtRAcKw7rdI4_Xln66hJxFekpdfWdiPQddQ6Y
1cK2U3obvUg7w"
        }
      }
    }
}
```

※ 仕様に提示された例に Digest Header の記載はなかったけど、おそらく記載漏れ？以下のようなヘッダーを記載する必要があるかと。

```
Digest: SHA=thvDyvhfIqlvFe+A9MYgxAfm1q5=
```

## 8.5 HTTP Signing

この方式を採用する場合、`proof`フィールドに`httpsig`を記載します。
RC が HTTP Signature Header （[draft-ietf-httpbis-message-signatures, Section 4](https://tools.ietf.org/html/draft-ietf-httpbis-message-signatures-00)）を生成します。DPoPと同様に、RC はリクエストボディのダイジェストを計算し、Digest ヘッダーに含める必要があります。

```
POST /tx HTTP/1.1
Host: server.example.com
Content-Type: application/json
Content-Length: 716
Signature: keyId="xyz-client", algorithm="rsa-sha256",
 headers="(request-target) digest content-length",
 signature="TkehmgK7GD/z4jGkmcHS67cjVRgm3zVQNlNrrXW32Wv7d
u0VNEIVI/dMhe0WlHC93NP3ms91i2WOW5r5B6qow6TNx/82/6W84p5jqF
YuYfTkKYZ69GbfqXkYV9gaT++dl5kvZQjVk+KZT1dzpAzv8hdk9nO87Xi
rj7qe2mdAGE1LLc3YvXwNxuCQh82sa5rXHqtNT1077fiDvSVYeced0UEm
rWwErVgr7sijtbTohC4FJLuJ0nG/KJUcIG/FTchW9rd6dHoBnY43+3Dzj
CIthXpdH5u4VX3TBe6GJDO6Mkzc6vB+67OWzPwhYTplUiFFV6UZCsDEeu
Sa/Ue1yLEAMg=="]}
Digest: SHA=oZz2O3kg5SEFAhmr0xEBbc4jEfo=

{
    "resources": [
        "dolphin-metadata"
    ],
    "interact": {
        "redirect": true,
        "callback": {
            "method": "push",
            "uri": "https://client.foo",
            "nonce": "VJLO6A4CAYLBXHTR0KRO"
        }
    },
    "client": {
      "display": {
        "name": "My Client Display Name",
        "uri": "https://example.net/client"
      },
      "proof": "httpsig",
      "key": {
        "jwk": {
                    "kty": "RSA",
                    "e": "AQAB",
                    "kid": "xyz-1",
                    "alg": "RS256",
                    "n": "kOB5rR4Jv0GMeLaY6_It_r3ORwdf8ci_J
tffXyaSx8xYJCCNaOKNJn_Oz0YhdHbXTeWO5AoyspDWJbN5w_7bdWDxgpD-
y6jnD1u9YhBOCWObNPFvpkTM8LC7SdXGRKx2k8Me2r_GssYlyRpqvpBlY5-
ejCywKRBfctRcnhTTGNztbbDBUyDSWmFMVCHe5mXT4cL0BwrZC6S-uu-LAx
06aKwQOPwYOGOslK8WPm1yGdkaA1uF_FpS6LS63WYPHi_Ap2B7_8Wbw4ttz
bMS_doJvuDagW8A1Ip3fXFAHtRAcKw7rdI4_Xln66hJxFekpdfWdiPQddQ6
Y1cK2U3obvUg7w"
        }
      }
    }
}
```

アクセストークンもASに提示する場合、認可ヘッダーも署名に含まれなければなりません。

## 8.6 OAuth Proof of Possession (PoP)

この方式を利用するには、`proof`フィールドに`oauthpop`を記載します。
RC は HTTP Authorization PoP ヘッダー（[draft-ietf-oauth-signed-http-request-03](https://tools.ietf.org/html/draft-ietf-oauth-signed-http-request-03)）を作成します。追加で以下の要件も含めます。

* アクセストークンを利用しないリクエストでは、`at`(access token)フィールドを省略しなければならない
* HTTP GET, OPTION, DELETE などリクエストボディがないケースを除いで、`b`(body hash)フィールドは必ず計算し含めかねればならない
* URL のすべてのコンポネントを計算し含めなければならな
* `m`(method)フィールドを必ず含めなければならない

```
POST /tx HTTP/1.1
Host: server.example.com
Content-Type: application/json
PoP: eyJhbGciOiJSUzI1NiIsImp3ayI6eyJrdHkiOiJSU0EiLCJlIjoi
QVFBQiIsImtpZCI6Inh5ei1jbGllbnQiLCJhbGciOiJSUzI1NiIsIm4iO
iJ6d0NUXzNieC1nbGJiSHJoZVlwWXBSV2lZOUktbkVhTVJwWm5ScklqQ3
M2Yl9lbXlUa0JrRERFalN5c2kzOE9DNzNoajEtV2d4Y1BkS05HWnlJb0g
zUVplbjFNS3l5aFFwTEpHMS1vTE5McW03cFhYdGRZelNkQzlPMy1vaXl5
OHlrTzRZVXlOWnJSUmZQY2loZFFDYk9fT0M4UXVnbWc5cmdORE9TcXBwZ
GFOZWFzMW92OVB4WXZ4cXJ6MS04SGE3Z2tEMDBZRUNYSGFCMDV1TWFVYW
RIcS1PX1dJdllYaWNnNkk1ajZTNDRWTlU2NVZCd3UtQWx5blR4UWRNQVd
QM2JZeFZWeTZwMy03ZVRKb2t2allURnFnRFZEWjhsVVhicjV5Q1RuUmhu
aEpndmYzVmpEX21hbE5lOC10T3FLNU9TRGxIVHk2Z0Q5TnFkR0NtLVBtM
1EifX0.eyJwIjoiXC9hcGlcL2FzXC90cmFuc2FjdGlvbiIsImIiOiJxa0
lPYkdOeERhZVBTZnc3NnFjamtqSXNFRmxDb3g5bTU5NFM0M0RkU0xBIiw
idSI6Imhvc3QuZG9ja2VyLmludGVybmFsIiwiaCI6W1siQWNjZXB0Iiwi
Q29udGVudC1UeXBlIiwiQ29udGVudC1MZW5ndGgiXSwiVjQ2OUhFWGx6S
k9kQTZmQU5oMmpKdFhTd3pjSGRqMUloOGk5M0h3bEVHYyJdLCJtIjoiUE
9TVCIsInRzIjoxNTcyNjQyNjEwfQ.xyQ47qy8bu4fyK1T3Ru1Sway8wp6
5rfAKnTQQU92AUUU07I2iKoBL2tipBcNCC5zLH5j_WUyjlN15oi_lLHym
fPdzihtt8_Jibjfjib5J15UlifakjQ0rHX04tPal9PvcjwnyZHFcKn-So
Y3wsARn-gGwxpzbsPhiKQP70d2eG0CYQMA6rTLslT7GgdQheelhVFW29i
27NcvqtkJmiAG6Swrq4uUgCY3zRotROkJ13qo86t2DXklV-eES4-2dCxf
cWFkzBAr6oC4Qp7HnY_5UT6IWkRJt3efwYprWcYouOVjtRan3kEtWkaWr
G0J4bPVnTI5St9hJYvvh7FE8JirIg

{
    "resources": [
        "dolphin-metadata"
    ],
    "interact": {
        "redirect": true,
        "callback": {
            "method": "redirect",
            "uri": "https://client.foo",
            "nonce": "VJLO6A4CAYLBXHTR0KRO"
        }
    },
    "client": {
      "display": {
        "name": "My Client Display Name",
        "uri": "https://example.net/client"
      },
      "proof": "oauthpop",
      "key": {
        "jwk": {
                    "kty": "RSA",
                    "e": "AQAB",
                    "kid": "xyz-1",
                    "alg": "RS256",
                    "n": "kOB5rR4Jv0GMeLaY6_It_r3ORwdf8ci_J
tffXyaSx8xYJCCNaOKNJn_Oz0YhdHbXTeWO5AoyspDWJbN5w_7bdWDxgpD-
y6jnD1u9YhBOCWObNPFvpkTM8LC7SdXGRKx2k8Me2r_GssYlyRpqvpBlY5-
ejCywKRBfctRcnhTTGNztbbDBUyDSWmFMVCHe5mXT4cL0BwrZC6S-uu-LAx
06aKwQOPwYOGOslK8WPm1yGdkaA1uF_FpS6LS63WYPHi_Ap2B7_8Wbw4ttz
bMS_doJvuDagW8A1Ip3fXFAHtRAcKw7rdI4_Xln66hJxFekpdfWdiPQddQ6
Y1cK2U3obvUg7w"
        }
      }
    }
}
```

# 9. ディスカバリー（discovery）

この仕様は、意図的に事前のdiscoveryの必要性を最小限に抑えています。RC は AS のエンドポイントとリクエストを署名するための鍵の種類さえ知っていればリクエストを送信できます。その他の情報は、AS と RC のやり取りの中に動的に含まれます。

しかし AS が持つ機能に制限があります。RC が AS に対するリクエストを送信する前に通信を最適化にしたい場合、Grant リクエストのエンドポイントに HTTP OPTION メソッドでリクエストを送信することで、サーバのディスカバリー情報を取得できます。その際、AS は JSON 形式で以下の情報を含めたレスポンスを返します。

| name | type | note |
| ---- | ---- | ---- |
| grant_request_endpoint | REQUIRED | AS の Grant リクエストエンドポイントの URL （フルパス）。<br/>ディスカバリーリクエストの投入先のURLと一致する。 |
| capabilities | OPTIONAL | AS の capability 一覧。<br/>RC が送信する `capability` （Section 2.6）フィールド値はその中から選択する可能。 |
| interaction_methods | OPTIONAL | AS のインタラクションモードの一覧。<br/>リクエストにある `interact`（Section 2.5）フィールド値はこのリストに対応する。<br/>例：redirect, app, callback, user_code |
| key_proofs | OPTIONAL | AS がサポートする鍵証明の仕組み一覧。<br/>リクエストにある`key.proof`（Section 2.3.1）フィールドに対応する。<br/>例：jwsd |
| sub_ids | OPTIONAL | AS がサポートする識別子種類一覧。<br/>リクエストにある `subject.sub_ids`（Section 2.2）フィールドに対応する。<br/>例：iss-sub, email |
| assertions | OPTIONAL | AS がサポートするアサーション種類一覧。<br/>リクエストにある `subject.assertions` （Section 2.2）フィールドに対応する。<br/>例：id_token, saml2 |

これらの情報は、最適化の目的にしか利用できません。かりに discovery にとある機能をサポートすると記載しても、AS はその機能を利用しているリクエストを拒否できます。

# 10. リソースサーバ

リソースサーバは AS と通信し、いくつかの機能実現する場合があります。

## 10.1 トークンイントロスペクション

RS がアクセストークンを受信後に、AS のイントロスペクション（introspection）エンドポイントにアクセスし、トークンの情報を取得できます。

※ 雰囲気的にトークンマネジメントAPI（Section 3.2）とはそれなりに違いがないが、RS はトークンマネジメントAPI知らないし、RC の鍵ではなく RS の鍵にバインドすると考えるとややこしくなりそうなので、別のエンドポイントにしているそうです。

```
+------+       +------+       +------+
|  RC  |--(1)->|  RS  |       |  AS  |
|      |       |      |--(2)->|      |
|      |       |      |<-(3)--|      |
|      |       |      |       +------+
|      |<-(4)--|      |
+------+       +------+
```

(1) RC は RS をアクセストークン付きでアクセスします。
(2) RS はトークンを AS に送り、イントロスペクトします。その際に、RS は自分の鍵でリクエストを署名する（RCの鍵やトークンの鍵ではなく）。アクセストークンはリクエストボディで送ります。

```
POST /introspect HTTP/1.1
Host: server.example.com
Content-type: application/json
Detached-JWS: ejy0...

{
    "access_token": "OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0",
}
```

(3) AS はトークンの値とリクエスト自体を検証し、イントロスペクションレスポンスを返却します。
レスポンスには、トークン現在のステータスと RS がトークンの有効性を検証するに必要な任意の情報（鍵証明の仕組みや公開鍵など）が含まれます。

```
Content-type: application/json

{
    "active": true,
    "resources": [
        "dolphin-metadata", "some other thing"
    ],
    "client": {
      "key": {
        "proof": "httpsig",
        "jwk": {
                    "kty": "RSA",
                    "e": "AQAB",
                    "kid": "xyz-1",
                    "alg": "RS256",
                    "n": "kOB5rR4Jv0GMeL...."
        }
      }
    }
}
```

(4) RS は RC のリクエストを実行する。

## 10.2 ダウンストリームのトークンの派生

アーキテクチャによって、RS が RC として振る舞い、派生したアクセストークンを取得し、第2の RS にリクエストする可能性があります。
この内部的なトークンはもととなるアクセストークンのコンテキストによって発行されます。

```
+------+       +-------+       +------+       +-------+
|  RC  |--(1)->|  RS1  |       |  AS  |       |  RS2  |
|      |       |       |--(2)->|      |       |       |
|      |       |       |<-(3)--|      |       |       |
|      |       |       |       +------+       |       |
|      |       |       |                      |       |
|      |       |       |-----------(4)------->|       |
|      |       |       |<----------(5)--------|       |
|      |<-(6)--|       |                      |       |
+------+       +-------+                      +-------+
```

(1) RC が RS1 をアクセストークン付きで呼び出します。
(2) RS1 はアクセストークンを AS に提示し、RS2 上に利用する派生トークンを取得します。RS1 は RO とインタラクションを行う能力を持っていないことを示します。RS1 は RC やトークンの鍵ではなく、自分の鍵でリクエストを署名します。
リクエストのフォーマットはSection 2で記載されたものに一致します。その際に既存のアクセストークンを`existing_access_token`フィールドに入れ AS に提示します。
RS は自分のアイデンティティを証明するために、自分の鍵でリクエストを署名する必要があります。

```
POST /tx HTTP/1.1
Host: server.example.com
Content-type: application/json
Detached-JWS: ejy0...

{
    "resources": [
        {
            "actions": [
                "read",
                "write",
                "dolphin"
            ],
            "locations": [
                "https://server.example.net/",
                "https://resource.local/other"
            ],
            "datatypes": [
                "metadata",
                "images"
            ]
        },
        "dolphin-metadata"
    ],
    "client": "7C7C4AZ9KHRS6X63AJAO",
    "existing_access_token": "OS9M2PMHKUR64TB8N6BW7OZB8CDFONP219RP1LT0"
}
```

(3) AS が派生トークンを RS1 に返します。レスポンスはSection 3で記載されているフォーマットに一致します。
(4) RS1 が派生トークンを利用して RS2 を呼び出します。
(5) RS2 が RS1 のリクエストを実行します。
(6) RS1 が RC のリクエストを実行します。

## 10.3 リソースハンドル（Resource Handle）を登録する

必要があれば RS は AS のリソース登録エンドポイントに、リソースセット（Section 2.1.1）を送信できます。
RS は自分のアイデンティティを証明するために、自分の鍵でリクエストを署名します。

```
POST /resource HTTP/1.1
Host: server.example.com
Content-type: application/json
Detached-JWS: ejy0...

{
    "resources": [
        {
            "actions": [
                "read",
                "write",
                "dolphin"
            ],
            "locations": [
                "https://server.example.net/",
                "https://resource.local/other"
            ],
            "datatypes": [
                "metadata",
                "images"
            ]
        },
        "dolphin-metadata"
    ],
    "client": "7C7C4AZ9KHRS6X63AJAO"

}
```

AS はリソースハンドルを返却します。そのハンドルは RS が提示したリソースリストにひも付きます。

```
Content-type: application/json

{
    "resource_handle": "FWWIKYBQ6U56NL1"
}
```

RS はこのハンドルをレスポンスの一部として利用してもいいし、開発者に提供するドキュメントに記載してもよいです。

## 10.4 アクセス権限不足の場合のリソース請求

RC がトークンなし、あるいは失効したトークンを利用して RS を呼び出しするときに、RS は RC に `GNAP` を示す認証ヘッダーを返しても良いです。その中に、GNAP AS のエンドポイントの URL を `as_uri` パラメータに含めます。

また、付加的に、RS は RC に使えるかもしれないリソースハンドルを `resource` パラメータで返しても良いです。このハンドルに含まれたリソースは、少なくとも RC がリクエスト内で要求するリソースを満たす必要があります。

これを実現するために、RS は Section 10.3 で述べた仕組みを利用してハンドルを登録できますし、すでに登録済みのハンドルを利用してもよいです。

このハンドルは RC と RS にとっては opaque であります。

```
WWW-Authenticate: GNAP as_uri=http://server.example/tx,resource=FWWIKYBQ6U56NL1
```

RC は `resource` 値を Grant リクエストの `resources` （Section 2.1.1）フィールドに含め、 `as_uri` にリクエストを送ります。
そのリクエストの中で追加で別のリソースを要求しても良いし、複数のトークンを要求してもよいです。

※ 10.3, 10.4 のハンドル周りの仕組みは、UMA 2.0 の `distributed authorization` モデルを真似していました。