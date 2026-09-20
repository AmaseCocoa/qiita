---
id: null

title: Gitのためのユーザー管理ツールを作った
tags:
  - 'Git'
private: false
updated_at: ''

organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに
私のように比較的複数のアカウントを切り替えて使う必要がある人は、`git config`や`gh auth switch`などのコマンドを使用していると思います。

しかし、それでは何らかの問題にぶつかります。例えば、ghの場合はgithub以外でも同じように切り替えが必要になる場合がある、などです。

私も元々は、既存ツールと`credential.namespace`の切り替えで対処していました。ですが、これでは非常に不便だったのでそのためのツールを自作することにしました。

https://github.com/AmaseCocoa/alter

## これは何
alterはgitのユーザー情報や資格情報、GPG鍵をコマンドラインで即座に切り替え、管理することができるCLIツールです。実は自分が欲しかった機能を詰め込んでいます。

### なぜRustを使ったのか
alterでRustを使った理由は主に2つあります。1つ目は、Rustのgit周りのエコシステムが比較的強力だと感じたからです。

Rustには[gitoxide](https://github.com/gitoxidelabs/gitoxide)というプロジェクトがあります。これが作られる過程で、gitに関連するクレートがいくつか作られました。その1つが[gix_config](https://crates.io/crates/gix-config)です。しかし、これだけでは動機として弱く、初期はgitconfigを読み取るライブラリが揃っていてコンパイル型言語なGoを使用しようと考えましたが、後述の理由も相まって結果的にRustになりました。

2つ目は、Rustの知識をある程度つけたかったからです。この時点でもインタプリタ型言語やGoなどの簡単なプログラミング言語はある程度扱えたものの、Rustのような比較的複雑な言語の能力もつけたいと考えていました。

## 設計

### CLI
alterのcliは簡単に扱えることを重視しています。例えば、profileを追加したい場合は`alter new`だけで追加できます。同じように、profileを切り替えたい場合は`alter use <profile> [--global (全体に適用したい場合)]`を実行するだけで切り替えできます。

また、現在優先されているprofileも、`alter current`を実行するだけで簡単に確認できます。
```
$ alter current
Current profile:
  Slug: (slug)
  ID: (id)
  Username: (user)
  Email: (email)
  Signing key: (key)
  Credentials:
    - github.com (logged in)
  Scope: global
```

credentialはgit pushなどの認証が必要な操作でalterが呼び出された場合に自動で登録を開始するので奥まった場所 (`alter cred setup`)にありますが、将来的にalter newを実行するタイミングでも追加できるようにしたいと考えています。

### ユーザー管理
alterは`~/.alter/profiles`以下にファイルベースでユーザーを管理しています。例として、ユーザーのプロフィールのtomlは以下のようになっています。

```toml
[profile]
id = "(uuid)"

[user]
username = "(user.name相当の文字列)"
email = "(user.email相当の文字列)"
signingkey = "(user.signingkey相当の文字列)"

[credentials]
hosts = ["github.com", ...(ユーザーのcredentialがあるホスト)]
```

データベースを使用しないことで実装コストを抑えつつ、ユーザーが必要に応じてファイル自体を編集することがこれでできるようになります。

### Config
alter自体も、configファイルを持っています。これは主にユーザーが独自にOAuthプロバイダーを構成するために使用されています。例えば、`.alter/config.toml`に以下のようなキーを追加すると、該当ホストではその情報を使用してOAuthを構成するようになります。

```toml
[oauth_providers."git.example.com"]
type = "generic"
host = "git.example.com"
client_id = "client-id"
auth_endpoint = "https://git.example.com/oauth2/authorize"
token_endpoint = "https://git.example.com/oauth2/token"
scopes = ["repository"]
```

また、typeが`github`、`gitlab`、`gitea`のいずれかの場合は、`auth_endpoint`と`token_endpoint`が自動的に生成されます。

### OAuth
alterが内蔵しているcredential helperはPKCEによるOAuth認証にのみ対応しています。主に簡略化が目的ですが、将来的に特定のホストに対してだけは別のcredential helperに引き渡しできるようにもしたいと考えています。

#### Keyring
alterが取得したOAuthトークンは、OSのkeyringに安全に保管されます。 (Linuxの場合は主にgnome-keyring、macOSの場合はkeychain、windowsの場合は資格情報マネージャー)

## 開発中の課題
### rustlsとnative-tls
もともとprebuild-binaryでは(なんとなく) rustlsを使っていたのですが、なぜかビルドできなくなる問題に当たったりしました。これは事前ビルドのバイナリではnative-tlsを使うようにしてなんとか解決しました。

## 感想
alterはかなり前に作ったツールだったのですが、当時は思ったよりもGitHub CopilotのClaude Haiku 4.5がコストに対して使えた印象があります。あとは、コーディングエージェントを活用しつつ作ったcredential helperも、現在までは特にバグは起こらずに安定して使い続けられています。
