# Docker image for RISC V (RV32IM) simulater

## ビルド方法

リポジトリをクローン

```sh
git clone --depth 1 https://github.com/Iyota-labo/rv32im_sim
```

Docker image をビルド

```sh
docker build -t riscv-rv32im -f docker/Dockerfile --build-context example=example .
```

以下のコマンドでテストが可能です(イメージに組み込まれている `example/hello.c` を使用します)

```sh
docker run --rm riscv-rv32im bash -c "\
    riscv32-unknown-elf-gcc example/hello.c -o hello.elf && \
    spike --isa=rv32im pk hello.elf"
```

## ビルド済みイメージのPull方法

下記のコマンドでビルド済みDocker imageを ghcr.io から入手できます。

```sh
docker pull ghcr.io/iyota-labo/rv32im_sim:master
```

このイメージを利用する場合は、イメージ名が`ghcr.io/iyota-labo/rv32im_sim:master`となります。

これ以降のコマンドの`riscv-rv32im`を`ghcr.io/iyota-labo/rv32im_sim:master`に置き換えてください。

## 利用方法

以下のコマンドでコンテナ内に入り、作業が可能となります。

```sh
docker run -it --rm riscv-rv32im
# docker run -it --rm ghcr.io/iyota-labo/rv32im_sim:master
```

また、以下のコマンドで、作業ディレクトリをコンテナ内と接続し、作業が可能となります。
コンテナ内の作業ディレクトリは `/home/rv-sim/work` です。

```sh
docker run -it --rm -v $(pwd):/home/rv-sim/work riscv-rv32im
# docker run -it --rm -v $(pwd):/home/rv-sim/work ghcr.io/iyota-labo/rv32im_sim:master
```

さらに、以下のように内部で実行したいコマンドを続けて記述することで、コンテナ内でコマンドを実行できます。

```sh
# docker run -it --rm -v $(pwd):/home/rv-sim/work riscv-rv32im <command>

docker run -it --rm -v $(pwd):/home/rv-sim/work riscv-rv32im riscv32-unknown-elf-gcc hello.c -o hello.elf


# # docker run -it --rm -v $(pwd):/home/rv-sim/work ghcr.io/iyota-labo/rv32im_sim:master <command>

# docker run -it --rm -v $(pwd):/home/rv-sim/work ghcr.io/iyota-labo/rv32im_sim:master riscv32-unknown-elf-gcc
```

ユーザーID、グループIDが1000ではない場合は、以下のように-uオプションでで実行するユーザーIDとグループIDを指定できます。

```sh
docker run -u $(id -u):$(id -g) ...
```

## docker compose での利用

`docker/compose.yml` を使うと、作業ディレクトリ `/home/rv-sim/work` の設定とホスト側 `src/` ディレクトリのマウント(`/home/rv-sim/work/src`)が済んだ状態でコンテナを起動できます。

```sh
docker compose -f docker/compose.yml run --rm rv-sim
```
