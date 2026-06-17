# AWS Blocks FileBucket Demo SPEC

## これはなに

AWS Blocks の `FileBucket` を使って、S3 の署名付き URL 的なファイルアップロード体験をローカルで動かすデモアプリを作る。

目的は「S3 の署名付き URL を AWS SDK + CDK で直接実装した場合」と比べて、AWS Blocks を使うと何が隠れて、何が楽になって、逆にどこが見えづらくなるのかを確認すること。

完成物は GitHub の新規リポジトリに置く。

想定リポジトリ名:

```txt
aws-blocks-filebucket-presigned-demo
```

## ゴール

- Next.js ベースの AWS Blocks サンプルアプリを作る
- `FileBucket` でファイルアップロード/ダウンロードを実装する
- ローカルで AWS アカウントなしにアップロード/ダウンロードできることを確認する
- `.bb-data/` にローカル保存される様子を確認できるようにする
- AWS Blocks を使った場合と SDK 直書きの場合の差分を `docs/findings.md` に残す
- Zenn 記事に使えるスクショ/実行ログ/メモを残す

## 非ゴール

- 認証機能は入れない
- 本番利用できるファイル管理アプリにはしない
- 複雑な権限管理、マルチテナント、ウイルススキャンは扱わない
- LocalStack / MinIO 版は実装しない
- SDK 直書き版の完全な比較アプリは作らない

## 技術スタック

- AWS Blocks
- Next.js
- React
- TypeScript
- `FileBucket`
- `ApiNamespace`

Node.js は AWS Blocks 側の README に合わせること。
現時点では Node.js 22 以上、npm 10 以上を想定する。

## 初期セットアップ

まず AWS Blocks の Next.js テンプレートから作る。

```sh
npm create @aws-blocks/blocks-app@latest aws-blocks-filebucket-presigned-demo -- --template nextjs
cd aws-blocks-filebucket-presigned-demo
npm install
npm run dev
```

もしコマンドやテンプレート名が変わっていたら、以下を確認して最新の手順に合わせる。

- https://aws.amazon.com/jp/products/developer-tools/blocks/
- https://github.com/aws-devtools-labs/aws-blocks
- https://docs.aws.amazon.com/blocks/latest/devguide/what-is-blocks.html

## アプリ仕様

### 画面

トップページだけでよい。

画面には次の要素を置く。

- ファイル選択 input
- アップロードボタン
- アップロード状況表示
- アップロード済みファイル一覧
- 各ファイルのダウンロード/表示ボタン
- 各ファイルの削除ボタン
- ローカル保存場所に関するメモ表示

見た目は凝らなくてよい。
ただし、デモで見せるために次の状態は分かるようにする。

- 未選択
- アップロード中
- アップロード成功
- アップロード失敗
- 一覧読み込み中
- 削除成功

### ファイル一覧

一覧には最低限これを表示する。

- object path
- file name
- content type
- size
- last modified または uploaded at 相当

`FileBucket.scan()` で取れる情報に合わせてよい。
足りない情報がある場合は無理に別ストアを作らず、分かる範囲で表示する。

## Backend API

`aws-blocks/index.ts` など、テンプレートの推奨構成に合わせて実装する。

最低限、次の API を作る。

```ts
createUploadUrl(input: {
  filename: string;
  contentType: string;
}): Promise<{
  path: string;
  uploadUrl: string;
}>
```

```ts
createDownloadUrl(input: {
  path: string;
}): Promise<{
  downloadUrl: string;
}>
```

```ts
listFiles(): Promise<{
  files: Array<{
    path: string;
    size?: number;
    contentType?: string;
    lastModified?: string;
  }>;
}>
```

```ts
deleteFile(input: {
  path: string;
}): Promise<void>
```

### FileBucket 定義

最初は最小でよい。

```ts
const bucket = new FileBucket(scope, "uploads", {
  removalPolicy: "destroy",
});
```

`corsRules` や `versioned` は、まず最小実装が動いてから検証する。
ローカルでは CORS / lifecycle が効かない可能性があるので、README の記述と実動作を `docs/findings.md` に残す。

### path の付け方

アップロード path は次の形式にする。

```txt
uploads/{yyyyMMdd}/{uuid}-{safeFileName}
```

要件:

- `filename` は path traversal できないように `/` や `..` を除去する
- 日本語ファイル名はできれば保持する
- 難しければ `encodeURIComponent` する
- UUID を付けて上書きを避ける

## Frontend Flow

### アップロード

1. ユーザーがファイルを選ぶ
2. `api.createUploadUrl({ filename, contentType })` を呼ぶ
3. 返ってきた `uploadUrl` に browser fetch で `PUT` する
4. 成功したら一覧を再読み込みする

```ts
await fetch(uploadUrl, {
  method: "PUT",
  headers: {
    "content-type": file.type || "application/octet-stream",
  },
  body: file,
});
```

### ダウンロード/表示

1. 一覧のファイルを選ぶ
2. `api.createDownloadUrl({ path })` を呼ぶ
3. 返ってきた `downloadUrl` を新しいタブで開く

### 削除

1. 一覧の削除ボタンを押す
2. `api.deleteFile({ path })` を呼ぶ
3. 成功したら一覧を再読み込みする

## 確認したいこと

実装が終わったら、以下を確認して `docs/findings.md` に書く。

### ローカル実装

- `.bb-data/` が作成されるか
- アップロードしたファイルが `.bb-data/` 配下に保存されるか
- メタデータや content body がどのような構造で保存されるか
- dev server を再起動してもファイルが残るか
- `.bb-data/` を削除すると状態が消えるか

### 署名付き URL 相当の挙動

- `putUrl()` が返す URL の形
- `getUrl()` が返す URL の形
- URL に token らしき query が付くか
- URL の有効期限を短くしたときに失効するか
- `contentType` と `PUT` 時の header がずれたときにどうなるか

### SDK 直書きとの差分

次の観点でメモする。

| 観点 | AWS Blocks FileBucket | SDK + CDK 直書き |
| --- | --- | --- |
| ローカル実行 | 何が不要になったか | 何を用意する必要があるか |
| コード量 | どこが短くなったか | どこを自分で書くか |
| デバッグ | 何が見やすいか | 何が見えづらいか |
| CORS | ローカルと AWS でどう違うか | CDK でどう管理するか |
| CDK | 生成されるものを見る | 自分で書く |

## CDK / deploy 確認

可能なら AWS deploy まではしなくてよい。
ただし、少なくとも CDK synth 相当でどのリソースが生成されるかは確認したい。

テンプレートの package script を確認し、該当するコマンドを実行する。

```sh
npm run
```

`synth` や `cdk synth` 相当のコマンドがあれば実行する。
生成された template や `cdk.out` は `docs/findings.md` に要点だけ書く。

確認したいリソース:

- S3 bucket
- Lambda
- API Gateway
- IAM role / policy
- CORS 設定

AWS アカウントに deploy する場合は別途確認してから行う。

## ディレクトリ構成

目安。
テンプレートの構成と違う場合は、テンプレートを優先する。

```txt
.
├── app/
│   └── page.tsx
├── aws-blocks/
│   └── index.ts
├── docs/
│   ├── findings.md
│   └── screenshots/
├── package.json
└── README.md
```

## README に書くこと

README には最低限これを書く。

- これは AWS Blocks `FileBucket` のデモであること
- セットアップ手順
- 起動手順
- アップロード/ダウンロードの試し方
- `.bb-data/` の消し方
- `docs/findings.md` へのリンク

例:

```sh
npm install
npm run dev
```

ローカル状態を消す:

```sh
rm -rf .bb-data
```

## 完了条件

以下を満たしたら完了。

- `npm install` が通る
- `npm run dev` でアプリが起動する
- ブラウザからファイルをアップロードできる
- アップロードしたファイルを一覧表示できる
- 署名付き URL 相当の URL でファイルを開ける
- ファイルを削除できる
- `.bb-data/` にローカル保存されていることを確認できる
- `docs/findings.md` に Blocks と SDK 直書きの差分メモがある
- README に起動手順がある

## デモで見せたい流れ

1. `npm run dev`
2. ブラウザでトップページを開く
3. 画像またはテキストファイルをアップロードする
4. 一覧に出る
5. ダウンロード/表示 URL を開く
6. `.bb-data/` 配下に保存されていることを見せる
7. `docs/findings.md` で SDK 直書きとの差分を見る

## 注意点

- AWS Blocks は新しいため、API やテンプレート構成が変わっている可能性がある
- 実装中に docs / README と異なる挙動があれば、コードを無理に合わせるより `docs/findings.md` に差分として残す
- この記事用の検証なので、動いた事実だけでなく「どこで迷ったか」「何が見えづらいか」も残す
