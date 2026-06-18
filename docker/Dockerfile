# --- Stage 1: Build Stage ---
FROM debian:bookworm AS builder

ENV RISCV=/opt/riscv
ENV PATH="${RISCV}/bin:${PATH}"

# BuildKitキャッシュを使用して高速インストール
# 必要な依存パッケージのインストール
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y autoconf automake autotools-dev curl python3 python3-pip python3-tomli libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake libglib2.0-dev libslirp-dev libncurses-dev mold device-tree-compiler libboost-regex-dev libboost-system-dev && apt-get clean

WORKDIR /build

# 各コンポーネントをクローンしてビルド
# --depth 1 を使用して .git ディレクトリの肥大化を防ぐ

# 1. riscv-gnu-toolchain
RUN git clone --depth 1 https://github.com/riscv-collab/riscv-gnu-toolchain && \
    cd riscv-gnu-toolchain && \
    ./configure --prefix=${RISCV} --with-arch=rv32im --with-abi=ilp32 && \
    mold -run make -j$(nproc) && \
    cd .. && rm -rf riscv-gnu-toolchain

# 2. spike (riscv-isa-sim)
RUN git clone --depth 1 https://github.com/riscv-software-src/riscv-isa-sim && \
    cd riscv-isa-sim && mkdir build && cd build && \
    ../configure --prefix=${RISCV} --with-target=riscv32-unknown-elf && \
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

# 1. 環境変数の定義 (ユーザー名やUIDを指定)
ARG USERNAME=riscv-sim
ARG USER_UID=1000
ARG USER_GID=1000

# 2. ユーザーとグループの作成
# 開発時にsudoを使いたくなる場合に備え、sudo権限も付与しておくと親切です
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME
    # && apt-get update \
    # && apt-get install -y sudo \
    # && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    # && chmod 0440 /etc/sudoers.d/$USERNAME

# 3. ツールチェーン(/opt/riscv)の所有者を変更
# これにより、一般ユーザーでもライブラリの追加インストールなどが可能になります
# RUN chown -R $USERNAME:$USER_GID /opt/riscv

# 4. 作業ディレクトリの準備
RUN mkdir -p /work && chown $USERNAME:$USER_GID /work
WORKDIR /work

# 5. デフォルトユーザーを切り替え
USER $USERNAME

CMD ["/bin/bash"]