# --- Stage 1: Build Stage ---
FROM debian:trixie AS builder

ENV RISCV=/opt/riscv
ENV PATH="${RISCV}/bin:${PATH}"

ARG TOOLCHAIN_TAG="2026.10.29"
ARG SPIKE_COMMIT="118480991eb16c6a6f40b94fdc35acc9da2ceacd"

# BuildKitキャッシュを使用して高速インストール
# 必要な依存パッケージのインストール
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt update && apt install -y autoconf automake autotools-dev curl \
    python3 python3-pip python3-tomli libmpc-dev libmpfr-dev libgmp-dev gawk \
    build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev \
    libexpat-dev meson ninja-build git cmake libglib2.0-dev expect device-tree-compiler \
    libslirp-dev libzstd-dev libncurses-dev mold libboost-regex-dev libboost-system-dev && \
    apt clean

WORKDIR /tmp/build

# 各コンポーネントをクローンしてビルド
# --depth 1 を使用して .git ディレクトリの肥大化を防ぐ

# 1. riscv-gnu-toolchain
# --disable-gdb: gdbは未使用のため、binutils-gdb.gitの別ブランチへの
# 再クローン・ビルドをスキップしてビルド時間とイメージサイズを削減する
RUN git clone -b ${TOOLCHAIN_TAG} --depth 1 https://github.com/riscv-collab/riscv-gnu-toolchain && \
    cd riscv-gnu-toolchain && mkdir build && cd build &&\
    ../configure --prefix=${RISCV} --with-arch=rv32im --with-abi=ilp32 --enable-newlib --disable-picolibc \
    --disable-gdb --disable-linux --disable-musl --disable-uclibc --disable-multilib && \
    mold -run make -j$(nproc) && \
    cd .. && mkdir build-picolibc && cd build-picolibc &&\
    ../configure --prefix=${RISCV}-picolibc --with-arch=rv32im --with-abi=ilp32 --enable-picolibc --disable-newlib \
    --disable-gdb --disable-linux --disable-musl --disable-uclibc --disable-multilib && \
    mold -run make -j$(nproc) && \
    cd ../.. && rm -rf riscv-gnu-toolchain

# 2. spike (riscv-isa-sim)
RUN git clone https://github.com/riscv-software-src/riscv-isa-sim && \
    cd riscv-isa-sim && git checkout ${SPIKE_COMMIT} && git submodule update --init --recursive && \
    mkdir build && cd build && ../configure --prefix=${RISCV} --with-target=riscv32-unknown-elf && \
    mold -run make -j$(nproc) install && \
    cd ../.. && rm -rf riscv-isa-sim

# 3. riscv-pk
# newlib2linuxブランチを使用して、open() flagsの互換性を確保
RUN git clone -b newlib2linux --depth 1 https://github.com/Iyota-labo/riscv-pk.git && \
    cd riscv-pk && mkdir build && cd build && \
    # CC/CXXを明示的に指定
    CC=${RISCV}/bin/riscv32-unknown-elf-gcc \
    CXX=${RISCV}/bin/riscv32-unknown-elf-g++ \
    ../configure --prefix=${RISCV} --host=riscv32-unknown-elf \
    --with-arch=rv32im_zicsr_zifencei && \
    make -j$(nproc) install && \
    cd ../.. && rm -rf riscv-pk

# riscv-pkが生成したバイナリ(pk, bbl, dummy_payload)を/opt/riscv/binに
# シンボリックリンクし、ターゲットトリプルのサブディレクトリを指定せずに
# PATH経由で直接実行できるようにする
RUN for bin in pk bbl dummy_payload; do \
        ln -sf ../riscv32-unknown-elf/bin/$bin ${RISCV}/bin/$bin; \
    done

# --- Stage 2: Runtime Stage ---
FROM debian:trixie AS runtime

ENV RISCV=/opt/riscv
ENV PATH="${RISCV}/bin:${PATH}"

# ビルド済みのツールチェーン/Spike/riscv-pkのみを引き継ぎ、
# ビルド専用パッケージやソースツリーを最終イメージから排除する
COPY --from=builder /opt/riscv /opt/riscv

# 対話利用に最低限必要なユーティリティと、ビルドステージで生成された
# バイナリ(gcc, spike など)が実行時に必要とする共有ライブラリ・外部コマンドをインストール
# (device-tree-compiler: spikeがシミュレーション起動時にdtcコマンドを呼び出すため必要)
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt update && apt upgrade -y && apt install -y sudo nano bash-completion cmake make meson python3-pip\
    ninja-build file less device-tree-compiler libmpc3 libmpfr6 libboost-regex1.83.0 libboost-system1.83.0 \
    git-lfs && \
    apt clean

# 1. 環境変数の定義 (ユーザー名やUIDを指定)
ARG USERNAME=rv-sim
ARG USER_UID=1000
ARG USER_GID=1000

# 2. ユーザーとグループの作成、passwordless sudoの付与
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# 3. 作業ディレクトリの準備
RUN mkdir -p /home/${USERNAME}/work && chown $USERNAME:$USER_GID /home/${USERNAME}/work

# example/ ディレクトリを名前付きビルドコンテキスト(example)から取り込む
# docker buildの --build-context example=example、または
# compose.ymlのadditional_contexts、CIのbuild-contextsで指定する
COPY --from=example --chown=$USER_UID:$USER_GID . /home/rv-sim/work/example

WORKDIR /home/${USERNAME}/work

# 4. デフォルトユーザーを切り替え
USER $USERNAME

RUN echo "alias spike-rv32im='spike --isa=RV32IM ${RISCV}/riscv32-unknown-elf/bin/pk'" >> /home/${USERNAME}/.bashrc

CMD ["/bin/bash"]
