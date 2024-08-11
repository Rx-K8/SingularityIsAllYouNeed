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

### 1.1 sandboxコンテナ作成する
#### 1.1.1 dockerイメージからsandboxコンテナを作成する
dockerイメージをDocker Hubから入手してsandboxコンテナにする．

```sh
$ singularity build --sandbox [sandboxコンテナ名] docker://[dockerイメージ名]
```

#### 1.1.2 Singularity Image Fileからsandboxコンテナを作成する
Singularity Image File（以降，SIF）からsandboxコンテナを作成する．
SIFは，dockerのイメージに相当する．

```sh
singularity build --sandbox [sandboxコンテナ名] [SIF名].sif
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

### 1.4 sandboxコンテナをSIFに変換する
別の環境にsandboxコンテナを使いたい場合，SIFを共有するのが（たぶん）一般的である．
sandboxコンテナからSIFに変化するには以下のとおりである．

```sh
$ singularity build [SIF名].sif [sandboxコンテナ（ディレクトリ）名]
```

### 1.5 実際にやってみる
コンテナイメージをカスタマイズして利用する方法を実際にやってみる．
NVIDIA社が公開しているDockerイメージを使って，pytorchで学習できるようにする．
CUDA周りはバージョンがシビアである．
ここでは，説明しないが，[解説している記事](https://qiita.com/ketaro-m/items/4de2bd3101bcb6a6b668)もあるのでそちらを参考に．

#### 1.5.1 dockerイメージからsandboxコンテナを作成する

```sh
# sandboxコンテナ外
$ singularity build --sandbox practice docker://nvidia/cuda:12.1.1-cudnn8-devel-ubuntu22.04
```

#### 1.5.2 sandboxコンテナ環境を構築する
sandboxコンテナを起動する．
いろんなオプションをつけている．
[その他](##その他)を参照してください．

```sh
# sandboxコンテナ外
$ singularity shell -w -f --no-home -s /bin/bash practice
```

必要なものをインストール，設定する．

```sh
# sandboxコンテナ内
$ echo 'export LANG=ja_JP.UTF-8' >> $HOME/.bashrc
$ apt update
$ apt install -y locales-all language-pack-ja-base build-essential libssl-dev zlib1g-dev liblzma-dev libreadline-dev libbz2-dev libsqlite3-dev libffi-dev libncurses-dev wget curl git zip unzip
$ echo '. "$HOME/.asdf/asdf.sh"' >> $HOME/.bashrc
$ echo '. "$HOME/.asdf/completions/asdf.bash"' >> $HOME/.bashrc
$ git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0
$ exit

# sandboxコンテナ外
$ singularity shell -w -f --no-home -s /bin/bash practice

# sandboxコンテナ内
$ asdf plugin-add python
$ asdf install python 3.11.9
$ asdf global python 3.11.9
$ pip install torch==2.2.2 torchvision==0.17.2 torchaudio==2.2.2 --index-url https://download.pytorch.org/whl/cu121
```

#### 1.5.3 sandboxコンテナ内でpythonファイルを実行する
GPUを使える環境でsandboxコンテナを起動する．

```sh
# sandboxコンテナ外
$ singularity shell --nv -f --no-home -s /bin/bash practice
```

適当なpytorchの学習コードを用意する．
これでGPUを使った機械学習が可能である．

```sh
# sandboxコンテナ内
$ python train.py
```

#### 1.5.4 sandboxコンテナ外からpythonファイルを実行する
sandboxコンテナ外からpythonファイルを実行しようとすると以下のとおりになる．
pythonコマンドまでのパスをフルパスで書いており，冗長である．
"/root/.asdf/shims/python"でなく，"python"だけでコマンドを叩くためには，環境変数PATHに追加する必要がある．

```sh
# sandboxコンテナ外
$ singularity exec --nv -f --no-home practice /root/.asdf/shims/python train.py
```

そこで，**sandboxコンテナ内**のenviromentファイル(/enviroment)に以下を記述する．
これで，pythonコマンドの実行が可能になる．

```sh
# envirimentファイル内
PATH="${PATH}:${HOME}/.asdf/shims"
```

これにより，以下のように実行できる．

```sh
# sandboxコンテナ外
$ singularity exec --nv -f --no-home practice python train.py
```

ここで，bashrcに記述すればよいと思うかもしれない．
[その他](##その他)に詳しく説明するが，"singularity exec"はbashrcを読み込まないようになっている．
そのため，bashrc内で環境変数PATHを変更したところで意味がなく，enviromentファイルに書き込むようにしている．

#### 1.5.5 SIFにしてpythonファイルを実行する
sandboxコンテナをSIFに変換する．

```sh
$ singularity build practice.sif practice
```
SIFを通してpythonファイル実行してみる．

```sh
$ singularity exec --nv -f --no-home practice.sif python train.py
```

### 1.6 個人的な見解
実際に，NVIDIA社が提供しているDockerイメージを使って，pytorchで機械学習してみた．
機械学習が目的であれば，PyTorchやTensorflowを使うだろう．
今回作成したコンテナイメージは問題点が多いのであくまで参考程度にとどめてほしい．
特に，理由がなければ，PyTorchやTeonsorflowの公式が提供しているDockerイメージを使うことを推奨する．
設定などめんどくさいことを省けるので．

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

ログインする．
```sh
$ singularity shell python.sif
```

pipでパッケージをインストールする．
実際に，インストールされていることを確認する．

```sh
$ pip install numpy
$ pip list
```

## 3 SDFからSIFを作成する
⚠ CreateSIF.defは動作が不安定である．自己責任で使ってください．

SDFはdockerfileと近い．
具体的な作り方は，dockerイメージをベースとしたSDF"CreateSIF.def"を参考にしてほしい．
細かい作り方は，[公式ドキュメント](https://docs.sylabs.io/guides/4.0/user-guide/definition_files.html)を参照．
SIFを作成するには，以下のとおりである．

```sh
$ singularity build [SIF名].sif [SDF名].def
```

ソフトウェアをインストールにroot権限が必要な場合は，sudoや--fakerootモードを使う．
"CreateSIF.def"はroot権限が必要なので，--fakerootモードを使う．

```sh
$ singularity build --fakeroot --build-arg python_version="3.11.9" practice.sif CreateSIF.def
```

こちらは，sandboxに比べ，再現性がある方法になる．
しかし，実行時にパッケージが足りていないなどのエラーが出た場合，イメージを作り直す必要があるので時間がかかる．
柔軟性の面では，sandboxコンテナを使うほうがよい．

## その他
### おすすめオプション

```sh
$ singularity shell --shell /bin/bash -f --no-home --nv [コンテナ名]
```

### --fakeroot(-f)の動作
--fakerootありとなしのコンテナ内のホームディレクトリ(\$HOME)に違いが生まれる．
--fakerootありだと，コンテナ内のホームディレクトリは"/root"になっていた．
その/rootは，ホスト上の\$HOMEがマウントされていた．
```sh
$ singularity shell -f [コンテナ名]
Singularity> echo $HOME
/root
Singularity> ls -a $HOME
ホスト上のホームディレクトリ直下にあるディレクトリやファイルが表示される
```

--fakerootなしだと，コンテナ内のホームディレクトリは"/home/ユーザ名"，つまり，ホスト上のホームディレクトリになっていた．
```sh
$ singularity shell [コンテナ名]
Singularity> echo $HOME
/home/ユーザ名
Singularity> ls -a $HOME
ホスト上のホームディレクトリ直下にあるディレクトリやファイルが表示される
```

結局のところ，コンテナ内のホームディレクトリは，--fの有無で変わらない．

しかし，--fakerootと--no-homeを組み合わせて使う場合に，ややこしくなる．
--fakerootと--no-homeありだと，コンテナ内のホームディレクトリは"/root"になっていた．
その/rootは，コンテナ内の/rootがマウントされていた．
```sh
$ singularity shell --no-home -f [コンテナ名]
Singularity> echo $HOME
/root
Singularity> ls -a $HOME
コンテナ内の/root直下にあるディレクトリやファイルが表示される
```

--no-homeだと，コンテナ内のホームディレクトリは"/home/ユーザ名"，つまり，ホスト上のホームディレクトリになっていた．
しかし，ホスト上のホームディレクトリをマウントしていないため，何も存在しない．
厳密には，コンテナがあるディレクトリは存在する．
```sh
$ singularity shell --no-home [コンテナ名]
Singularity> echo $HOME
/home/keito_fukuoka
Singularity> ls -a $HOME
マウントしていないため，ホスト上のホームディレクトリ直下にあるディレクトリやファイルが表示されない．
厳密には，コンテナがあるディレクトリは表示される．
```

### bashrcの読み込み
"singurarity shell"を実行することで，コンテナ内に新しいシェルを生成し，仮想マシンのように操作することができる．
しかし，このままコンテナ内に入ると，.bashrcは読み込まれない．
それは，singularityはnorcオプションを使用して，bashが.bashrcを読み込むのを防いでいる([参照](https://github.com/apptainer/singularity/issues/4808))．
norcオプションを使用しないようにするには，-s(--shell)で指定する必要がある．
それに加えて，-f(--fakeroot)でroot権限を与える必要もある．
先ほど説明したように，--fakeroot有無，--no-homeとの組み合わせで\$HOMEのディレクトリが変わるので注意が必要である．

次のコマンドで，**ホスト上**の.bashrcを読み込むことが可能になる．
```sh
$ singularity shell --fakeroot --shell /bin/bash [コンテナ名]
```

しかし，**コンテナ内**に存在する.bashrcを読み込みたいときもある．
その際は，--no-homeオプションをつけることで実現できる．
--no-homeオプションは，\$HOMEディレクトリをマウントせずに現在の作業ディレクトリをマウントできる（[参照](https://docs.sylabs.io/guides/4.0/user-guide/bind_paths_and_mounts.html#no-home)）．

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
  $ singularity exec -f --no-home [コンテナ名] bash
  ```