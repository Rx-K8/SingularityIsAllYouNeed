# SingularityIsAllYouNeed

🚧 作成中

⚠️ フィードバック歓迎

## 1. コンテナイメージをカスタマイズして利用する
対話的にコンテナの環境を構築する方法である．
再現性がない方法になるので，SingularityCE Definition File（以降，SDF）を作成することが好ましい．
しかし，足りないパッケージをその場でインストールできるので，SDFより柔軟に環境構築ができる．
手順は次のとおりになる．

sandboxコンテナを作成する．
sandoboxコンテナを作成後は，2つの起動方法を使い分ける．
- 環境構築をしたいときは，書き込みオプション(--writable)をつけて起動する．
- 実行するときは，GPUを使用するためのオプション(--nv)をつけて起動する．

両方のオプションをつけると起動できるが，GPUを認識しないため，2つのオプションを使い分ける必要がある．

### 1.1 sandboxコンテナ作成
#### 1.1.1 dockerイメージからsandboxコンテナを作成
dockerイメージをDocker Hubから入手してsandboxコンテナにする．

```sh
$ singularity build --sandbox [sandboxコンテナ名] docker://[dockerイメージ名]
```

例として，pytorchが提供しているdockerイメージからsandboxコンテナを作成する方法を示す．

```sh
$ singularity build -s pytorch_edit docker://pytorch/pytorch:2.2.2-cuda11.8-cudnn8-devel
```

#### 1.1.2 Singularity Image Fileからsandboxコンテナを作成
Singularity Image File（以降，SIF）からsandboxコンテナを作成する．
SIFは，dockerのイメージに相当する．

```sh
singularity build --sandbox [sandboxコンテナ名] [sifファイル名].sif
```

### 1.2 書き込みオプションを付けてコンテナを起動する
--writable(-w)で書き込み可能になる．
書き込みにはroot権限が必要になることがあるため，--fakeroot(-f)を使って起動するとよい．

```sh
$ singularity shell -w -f [sandboxコンテナ名]
```

--writableオプションは，読み込み専用のファイルやディレクトリを編集することができるようになる．
つまり，読み込み専用になっていないファイルやディレクトリは，--writableオプションを使わなくて編集できる．

使用例: 
- aptによるパッケージのインストール
- 読み込み専用のファイルやディレクトリの編集

### 1.3 GPUを使って起動する
GPUを使うためには，--nvオプションを付けて起動する．
この起動方法で，実行しながらファイルを編集するとよい．

```sh
$ singularity shell --nv [sandboxコンテナ名]
```

### 1.4 sandboxコンテナをsifファイルに変換
別の環境にsandboxコンテナを使いたい場合，sifファイルを共有するのが（たぶん）一般的である．
sandboxコンテナからsifファイルに変化するには以下のとおりである．

```sh
$ singularity build [sifファイル名].sif [sandboxコンテナ（ディレクトリ）名]
```

## 2. コンテナイメージを少し変更して環境構築
読み込み専用になっていないところの変更であれば，sandboxコンテナを作成しなくても，環境構築ができる．
例えば，「pipでパッケージをインストール」が当てはまる．
ただし，pipでのパッケージのインストール先が読み込み専用になっていれば，不可能である．
pytorchやpythonが公式で提供しているdockerイメージをSIFに変更したのであれば，pipでパッケージをインストールすることができた．
以下に，例を示す．

dockerイメージからSIFを作成する．
```sh
$ singularity build python.sif docker:python:3.11-alpine3.18
```

対話モードでコンテナにログインする．
```sh
$ singularity shell python.sif
```

pipでパッケージをインストールする．

```sh
$ pip install numpy
```

## 3 SingularityCE definition fileでSIFを作成
⚠ CreateSIF.defは動作が不安定である．自己責任で使ってください．


defファイルはdockerfileと近い．
具体的な作り方は，dockerイメージをベースとしたdefファイル"CreateSIF.def"を参考にしてほしい．
細かい作り方は，[公式ドキュメント](https://docs.sylabs.io/guides/4.0/user-guide/definition_files.html)を参照．
sifファイルを作成するには，以下のとおりである．

```sh
$ singularity build [sifファイル名].sif [defファイル名].def
```

ソフトウェアをインストールにroot権限が必要な場合は，sudoや--fakerootモードを使う．
"CreateSIF.def"はroot権限が必要なので，--fakerootモードを使う．

```sh
$ singularity build --fakeroot --build-arg python_version="3.11.9" practice.sif CreateSIF.def
```

こちらは，sandboxに比べ，再現性がある方法になる．
しかし，実行時にパッケージが足りていないなどのエラーが出た場合，イメージを作り直す必要があるので時間がかかる．
柔軟性の面では，sandboxコンテナを使うほうがよい．

## 躓いたこと
### bashrcの読み込み
"singurarity shell"を実行することで，コンテナ内に新しいシェルを生成し，仮想マシンのように操作することができる．
しかし，このままコンテナ内に入ると，.bashrcは読み込まれない．
それは，singularityはnorcオプションを使用して，bashが.bashrcを読み込むのを防いでいる([参照](https://github.com/apptainer/singularity/issues/4808))．
norcオプションを使用しないようにするには，-s(--shell)で指定する必要がある．
それに加えて，-f(--fakeroot)でroot権限を与える必要もある（なぜ，-fオプションが必要か不明）．

次のコマンドで，**ホスト上**の.bashrcを読み込むことが可能になる．
```sh
$ singularity shell --fakeroot --shell /bin/bash [コンテナ名]
```

しかし，**コンテナ内**に存在する.bashrcを読み込みたいときもある．
その際は，--no-homeオプションをつけることで実現できる．
--no-homeオプションは，$HOMEディレクトリをマウントせずに現在の作業ディレクトリをマウントできる（[参照](https://docs.sylabs.io/guides/4.0/user-guide/bind_paths_and_mounts.html#no-home)）．

次のコマンドで，**コンテナ内**の.bashrcを読み込むことが可能になる．
```sh
$ singularity shell --fakeroot --shell /bin/bash --no-home [コンテナ名]
```

それ以外にも以下のようにして.bashrcを読み込むことができる．

- singularity shellで入ったあと，bashで更に入る．
  ```sh
  # ホスト上の.bashrcを読み込む．-fオプションはなくてもあってもよい．
  $ singularity shell [コンテナ名]
  Singularity> bash
  ```

  ```sh
  # コンテナ内の.bashrcを読み込む．-fオプションは必須
  $ singularity shell --no-home -f [コンテナ名]
  Singularity> bash
  ```

- 現在のシェルプロセスを新しいシェルで置き換える．

  ```sh
  # ホスト上の.bashrcを読み込む．-fオプションはなくてもあってもよい．
  $ singularity exec [コンテナ名] bash
  ```

  ```sh
  # コンテナ内の.bashrcを読み込む．-fオプションは必須
  $ $ singularity exec -f --no-home [コンテナ名] bash
  ```

<!-- 
## 個人的知見

### buildコマンド
2つの異なる形式でコンテナを作成できる．
- Singularity Image File(SIF): 圧縮された読み取り専用．
- 書込み可能な(ch)rootディレクトリであるサンドボックスを使用したインタラクティブな開発用．

#### sandboxオプション
書込み可能なディレクトリ（サンドボックス）内にコンテナを作成する場合，--sandboxオプションを使用して作成できる．
サンドボックスコンテナ内で永続的な変更を行うには，コンテナを起動する際に，--writableフラグを使用する．
変更するファイルとディレクトリにアクセスする権限が必要なとき，rootとして実行することを推奨する．
そしくは，--fakerootを使用することがよい． -->