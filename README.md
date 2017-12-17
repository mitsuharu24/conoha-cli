# Overview: 概要紹介

## これは何？

ConoHa で簡単な勉強用VPSを構築したい人向けのお手軽CLIです。  
Go言語製で様々なOSに対応(予定)です。

このプラグインは非公式ですので、ConoHaのサポートなどへの問い合わせはご遠慮ください。

## なんで作ったの？

ConoHa CLI は既存プロジェクト [vagrant-conoha](https://github.com/hironobu-s/vagrant-conoha) のほぼ下位互換です。  
従ってわざわざ使う必要はありません。  
私もこのvagrant-conohaを愛用していましたが、色々と辛みが出てきた為に直接APIを叩こうと決意しました。

- [VagrantfileにAPIのログイン情報を載せる](https://qiita.com/hironobu_s/items/8422a427fd5571747196)ので一手間加えないとGitHub等へ上げられない
- [仕様変更で使えなくなった期間](https://twitter.com/hironobu_s/status/941213992252465152)があり超困った
- [スタートアップスクリプト](https://www.conoha.jp/guide/startupscript.php)のリリースでVagrantプロビジョンへの依存度が下がった
- 勉強用の開発環境は1ファイル1インスタンスで十分、[Dockerもドカドカ動かせる](https://www.conoha.jp/download/books/conoha-book-03-docker.pdf)からね
- そもそも依存元のVagrantがバギーでvagrant-conohaの人にゴルァするべきか判断つかない

vagrant-conohaのヘビーユーザーの内、幾人かは共感できたかと思います。  
しかし、このConoHa CLIも私個人で管理しているので、ConoHaの仕様変更等であっさり使えなくなるかもしれません。  
Vagrantを意識しながらシンプルな設計を心がけていき、フォークやプルリクを気軽に出来るように頑張ります。

## 日本語？マルチ言語対応は？

基本的には日本人しか使わないと思うのでガチガチな日本語にしました。  
CLIも日本語で説明していくようにします。

# Installation: インストール

折角のGo言語製なので様々なOSのバイナリファイルを用意する予定ですが、  
<del>時間が無かったので</del>まだ対応していません。

暫くはGo言語のインストールされているマシンを用意してお使いください。

```Bash
// Macの場合
$ brew install go
$ brew install dep

$ go get github.com/miyabisun/conoha-cli

// 環境変数のGOPATHがデフォルトの状態
$ cd ~/go/src/github.com/miyabisun/conoha-cli

$ dep ensure

$ go build conoha.go

// PATHの通っているディレクトリへ移動
$ mv conoha /usr/local/bin
```

この手順は酷すぎるな…バイナリの配布頑張ろう（汗

# Usage

各種コマンドやサブコマンドにはヘルプが存在します。  
困ったら`-h`オプションを付けて起動してください。

```Bash
$ conoha -h
This tool is a conoha cli.

Usage:
  conoha [flags]
  conoha [command]

Available Commands:
  destroy     destroy in ConoHa API.
  help        Help about any command
  info        get Infomation from ConoHa API.
  login       login to ConoHa API.
  status      status in ConoHa API.
  up          up in ConoHa API.
  version     Print the version number of conoha-cli

Flags:
  -h, --help   help for conoha

Use "conoha [command] --help" for more information about a command.
```

## 全体的な流れ

1. ログインを行う
2. `spec.toml`を作成する
3. インスタンスを操作

### 1. ログインを行う

ConoHa APIは各種操作にトークンIDを要求します。  
その有効期限は丁度1日、一度取得しても日を跨げば何度も取り直しになってしまうわけです。

ConoHa CLIでは`~/.config/conoha.toml`というファイルを作成して、  
ログイン情報やトークンIDを保存するという設計にしています。

初回の`~/.config/conoha.toml`作成にはログインコマンドを使用します。  
コマンドライン引数で設定することも出来ますし、対話シェルで入力することも出来ます。

```Bash
$ conoha login -h
login to ConoHa API.

Usage:
  conoha login [flags]

Flags:
  -h, --help              help for login
  -p, --password string   パスワード (9文字以上)
  -t, --tenantid string   テナントID (半角英数32文字)
  -u, --username string   ユーザー名 (gncu00000000)

# 足りない情報は対話シェルで聞かれます
$ conoha login
username: gncu00000000
password: Hogehoge-0
tenant_id: 1234567890abcdef1234567890abcdef

login successful.
```

### 2. `spec.toml`を作成する

[examples/vps-instance/spec.toml](examples/vps-instance/spec.toml)にサンプルを用意しました。

```TOML
name = "test_machine"
image = "vmi-docker-17.06-ubuntu-16.04-unified"
flavor = "g-1gb"
sshkey = "conoha-ssh-key-name"
```

- name: サーバーリストのネームタグで表示、ConoHa CLIでは一意な名称として扱う
- image: イメージ名
- flavor: プラン名
- sshkey: ConoHaに登録済のSSH Key名

イメージ名、プラン名の候補は下記のコマンドで表示することができます。

```Bash
$ conoha info images
vmi-drupal-8.4.2-centos-7.4-amd64-20gb
vmi-drupal-8.4.2-centos-7.4-amd64
vmi-minecraft-1.12.2-centos-7.4
vmi-centos-7.4-amd64-20gb
vmi-centos-7.4-amd64
...(他にも大量に表示される)

$ conoha info flavors
g-2gb
g-16gb
g-4gb
g-1gb
g-8gb
g-32gb
g-64gb
g-512mb
```

### 3. インスタンスを操作

ConoHa CLIのインスタンス操作は Vagrant を意識してつくられています。  
カレントディレクトリに`spec.toml`があるフォルダで各種コマンドを実行することでインスタンスの状態確認・作成・削除を行います。

```Bash
# インスタンスの状態確認
$ conoha status
NONE

# インスタンスの作成
$ conoha up
$ conoha status
ACTIVE

# インスタンスの削除
$ conoha destroy
$ conoha status
NONE
```

# license: ライセンス

MIT Licenseを適用します。
