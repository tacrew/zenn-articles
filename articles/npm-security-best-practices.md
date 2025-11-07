---
title: "npmのサプライチェーン攻撃に備えてやったこと【サンプル付き】"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["npm", "pnpm", "security", "package", "nodejs"]
published: false
publication_name: omakase
---

npm まわりでサプライチェーン攻撃が頻発していますね。直近でいうと以下の 3 件が立て続けに発生しています。

- 2025-08-26 — [Nx（nrwl/nx）関連「S1ngularity」](https://github.com/nrwl/nx/security/advisories/GHSA-cxm3-wv7p-598c)
- 2025-09-08 — [Qix（chalk / debug など 18 パッケージ）](https://nvd.nist.gov/vuln/detail/CVE-2025-59144)
- 2025-09-15 — [自己伝播ワーム「Shai-Hulud」](https://www.sysdig.com/blog/shai-hulud-the-novel-self-replicating-worm-infecting-hundreds-of-npm-packages)

これらの脅威を受けて、パッケージを利用する側が対策できることも数多く共有されています。

- [サプライチェーン攻撃への防御策](https://blog.jxck.io/entries/2025-09-20/mitigate-risk-of-oss-dependencies.html)
- [npmパッケージ/GitHub Actionsを利用する側/公開する側でサプライチェーン攻撃を防ぐためにやることメモ](https://zenn.dev/azu/articles/ad168118524135)

また、npm パッケージ管理ツールの pnpm では [v10.16.0](https://pnpm.io/ja/blog/releases/10.16) にて新バージョン公開から指定時間の間インストールから除外する `minimumReleaseAge` 設定が導入されるなど、業界全体でセキュリティ意識がより一層強くなっているのを感じます。

サプライチェーン攻撃対策を大別すると、感染の前後の 2 段階で分けられます。

1. 予防：攻撃を受けたパッケージのインストールをさけて感染しないようにするなど
2. 隔離：秘匿情報を平文で local に格納しない、開発環境をコンテナに分離するなど

本記事では前者の**予防**に焦点を当て、npm パッケージのサプライチェーン攻撃を受けにくい環境の構築例を共有します。

## npmサプライチェーン攻撃予防策

どれだけ予防策を重ねても攻撃を完全に防げませんし、過剰な対策は運用負荷になります。過去の攻撃内容を振り返ると、被害の主因は以下の 2 つに集約されます。

1. 攻撃された最新バージョンをインストールしてしまった
2. インストール時に `postinstall` 等のスクリプトが実行された

これに対して効果が高い対策は次の通りです。

- 1.への対策
  - バージョンを固定する
  - リリース直後の最新バージョンの導入を避ける
- 2.への対策
  - `postinstall` 等のスクリプトは必要なものだけ明示的に実行する

以上を踏まえ、本記事では「スクリプトを default で許可しない」かつ「リリース直後のバージョンの待機時間設定機能」を備える pnpm を活用する仕組みを共有します。
次章ではこの仕組みを構築する手順を解説していきます。なお同じ手順で構築したサンプルを以下のリポジトリで公開していますので参考になれば幸いです。
[サンプルリポジトリ](https://github.com/tacrew/npm-package-lock-sample)

## 予防環境の構築ステップ

### 1. pnpmの導入と利用の強制

このステップでは pnpm 自体をバージョン管理しつつ導入する手順を紹介します。

#### pnpmの導入

pnpm をインストールしていきます。今回、pnpm のバージョン管理は Node 付属の corepack で行います。

まずは Node をインストールします。バージョン管理には [Volta](https://volta.sh/) を採用していますが、mise などその他の管理ツールをお好みに合わせて利用ください。

```bash
volta pin node@22
```

次に corepack をインストールして pnpm を有効化します。

```bash
volta install corepack
corepack enable
corepack use pnpm@latest-10
```

バージョンを確認すると、packageManager で指定したバージョンと一致します。

```bash
$ pnpm -v
10.20.0

# v10.20.0で一致
$ cat package.json | grep packageManager
  "packageManager": "pnpm@10.20.0+sha512.cf9998222162dd85864d0a8102e7892e7ba4ceadebbf5a31f9c2fce48dfce317a9c53b9f6464d1ef9042cba2e02ae02a9f7c143a2b438cd93c91840f0192b9dd"
```

#### pnpmの強制

せっかく pnpm を導入したのに手癖で npm/npx を使ってしまったら意味がありません。alias で npm/npx を使えないようにしてもよいですが、今回は確認を挟むようにしました。やむを得ず script 等で npm/npx を使う場合は `USE_NPM_ANYWAY=1` で通ります。
zsh を利用している場合は.zshrc に以下を追加します。

```bash
npm() {
  # 非対話 or 明示許可なら確認なしで通す
  if [ -n "${USE_NPM_ANYWAY:-}" ] || [ ! -t 0 ]; then
    command npm "$@"
    return
  fi

  printf "⚠️ pnpm を推奨しています。\n"
  printf "本当に npm を実行しますか？ [y/N] "
  IFS= read -r ans || { echo; return 1; }

  case "$ans" in
    y|Y|yes|YES) command npm "$@";;
    *) echo "中止しました。"; return 1;;
  esac
}

npx() {
  # 非対話 or 明示許可なら確認なしで通す
  if [ -n "${USE_NPM_ANYWAY:-}" ] || [ ! -t 0 ]; then
    command npx "$@"
    return
  fi

  printf "⚠️ pnpm dlx を推奨します。\n"
  printf "本当に npx を実行しますか？ [y/N] "
  IFS= read -r ans || { echo; return 1; }

  case "$ans" in
    y|Y|yes|YES) command npx "$@";;
    *) echo "中止しました。代替例: pnpm dlx $*"; return 1;;
  esac
}
```

#### （おまけ）GitHubActions上でのpnpmのセットアップ

GitHubActions 上での pnpm のセットアップでも package.json の packageManager フィールドを利用したバージョン管理を活用します。
以下のように corepack を有効化した後、actions/setup-node で pnpm をセットアップします。

```yml
steps:
  - name: Checkout repository
    uses: actions/checkout@08c6903cd8c0fde910a37f88322edcfb5dd907a8

  - name: Enable corepack
    run: corepack enable

  - name: Setup Node.js
    uses: actions/setup-node@a0853c24544627f65ddf259abe73b1d18a591444
    with:
      node-version: 22.20.0
      cache: pnpm

  - name: Install dependencies
    run: pnpm install --frozen-lockfile
```

[pnpm/action-setup](https://github.com/pnpm/action-setup) を利用したセットアップが有名ですが、上記内容の方が依存も減りシンプルです。

### 2. パッケージ追加時のバージョン固定の強制

このステップでは npm パッケージをバージョン固定して追加・維持する仕組み作りを紹介します。

#### `.npmrc` の導入

以下の内容の `.npmrc` を project root に配置してパッケージ追加時にバージョンを固定させます。

```text
save-exact=true
```

#### バージョン固定checkのLinter導入

[npm-package-json-lint](https://github.com/tclindner/npm-package-json-lint) を利用して package.json 記載の各パッケージのバージョンが固定されているか確認できるようにします。

```bash
pnpm add -D npm-package-json-lint
```

以下の内容の設定ファイル `.npmpackagejsonlintrc.json` を作成します。

```json
{
  "rules": {
    "no-caret-version-dependencies": "error",
    "no-tilde-version-dependencies": "error",
    "prefer-absolute-version-dependencies": "error",
    "prefer-absolute-version-devDependencies": "error"
  }
}
```

package.json の scripts に以下のようなコマンドを追加します。

```json
{
  "scripts": {
    "lint:package-json": "pnpm exec npmPkgJsonLint ."
  }
}
```

これでバージョンが固定されていない状態で lint を行うと以下のように検知できます。CI workflow にこの lint 処理を組み込んでおくと良いでしょう。

```bash
$ pnpm lint:package-json

./package.json
✖ no-caret-version-dependencies - node: dependencies - You are using an invalid version range. Please do not use ^. Invalid dependencies include: aws-cdk
✖ prefer-absolute-version-dependencies - node: dependencies - You are using an invalid version range. Please use absolute versions. Invalid dependencies include: aws-cdk
2 errors
0 warnings
 ELIFECYCLE  Command failed with exit code 2.
```

### 3. 最新バージョンの待機時間設定

このステップでは予防のメインとなる新規バージョンの公開から一定時間が経過するまで導入を待つ設定をパッケージの新規追加・更新の両ケースに導入していきます。

#### パッケージの新規追加の場合

pnpm の [minimumReleaseAge](https://pnpm.io/ja/settings#minimumreleaseage) によってリリース間もない最新バージョンのインストールを回避します。
指定する期間は pnpm のリリースブログなどでも記載されているとおり、大半の攻撃において発覚から当該バージョンの削除等まで数時間程度であることを踏まえて 3 日間程度が良い印象です。逆に長い時間を指定しても security patch の適用のないバージョンを追加する恐れがあります。
今回は以下の内容で pnpm-workspace.yaml を追加します。

```yaml
minimumReleaseAge: 4320 # 3 days
```

#### パッケージのバージョン更新の場合

renovate の [minimumReleaseAge](https://docs.renovatebot.com/configuration-options/#minimumreleaseage)（旧項目名 `stabilityDays`）によってリリース間もない最新バージョンへの更新を待機します。
待機期間に関しては pnpm の `minimumReleaseAge` で指定した期間と同一で良いでしょう。
以下内容の renovate.json を追加します。かなり簡素な例なので実運用の際はチーム状況に応じてその他設定を追加してください。

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "minimumReleaseAge": "3 days",
  "prConcurrentLimit": 1
}
```

## おわりに

長々と手順を解説してきましたが、設定自体は数も少なくシンプルです。記事を見ながら設定をしたら最後に[サンプルリポジトリ](https://github.com/tacrew/npm-package-lock-sample)を参照して、抜け漏れがないか確認してみてください。

また、CI を利用するほどでもないリポジトリの場合は husky 等を利用して precommit 時に `lint:package` を走らせるなどのアレンジも可能です。例えば zenn の記事を管理する[弊社テンプレ](https://github.com/0xmakase/zenn-articles)ではこのアレンジを採用しています。

リポジトリ内容や開発運用方針に合わせて本記事の内容を部分的にも採用することで、サプライチェーン攻撃の予防に繋がれば幸いです。

## 参考記事

- [サプライチェーン攻撃への防御策](https://blog.jxck.io/entries/2025-09-20/mitigate-risk-of-oss-dependencies.html)
- [npmパッケージ/GitHub Actionsを利用する側/公開する側でサプライチェーン攻撃を防ぐためにやることメモ](https://zenn.dev/azu/articles/ad168118524135)
- [How to integrate Corepack and Volta](https://www.yanglinzhao.com/til/integrating-corepack-and-volta)
