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
RUN git clone --depth 1 https://github.com/riscv-software-src/riscv-pk && \
    cd riscv-pk && mkdir build && cd build && \
    # CC/CXXを明示的に指定
    CC=${RISCV}/bin/riscv32-unknown-elf-gcc \
    CXX=${RISCV}/bin/riscv32-unknown-elf-g++ \
    ../configure --prefix=${RISCV} --host=riscv32-unknown-elf \
    --with-arch=rv32im_zicsr_zifencei && \
    make -j$(nproc) install && \
    cd ../.. && rm -rf riscv-pk

WORKDIR /work
CMD ["/bin/bash"]