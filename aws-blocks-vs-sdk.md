# AWS Blocks と SDK 直書きの違いメモ

S3 の署名付き URL をローカルで試す前提で、AWS Blocks を使う場合と、AWS SDK + CDK で直接組む場合の違いを一旦整理したメモ。

## ざっくり結論

AWS Blocks は LocalStack のように AWS API 互換 endpoint をローカルに立てるものというより、アプリ機能単位の Block が「ローカル実装」「AWS デプロイ時のインフラ」「実行時 API」をまとめて持つツールとして見るのが近い。

S3 の署名付き URL だけを見るなら SDK 直書きのほうが仕組みは見えやすい。一方で、ファイルアップロード機能をアプリに早く入れたいなら AWS Blocks の `FileBucket` のほうが書く量は少なくなる。

## 比較

| 観点 | AWS Blocks | AWS SDK + CDK |
| --- | --- | --- |
| ローカル実行 | AWS アカウントなしで動かす。Block ごとのローカル実装を使う | LocalStack、MinIO、実 AWS、自前モックなどを別途用意する |
| S3 相当の扱い | `FileBucket` を使う。ローカルでは `.bb-data/`、AWS では S3 | `S3Client` で S3 互換 endpoint または実 S3 に接続する |
| 署名付き URL | `FileBucket` の API で発行する | `getSignedUrl()` と `PutObjectCommand` / `GetObjectCommand` で発行する |
| インフラ | Block から CDK construct / CloudFormation が生成される | CDK で S3 バケット、CORS、権限を自分で定義する |
| CDK 連携 | Blocks アプリ自体が CDK アプリ。必要なら CDK construct へ降りられる | 最初から CDK を直接書く |
| 抽象度 | ファイルアップロード機能として扱う | S3 API として扱う |
| デバッグ観点 | Block のローカル実装と生成される AWS リソースを見る | SDK リクエスト、署名、endpoint、S3/CDK 設定を見る |
| 向いていそうな用途 | アプリ機能を早く組む、ローカル first に試す | S3 固有の挙動、署名、CORS、IAM まで細かく確認する |

## 今回の検証で見たい差分

1. `PUT` 用の署名付き URL を発行するコード量
2. ブラウザから直接アップロードするときの CORS の扱い
3. ローカルに保存されたファイルの見え方
4. AWS にデプロイしたときに生成される S3 バケットの設定
5. SDK 直書きで必要な endpoint / credentials / path-style 設定が、Blocks ではどこまで不要になるか
6. S3 の細かい設定に降りたいとき、Blocks から CDK にどれくらい自然に移れるか

## CDK はどうなるか

AWS Blocks を使う場合、CDK を別の `infra/` アプリとして必ず分離するというより、サーバーサイドのアプリケーションコードの中に Blocks と CDK の両方が入るイメージになる。

ここでいうサーバーサイドのアプリケーションコードは、Next.js の Route Handler や Express/NestJS の controller に近い責務のコードを指している。つまり、フロントエンドから呼ばれて、署名付き URL を発行したり、データを読んだり、認可を見たりする層。

公式 docs の説明では、Blocks アプリ自体が CDK アプリで、同じ `new FileBucket(scope, "uploads")` のようなコードが、実行コンテキストによって次のように変わる。

| コンテキスト | 同じ Block 定義がどう振る舞うか |
| --- | --- |
| ローカル開発 | ファイルシステムやインメモリのローカル実装として動く |
| CDK synth | CDK construct を生成し、CloudFormation template になる |
| AWS Lambda runtime | AWS SDK 経由で実 AWS サービスを呼ぶ |

つまり、こういう分担になる。

```txt
frontend/
  Next.js の画面

server または aws-blocks/
  ApiNamespace などのサーバーサイド API
  FileBucket などの Block 定義
  必要なら CDK construct の追加設定

cdk synth / deploy
  Blocks が CDK construct を生成
  追加で書いた CDK construct も同じ template に入る
```

SDK + CDK 直書きだと、だいたいこう分かれる。

```txt
app/
  Next.js Route Handler
  S3Client
  getSignedUrl()

infra/
  new Bucket(...)
  CORS
  IAM
  output
```

AWS Blocks だと、`FileBucket` が「アプリから使う API」と「AWS に作る S3 バケット」をまたぐので、アプリケーションのバックエンドコードの中にインフラ定義も混ざる。これは雑に混ざるというより、Blocks がそういう設計になっている。

Next.js で言うなら、SDK 直書きでは `app/api/uploads/route.ts` に置いていた処理が、AWS Blocks では `ApiNamespace` の関数に寄る。その `ApiNamespace` のコードが、ローカルでは開発サーバー上で動き、AWS では Lambda と API Gateway 側に載る。

ただし、S3 の細かい設定や周辺リソースが必要になったら CDK に降りる。公式 docs でも、必要なときは CDK を直接使える、既存の CDK stack に Blocks を埋め込める、という位置づけになっている。

今回のサンプルなら最初はこう捉える。

```ts
// aws-blocks/index.ts みたいなサーバーサイド定義
import { ApiNamespace, FileBucket, Scope } from "@aws-blocks/blocks";

const scope = new Scope("file-upload-app");
const uploads = new FileBucket(scope, "uploads");

export const api = new ApiNamespace(scope, "api", () => ({
  async createUploadUrl(path: string, contentType: string) {
    return {
      uploadUrl: await uploads.putUrl(path, {
        contentType,
        expiresIn: 600,
      }),
    };
  },
}));
```

この `FileBucket` がローカルでは `.bb-data/`、AWS では S3 バケットになる。なので「CDK はどこ？」への答えは、Blocks の裏側に CDK synth の層があり、必要なときだけ同じアプリ内で CDK construct を直接足す、になる。

## 既存の `/client` `/server` `/cdk` 構成からどう変わるか

これまでの構成がこうだとする。

```txt
/client
  React / Next.js frontend
  build artifact -> S3 + CloudFront

/server
  API server
  container image -> ECR
  runtime -> ECS

/cdk
  S3
  CloudFront
  ECR
  ECS
  ALB
  IAM
```

AWS Blocks を素直に使うと、まず `/server -> ECR -> ECS` の部分が薄くなる。Blocks の backend code は基本的に Lambda + API Gateway に載るため、常駐コンテナの API サーバーを前提にしない。

```txt
/client
  Next.js / React frontend

/server or /aws-blocks
  ApiNamespace
  FileBucket
  Auth / KV / Database などの Blocks
  必要なら CDK construct もここに近い場所で追加

/cdk
  薄くなる、または server 側に統合される
  既存 VPC / HostedZone / CloudFront のような周辺だけ残す選択はあり
```

対応表にするとこう。

| これまで | AWS Blocks での扱い |
| --- | --- |
| `/client` | ほぼそのまま。Next.js / React の UI |
| `/server` | `ApiNamespace` などの Blocks backend に寄る |
| `/server` の Dockerfile | 多くの場合いらない。Lambda runtime になる |
| ECR | 多くの場合いらない |
| ECS | 多くの場合いらない |
| `/cdk` | Blocks が生成する CDK に吸収される。周辺インフラだけ残すか、同じ CDK app に寄せる |
| S3 バケット定義 | `FileBucket` が持つ。細かい設定が必要なら CDK に降りる |
| CloudFront + S3 frontend hosting | Blocks の `Hosting` を使うか、既存 CDK で残す |

つまり、AWS Blocks を採用すると次のような変化になる。

1. `/server` は「コンテナ化する API サーバー」ではなく「Blocks backend」になる
2. `/cdk` は独立した全インフラ管理アプリではなく、Blocks backend の synth 結果または拡張層になる
3. ECS/ECR はデフォルトの選択肢ではなくなる
4. 既存の ECS 前提の設計を残したいなら、AWS Blocks は部分採用にしたほうがよい

なので今回の S3 署名付き URL サンプルなら、AWS Blocks 版はこう始めるのが自然。

```txt
/client
  upload UI

/server
  aws-blocks/index.ts
  FileBucket
  ApiNamespace.createUploadUrl()

なし、または薄い /cdk
  追加の CDK construct
  既存ドメインや CloudFront など、Blocks 外の周辺だけ
```

逆に、既存の `/server -> ECR -> ECS` を維持したいなら、AWS Blocks のうまみはかなり減る。その場合は AWS SDK + CDK 直書きで、LocalStack / MinIO / 実 AWS を切り替える従来構成のほうが素直。

## 記事下書き

比較込みの下書きは次に置いている。

- `articles/aws-blocks-s3-presigned-url-local.md`

## 参考

- AWS Blocks Developer Guide: `https://docs.aws.amazon.com/blocks/latest/devguide/what-is-blocks.html`
- AWS Blocks Data storage: `https://docs.aws.amazon.com/blocks/latest/devguide/bb-data-storage.html`
