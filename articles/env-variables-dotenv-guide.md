---
title: "環境変数とは何か？.envファイルの仕組みを理解する"
emoji: "🔐"
type: "tech"
topics: ["env", "dotenv", "nextjs", "vite", "セキュリティ"]
published: false
---

## 0. 超要約（クイックリファレンス）

**結論だけ先に知りたい人向けのまとめです。**

- **環境変数**：
  - OSが提供する「変数名=値」形式のデータ共有の仕組みです。
  - アプリケーションの設定や秘密情報をコードの外に分離できます。

- **.envファイル**：
  - プロジェクトごとに環境変数を定義するテキストファイルです。
  - dotenvライブラリ（またはNode.js v20.6.0+のネイティブ機能）で読み込みます。

- **最も大切なこと**：
  - `.env` は `.gitignore` に必ず追加し、Gitにコミットしてはいけません。
  - `NEXT_PUBLIC_` や `VITE_` 付きの変数にAPIシークレットなどの機密情報を入れてはいけません。

👉 **迷った場合の判断基準**

:::message
ブラウザに見せてよい値 → `NEXT_PUBLIC_` / `VITE_` 付き
秘密にすべき値 → プレフィックスなし（サーバー側でのみ使用）
:::

---

## 1. 環境変数とは

環境変数は、OS上で動作するプロセスに対して**外部からデータを渡す仕組み**です。

「変数名=値」の形式で、アプリケーションの設定をコードの外に分離できます。

### 身近な例：PATH

開発者にとって最も身近な環境変数は `PATH` です。ターミナルで `node` や `git` と入力したとき、OSはこの `PATH` を見て「どこにある実行ファイルか」を探しています。

```bash
# Linuxでの確認
echo $PATH

# Windowsでの確認（PowerShell）
$env:PATH
```

### 設定方法

```bash
# Linux / macOS
export API_KEY=my-secret-key

# Windows（PowerShell）
$env:API_KEY = "my-secret-key"
```

### プロセスとの関係

環境変数は**プロセス単位**で保持されます。親プロセスの環境変数は子プロセスに継承されますが、子プロセスで変更しても親には影響しません。

```text
親プロセス（ターミナル）
│  API_KEY=xxx
│
├── 子プロセス（node app.js）
│     API_KEY=xxx  ← 親から継承される
│
└── 子プロセス（python main.py）
      API_KEY=xxx  ← 同様に継承される
```

---

## 2. なぜWeb開発で環境変数が必要なのか

### 理由1：秘密情報をコードに書きたくない

以下のコードを見てください。

```javascript
// ❌ やってはいけない例
const client = new S3Client({
  credentials: {
    accessKeyId: "AKIAIOSFODNN7EXAMPLE",
    secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  },
});
```

このコードをGitHubにpushした瞬間、世界中の誰でもAWSの認証情報を閲覧できてしまいます。

:::message alert
GitGuardianの調査（2025年）によると、GitHubの公開リポジトリで**約2,900万件**の秘密情報漏洩が検出されています。さらに、漏洩した秘密情報の**70%が2年後もまだ有効**（失効させていない）であることがわかっています。
:::

環境変数を使えば、コードと秘密情報を分離できます。

```javascript
// ✅ 環境変数を使う
const client = new S3Client({
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  },
});
```

### 理由2：環境ごとに設定を切り替えたい

開発・ステージング・本番で、接続先のDBやAPIのURLが異なるのは一般的です。

```text
開発環境:   DATABASE_URL=postgresql://localhost:5432/mydb_dev
本番環境:   DATABASE_URL=postgresql://prod-db.example.com:5432/mydb
```

環境変数を使えば、**コードを一切変更せずに**環境ごとの設定を切り替えられます。

---

## 3. .envファイルの仕組み

### 毎回 export するのは面倒

環境変数をターミナルで手動設定するのは手間がかかります。プロジェクトで使う環境変数が10個あったら、毎回10行の `export` を打つことになります。

そこで登場するのが **`.env`ファイル** です。

### .envファイルとは

プロジェクトルートに置くテキストファイルで、環境変数を一行ずつ定義します。

```text
# .env
DATABASE_URL=postgresql://localhost:5432/mydb
API_KEY=my-secret-api-key
PORT=3000
```

### dotenvライブラリの役割

`.env`ファイルはただのテキストファイルです。OSが自動的に読み込んでくれるわけではありません。

**dotenvライブラリ**がアプリ起動時にこのファイルを読み込み、`process.env` に値を注入してくれます。

```bash
npm install dotenv
```

```javascript
// app.js
require('dotenv').config();

// これで .env の値が使える
console.log(process.env.DATABASE_URL);
// → "postgresql://localhost:5432/mydb"
```

処理の流れを整理すると以下のようになります。

```text
.envファイル（テキスト）
    │
    │ dotenvが読み込み
    ▼
process.env に値を注入
    │
    │ アプリケーションが参照
    ▼
process.env.API_KEY → "my-secret-api-key"
```

### Node.js v20.6.0+ のネイティブサポート

Node.js v20.6.0 以降では、dotenvライブラリなしで `.env` ファイルを読み込めるようになりました。

```bash
node --env-file=.env app.js
```

ただし、複数の `.env` ファイルの切り替えなど、高度な用途では引き続きdotenvライブラリが便利です。

---

## 4. .envファイルをGitにコミットしてはいけない理由

:::message alert
**これは最も重要なルールです。** `.env` ファイルには秘密情報が含まれるため、絶対にGitにコミットしてはいけません。
:::

### なぜ危険なのか

1. **公開リポジトリ** → pushした瞬間、世界中から閲覧可能になる
2. **プライベートリポジトリ** → チームメンバー全員に秘密情報が共有される（退職者のアクセス管理も問題になる）
3. **Git履歴に残る** → ファイルを削除しても、`git log` からいつでも復元できる

### 一度コミットしたらどうなるか

```text
git add .env       ← コミットしてしまった
git rm .env        ← ファイルを削除
git commit         ← 削除をコミット

しかし...
git log -p -- .env ← 過去の履歴から内容が見える！
```

**一度コミットした秘密情報は「漏洩した」と考えるべき**です。ファイルを削除するだけでは不十分で、APIキーやパスワードを即座に無効化して再発行する必要があります。

### 対策：.gitignore に追加する

プロジェクト作成時に必ず `.gitignore` に追加しましょう。

```text
# .gitignore
.env
.env.local
.env*.local
```

---

## 5. .envファイルの種類と使い分け

プロジェクトでは複数の `.env` ファイルを使い分けることがあります。

| ファイル | Gitに含める | 用途 |
|---|---|---|
| `.env` | 状況による※ | 全環境共通のデフォルト値 |
| `.env.local` | **含めない** | ローカル固有の秘密情報 |
| `.env.development` | 含める | 開発環境のデフォルト値 |
| `.env.production` | 含める | 本番環境のデフォルト値 |
| `.env.example` | **含める** | 必要な変数のキーだけを列挙したテンプレート |

※ `.env` に秘密情報を含まない場合のみGitに含めてよい

### .env.example の重要性

`.env` は `.gitignore` で管理対象外になるため、チームメンバーがリポジトリをcloneした直後に「どの環境変数を設定すればいいかわからない」問題が起きます。

`.env.example` は値を空にした（またはダミー値を入れた）テンプレートです。

```text
# .env.example（Gitにコミットする）
DATABASE_URL=
API_KEY=your-api-key-here
PORT=3000
```

これをGitに含めておけば、新しいメンバーはこれをコピーして値を埋めるだけで開発を始められます。

```bash
cp .env.example .env
# → .env を編集して実際の値を入力
```

---

## 6. フレームワークでの環境変数の扱い

### 共通の大原則

モダンなフレームワークでは、**ブラウザに公開してよい変数かどうか**を特定のプレフィックスで区別します。

```text
秘密情報の扱い
├─ プレフィックスなし
│   └─ サーバー側でのみアクセス可能（安全）
│
└─ NEXT_PUBLIC_ / VITE_
    └─ ビルド時にJavaScriptバンドルに埋め込まれる（ブラウザから見える）
```

### Next.js の場合

```text
# ブラウザに公開される（ビルド時にバンドルに埋め込まれる）
NEXT_PUBLIC_API_URL=https://api.example.com

# サーバーでのみ利用可能
DATABASE_URL=postgresql://...
SECRET_KEY=xxx
```

**読み込み優先順位（上が最優先）：**

1. `process.env`（OS側で設定済みの値）
2. `.env.$(NODE_ENV).local`
3. `.env.local`（`NODE_ENV=test` の場合は読み込まれない）
4. `.env.$(NODE_ENV)`
5. `.env`

### Vite の場合

Viteでは `process.env` ではなく **`import.meta.env`** を使います。

```javascript
// Viteでの環境変数アクセス
console.log(import.meta.env.VITE_API_URL);
```

**読み込み優先順位（上が最優先）：**

1. `.env.[mode].local`
2. `.env.[mode]`
3. `.env.local`
4. `.env`

### プレフィックス付き変数の注意点

:::message alert
`NEXT_PUBLIC_` や `VITE_` 付きの変数は、ビルド時にJavaScriptバンドルにハードコードされます。つまり、ブラウザの開発者ツールから値を確認できます。**APIシークレットやDBパスワードには絶対に使わないでください。**
:::

```text
# ✅ ブラウザに見せてよい情報
NEXT_PUBLIC_APP_NAME=MyApp
NEXT_PUBLIC_GA_ID=G-XXXXXXXXXX

# ❌ プレフィックスを付けてはいけない情報
NEXT_PUBLIC_DATABASE_URL=...     ← 絶対にダメ
NEXT_PUBLIC_SECRET_KEY=...       ← 絶対にダメ
```

---

## 7. 本番環境での環境変数の設定方法

本番環境では `.env` ファイルを直接サーバーに配置するのではなく、ホスティングサービスの機能を使うのが一般的です。

### Vercel

ダッシュボードの「Settings > Environment Variables」から設定します。Production / Preview / Development の各環境ごとに異なる値を設定でき、保存時に暗号化されます。

### AWS

| サービス | 用途 | 特徴 |
|---|---|---|
| Secrets Manager | APIキー、DBパスワード | 自動ローテーション対応 |
| Parameter Store | 設定値の管理 | Secrets Managerより安価 |

### Docker

```yaml
# docker-compose.yml
services:
  app:
    image: my-app
    environment:
      - DATABASE_URL=postgresql://...
    # または
    env_file:
      - .env
```

### GitHub Actions

リポジトリの「Settings > Secrets and variables」から設定します。ワークフロー内で `${{ secrets.API_KEY }}` のように参照できます。

---

## 8. よくある間違い Q&A

### Q1. `.env` をGitにコミットしてしまいました。どうすればいいですか？

ファイルを削除してコミットするだけでは**不十分**です。Git履歴に値が残っています。

1. 漏洩した秘密情報をすべて**即座に無効化**して再発行する
2. `.gitignore` に `.env` を追加する
3. 必要に応じて `git filter-branch` やBFG Repo-Cleanerで履歴から削除する

**最も重要なのは1（秘密情報の無効化）です。**

---

### Q2. `.env` を変更したのに反映されません。

開発サーバーの**再起動が必要**です。Next.jsやViteでは、`.env` の変更はホットリロードの対象外です。

---

### Q3. `NEXT_PUBLIC_` をビルド後に変更したのに反映されません。

`NEXT_PUBLIC_` 付きの変数は **ビルド時に値が固定**されます。変更後は再ビルドが必要です。

```bash
# 変更後は再ビルドが必要
npm run build
```

---

### Q4. 環境変数の値が `undefined` になります。

よくある原因は以下の通りです。

- dotenvの読み込み（`require('dotenv').config()`）がされていない
- `.env` のファイル名にタイプミスがある（`.env.` のように余計なドットがある等）
- `NEXT_PUBLIC_` や `VITE_` のプレフィックスが付いていない（クライアント側で参照している場合）
- 動的に参照している（`process.env[key]` のような書き方はインライン化されない）

---

### Q5. `.env` の値は全部文字列になりますか？

はい。`.env` で定義した値はすべて**文字列**として読み込まれます。

```text
PORT=3000
DEBUG=true
```

```javascript
typeof process.env.PORT   // → "string"（"3000"）
typeof process.env.DEBUG  // → "string"（"true"）

// 数値として使う場合は変換が必要
const port = parseInt(process.env.PORT, 10);
```

---

## 9. まとめ

- 環境変数は、アプリの設定や秘密情報をコードの外に分離するOS標準の仕組みです。
- `.env` ファイルは、プロジェクト単位で環境変数を管理するための便利なテキストファイルです。
- `.env` は `.gitignore` に必ず追加し、Gitにコミットしてはいけません。
- `NEXT_PUBLIC_` や `VITE_` 付きの変数はブラウザから見えるため、機密情報を設定してはいけません。
- 本番環境では `.env` ファイルではなく、ホスティングサービスの環境変数管理機能を使いましょう。
- チーム開発では `.env.example` を用意して、必要な変数を明示しましょう。

:::message
**一言でまとめると：** 「コードに秘密情報を書かない。`.env` はGitに入れない。ブラウザに渡す変数と渡さない変数を区別する。」この3点を守れば、環境変数に関するトラブルの大半は防げます。
:::

## 参考

- [Next.js公式: Environment Variables](https://nextjs.org/docs/pages/guides/environment-variables)
- [Vite公式: Env Variables and Modes](https://vite.dev/guide/env-and-mode)
- [dotenv - npm](https://www.npmjs.com/package/dotenv)
- [GitGuardian: State of Secrets Sprawl 2025](https://www.gitguardian.com/state-of-secrets-sprawl-report-2025)
