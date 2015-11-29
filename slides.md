## 社内itamaeプラグインとか作ってる話
sue445

2015/12/09 Itamae Meetup #1

---
## 自己紹介
[![sue445](img/sue445.png)](https://twitter.com/sue445)

* [sue445](https://twitter.com/sue445)
  * Ruby歴3年半、golang歴3ヶ月、Go歴33年(33歳)
* [株式会社ドリコム](http://www.drecom.co.jp/) 所属
* サーバサイド全般の雑用
  * インフラ、アプリ、ライブラリ、社内ツールetc
* 月1〜2個くらいgemを作ってる
* Rubyでプリキュアを作ったことで有名 ([rubicure](http://sue445.hatenablog.com/entry/2013/12/16/000011)) [![hatebu](http://b.hatena.ne.jp/entry/image/large/http://sue445.hatenablog.com/entry/2013/12/16/000011)](http://b.hatena.ne.jp/entry/http://sue445.hatenablog.com/entry/2013/12/16/000011)
* 最近golangでGo! プリンセスプリキュアを作った ([GoPrecure](http://sue445.hatenablog.com/entry/2015/12/07/000000))  [![hatebu](http://b.hatena.ne.jp/entry/image/large/http://sue445.hatenablog.com/entry/2015/12/07/000000)](http://b.hatena.ne.jp/entry/http://sue445.hatenablog.com/entry/2015/12/07/000000)【NEW!】

---
## 【今期の嫁】キュアトゥインクル
![cure_twinkle](img/cure_twinkle.png)

---
## 【本妻】キュアピース
![cure_peace](img/cure_peace.png)

---
## Agenda
だいたい作ったgemの紹介するだけです

* ドリコムプロビジョニング勢力図
* 俺のitamaeの実行方法
* itamaeモンキーパッチgem系
  * `specinfra-plain_sudo`（社内gem）
  * `drecom-itamae`（社内gem）
* その他便利なプラグイン
  * `itamae-plugin-recipe-drecom_rails`（社内gem）
  * `itamae-plugin-resource-encrypted_remote_file` (OSS)

---
## ドリコムプロビジョニング勢力図
* ほとんどがChef
* 一部Ansible
* 個人レベルでitamae (今日はこの話)

---

## 俺のitamaeの実行方法
Ralefileでhosts.ymlを読み込んでrake taskを動的生成

### hosts.yml
```yml
staging:
  - hostname:   myapp-staging-01
    ip_address: xxx.xxx.xxx.xxx
    role_suite: staging
production:
  - hostname:   myapp-01
    ip_address: xxx.xxx.xxx.xxx
    role_suite: production_master
  - hostname:   myapp-02
    ip_address: xxx.xxx.xxx.xxx
    role_suite: production_slave
```

---
### Usage
```sh
rake recipe:production                       # Run itamae recipes to [production] all
rake recipe:production:myapp-01              # Run itamae recipes to [production] myapp-01 (xxx.xxx.xxx.xxx)
rake recipe:production:myapp-02              # Run itamae recipes to [production] myapp-02 (xxx.xxx.xxx.xxx)
rake recipe:staging                          # Run itamae recipes to [staging] all
rake recipe:staging:myapp-staging-01         # Run itamae recipes to [staging] myapp-staging-01 (xxx.xxx.xxx.xxx)
rake spec:production                         # Run serverspec tests to [production] all
rake spec:production:myapp-01                # Run serverspec tests to [production] myapp-01 (xxx.xxx.xxx.xxx)
rake spec:production:myapp-02                # Run serverspec tests to [production] myapp-02 (xxx.xxx.xxx.xxx)
rake spec:staging                            # Run serverspec tests to [staging] all
rake spec:staging:myapp-staging-01           # Run serverspec tests to [staging] myapp-staging-03 (xxx.xxx.xxx.xxx)
```

* ソース: https://gist.github.com/sue445/39ff664b462ab7fb230b

---
## itamaeモンキーパッチgem系

---
### 会社でitamae使う時に困ったこと
* ssh経由で実行すると `sudo /bin/sh -c <元のコマンド>` がうちのインフラだと一部OSでエラーになる
  * LDAPのユーザを `/etc/sudoers` を紐付けるのが原因だと思うんだけど詳細不明
  * specinfraが生成してる部分なのでitamaeとserverspec両方ダメ :cry:
* 上記を解決しても（後述） `itamae ssh` すると /tmpにディレクトリを作れなくてこける
* ローカルに対して実行する形式だと無問題

```sh
INFO : Starting Itamae...
ERROR : Command mkdir -p /tmp/itamae_tmp failed. (exit status: 1)
ERROR : stdout | ユーザー sueyoshi_go は'/bin/sh -c mkdir -p /tmp/itamae_tmp' を root として sue445-dev-01 上で実行することは許可されていません。すみません。
```
---
### 解決策1: `specinfra-plain_sudo`（社内gem）
* `sudo /bin/sh -c <元のコマンド>` してるところを `sudo <元のコマンド>` に整形し直すgem
* `Specinfra::Backend::DrecomSsh` のように新しいBackendを追加する形式だとitamaeで使えないので `Specinfra::Backend::Ssh` にパッチを当てる方式
* https://gist.github.com/sue445/4b8013ad19a3f5917aee

---
### 解決策2: `drecom-itamae`（社内gem）
* 起動する一瞬だけ `disable_sudo` を有効にするラッパ

---
#### /exe/drecom-itamae

```ruby
#!/usr/bin/env ruby

require 'itamae/cli'
require 'drecom-itamae'

require 'specinfra/helper/set'
include Specinfra::Helper::Set

set :disable_sudo, true    # 【ここだけ追加】

Itamae::CLI.start
```
---

#### itamaeの初期化が終わってrecipe実行する直前にsudo使えるようにする

```ruby
module Itamae
  class Runner
    def initialize_with_enable_sudo(*args)
      initialize_without_enable_sudo(*args)

      # itamaeの初期化が終わってrecipe実行する直前にsudo使えるようにする
      Specinfra.configuration.disable_sudo = false
    end
    alias_method_chain :initialize, :enable_sudo
  end
end
```

---
* 酷いモンキーパッチだけどこれで社内でitamae使えるようになった
* どちらも弊社インフラに起因するものなのでOSSにするつもりはありません :innocent:

---
## その他便利なプラグイン

---
### `itamae-plugin-recipe-drecom_rails`（社内gem）
Railsアプリあるあるミドルのレシピをひとまとめにgemにした

* 3つ以上アプリをitamae化をすると似たようなレシピ増産される弊害
  * 飽きるしDRYじゃない
* ruby, nginx, redis, memcached, nodejs, percona-server(MySQL) あたりをインストール

---
#### 構成
```
lib
└── itamae
    └── plugin
        └── recipe
            ├── drecom_rails
            │   ├── cookbooks
            │   │   ├── files/
            │   │   ├── memcached.rb
            │   │   ├── myhome.rb
            │   │   ├── nginx.rb
            │   │   ├── nodejs.rb
            │   │   ├── percona.rb
            │   │   ├── redis.rb
            │   │   ├── ruby.rb
            │   │   ├── setup.rb
            │   │   └── templates/
            │   ├── roles
            │   │   ├── all.rb # ミドル全部入り
            │   │   ├── cache.rb # memcachedをインストール
            │   │   ├── db.rb # perconaをインストール
            │   │   ├── kvs.rb # redisをインストール
            │   │   └── web.rb # nginx,nodejsをインストール
            │   └── version.rb
            └── drecom_rails.rb
```
* roleの中でさらにcookbookのレシピを `incliue_recipe` している

---
#### 利用イメージ
roles/staging.rb

```ruby
include_recipe "drecom_rails::roles::all"

# アプリで個別に必要なパッケージをインストール
include_recipe "../cookbooks/packages/default.rb"
```

---
#### その他
* レシピだけgemにいれておいて、設定値はnode.ymlから差し込む方式
* ミドル単位でレシピを作ってOSSにするという手もあったけど、社内利用に限定することでシンプルに使えるようにしたかった

---
### `itamae-plugin-resource-encrypted_remote_file` (OSS)
[https://github.com/sue445/itamae-plugin-resource-encrypted\_remote_file](https://github.com/sue445/itamae-plugin-resource-encrypted_remote_file)

* リポジトリには暗号化してコミットしつつ、プロビジョニング時には復号化して転送するためのプラグイン
* Chefでいうところの [knife-solo\_data\_bag](https://github.com/thbishop/knife-solo_data_bag) みたいなやつ
* 詳しいこと: [itamae-plugin-resource-encrypted\_remote_file を作った - くりにっき](http://sue445.hatenablog.com/entry/2015/05/09/185807)
* [itamae-secrets](https://github.com/sorah/itamae-secrets) の方が便利そう :innocent:

---
#### 使い方
```ruby
encrypted_remote_file "/home/deployer/.ssh/id_rsa" do
  owner    "root"
  group    "root"
  source   "remote_files/id_rsa.encrypted"
  password ENV["ID_RSA_PASSWORD"]
end
```

* `remote_file` とだいたい同じ
* 暗号化する時にパスワードをかけられるのが特徴
* [dotenv](https://github.com/bkeepers/dotenv) と併用することを想定してる

---
## まとめ
* レシピ書くのに飽きたらgemにすると捗る
* OSSにしづらいものは社内gemにしておくとよい

---
## 参考ポエム
* [社内gemとOSSのgemのメンテについて - くりにっき](http://sue445.hatenablog.com/entry/2015/12/01/000000) [![hatebu](http://b.hatena.ne.jp/entry/image/large/http://sue445.hatenablog.com/entry/2015/12/01/000000)](http://b.hatena.ne.jp/entry/http://sue445.hatenablog.com/entry/2015/12/01/000000)

<!--
  disable uppercase
  via. http://srz-zumix.blogspot.jp/2014/09/revealjs-markdown.html
-->
<style type="text/css">
    .reveal h1,
    .reveal h2,
    .reveal h3,
    .reveal h4,
    .reveal h5,
    .reveal h6 {
      text-transform: none;
    }
</style>
