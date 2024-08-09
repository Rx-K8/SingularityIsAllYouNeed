# SingularityIsAllYouNeed

🚧 作成中

⚠️ 動かない可能性大

## 1. コンテナイメージ自体をカスタマイズして利用する
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

#### 1.1.2 sifファイルからsandboxコンテナを作成
sifファイルからsandboxコンテナを作成する．
sifファイルは，dockerのイメージに当たる．

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