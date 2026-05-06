# Docker image for RISC V (RV32IM) simulater

## ビルド方法

リポジトリをクローン

```sh
git clone --depth 1 https://github.com/Iyota-labo/rv32im_sim
```

Docker image をビルド

```sh
docker build -t riscv-rv32im -f docker/Dockerfile .
```

以下のコマンドでテストが可能です

```sh
docker run --rm -v $(pwd):/work riscv-rv32im bash -c "\
    riscv32-unknown-elf-gcc example/hello.c -o hello.elf && \
    spike --isa=rv32im /opt/riscv/riscv32-unknown-elf/bin/pk hello.elf"
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
```

また、以下のコマンドで、作業ディレクトリをコンテナ内と接続し、作業が可能となります。

```sh
docker run -it --rm -v $(pwd):/work riscv-rv32im
```

さらに、以下のように内部で実行したいコマンドを続けて記述することで、コンテナ内でコマンドを実行できます。

```sh
# docker run -it --rm -v $(pwd):/work riscv-rv32im <command>

docker run -it --rm -v $(pwd):/work riscv-rv32im riscv32-unknown-elf-gcc hello.c -o hello.elf
```

ユーザーID、グループIDが1000ではない場合は、以下のように-uオプションでで実行するユーザーIDとグループIDを指定できます。

```sh
docker run -u $(id -u):$(id -g) ...
```
