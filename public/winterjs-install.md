---
title: WinterJSをローカル(MacOS)にインストールするために試行錯誤したことまとめ
tags:
  - 'rust'
  - 'javascript'
  - 'winterjs'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

注意：この記事の内容の利用は自己責任でお願いします。
## Summary
- winterjsの依存クレートの一つであるmozjsおよびmozjs_sysのコンパイルが通すのが目標。
- 依存しているソフトウェアのインストールを確認すること。pythonは3.11系統だとトラブルことがなかった。

## WinterJSって？
RustバインディングされたSpiderMonkey(Mozilla製)を用いたサーバーサイド用JavaScriptエンジン。
HTTPサーバとして用いると最速級らしい。
下の記事を読んだ方が速いです。（本題ではないので省かせてもらいます。）

https://zenn.dev/yoshii0110/articles/2ccdc64dfb7f91

## バージョン1.0が発表されたのでローカルで動かしてみたい。
そんな簡単には動かさせてもらえなかった。
公式ではMacOSにインストールする手順が案内されていないので、手探りでやってみる。
WinterJSの公式リポジトリのReadmeによるとRustのパッケージマネージャーcargoのコマンドでインストールできるっぽい。

```shell
$ cargo install --git https://github.com/wasmerio/winterjs winterjs
```

が、うまくインストールできない。
原因を調べると依存クレーとの一つ、mozjs_sysのコンパイルに失敗しているらしい。
エラーをよくみてみると、
```shell
File "/Users/YourUserName/.cargo/git/checkouts/mozjs-e1c76167ba7c548f/91349dc/mozjs-sys/mozjs/python/mozbuild/mozbuild/configure/__init__.py", line 18, in <module>
  from six.moves import builtins as __builtin__
ModuleNotFoundError: No module named 'six.moves'
```
とありmozjsをビルドする際に実行されているPythonスクリプトでエラーが出ている。
まず、RustのライブラリをコンパイルしているときにPythonを動かすなという気持ちは置いといて、このエラーの解決を図る。
1. 自環境で現在利用されているPythonのバージョンを調べる。
```shell
$ python
Python 3.11.8 (main, Mar 13 2024, 17:07:06) [Clang 15.0.0 (clang-1500.3.9.4)] on darwin
```
pythonが3.12系だとなぜか動かなかった。（原因は調べていませんごめんなさい。）

2. 使われているインタプリタでsixが利用可能か調べる。

```shell
$ pip list
Package    Version
---------- -------
pip        24.0
setuptools 65.5.0
six        1.16.0
```
インストールされていなかったら pip install sixでインストールしましょう。
グローバルでインストールできない系のエラーが出たらこの記事などを参考にしてください。

https://news.mynavi.jp/techplus/article/20201129-1537033/

# そもそもインストールするために何が必要なのか。
llvm, python, yasmが必要らしい。
公式リポジトリには何も言及がないが,Github Actionsの設定ファイルを覗くことでMacOSでのビルト時に必要なパッケージがわかる。

```yaml:winterjs/.github/workflows/rust.yml
jobs:
  mac:
    runs-on: macos-13
    strategy:
      fail-fast: false
      matrix:
        features: ["--features debugmozjs", ""]
    env:
      RUSTC_WRAPPER: sccache
      CCACHE: sccache
      SCCACHE_GHA_ENABLED: "true"
    steps:
      - uses: actions/checkout@v3
      - name: Install deps
        run: brew install python llvm yasm
      - name: Run sccache-cache
        uses: mozilla-actions/sccache-action@v0.0.3
      - name: Build
        run: |
          cargo build --verbose ${{ matrix.features }}
          cargo test --verbose ${{ matrix.features }}
```

homebrewを利用して、python,llvm,それとyasm?をインストールしている。
これに倣って自環境にもllvmとyasmをインストールする。
```shell
brew install llvm yasm
```

ここまでやってみてもう一度cargo installを試してみる。

## それでもダメだった場合。
上記のことをやっても最初はうまくビルドできずインストールできませんでした。
次に私がとったアプローチはコンパイルに失敗しているmoz-jsをローカルでコンパイル成功するように修正することです。

winterjsが使用しているmoz-jsのバージョン等を特定します。
```toml:winterjs/Cargo.toml
[dependencies]

# (省略)
mozjs = { version = "0.14.1", git = "https://github.com/wasmerio/mozjs.git", branch = "wasi-gecko", features = [
    "streams",
] }
mozjs_sys = { version = "0.68.2", git = "https://github.com/wasmerio/mozjs.git", branch = "wasi-gecko" }

# (省略)
```
wasmerio/mozjsはservo/mozjsのフォークです。これをgit cloneしてローカル上でビルドできるか調査します。

c++のlinkerの一つであるlldがmacOSで利用できないのに、オプションで指定されておりエラーでした。
サードパーティのld64.lldを利用して正常動作を狙います。
```shell
$ brew install keith/formulae/ld64.lld
```

```diff_shell:mozjs/mozjs-sys/mozjs/build/build-clang/build-clang.py
        extra_ldflags = [
-           "-fuse-ld=lld"
+           "-fuse-ld=ld64.lld",
            "-Wl,-dead_strip",
        ]
```
~~またPythonかよ…~~
これでうまくビルドできればいいですが、"llvm-hogehoge"のコマンドが見つからないよ！みたいなエラーが出たら、パスが通ってるか確認してください。

私が遭遇していないエラーが出てくるかもしれませんが、各自頑張ってください…

## ローカルでビルドしたmozjsを使ってwinterjsをビルドする。
1. winterjsをgit cloneしてきます。
2. winterjs/Cargo.tomlに以下を追加します。
```toml:winterjs/Cargo.toml
[patch."https://github.com/wasmerio/mozjs"]
mozjs = { path= "path/to/your/mozjs/mozjs"}
mozjs_sys={ path="path/to/your/mozjs/mozjs-sys"}
```
これはcargoが特定のurlをふむ時に以下のクレーとのpathにオーバライドしたものを使用するようになります。
これでwinterjsのディレクトリかでcargo buildを実行しwinterjsがビルドできるか試します。

```shell:winterjs
$ cargo build
```

結構時間がかかります。もし、ビルドができたらインストールコマンドを使って、ローカルのリポジトリからインストールできます！
```shell:winterjs
$ cargo install --path .
```

## 最後に
ここまで紹介したのは私がぶつかったエラーです。人によっては別のエラーに遭遇する可能性があります。他のエラーが出たら教えてください。
記事の改善提案PRお待ちしております。

https://github.com/Stead08/qiita-content/tree/main/public/winterjs-install.md