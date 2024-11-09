+++
title = "交叉编译 OpenSSL"
date = 2024-04-11
[taxonomies]
  tags = ["rust", "cross compile", "openssl"]
+++

Debian 交叉编译 rdkafka 时报错，直接在 centos7 上安装了 rustup, 但是需要安装一些环境：

```bash
yum groupinstall "Development Tools"
yum install cyrus-sasl-devel
yum install cmake
```

cmake 版本过低, 升级 cmake

```bash
yum remove cmake -f
wget -c https://cmake.org/files/v3.22/cmake-3.22.6.tar.gz
gunzip -c cmake-3.22.6.tar.gz | tar xopf -
cd cmake-3.22.6
./configure --prefix=/usr/local/cmake
make && make install
ln -s /usr/local/cmake/bin/cmake /usr/bin/cmake

$vim /etc/profile
export CMAKE_HOME=/usr/local/cmake
export PATH=$PATH:$CMAKE_HOME/bin
```

## 交叉编译

```bash
cargo build --release --target x86_64-unknown-linux-musl
```

编译的时候报错：

```

  --- stderr
  thread 'main' panicked at /home/sxw/.cargo/registry/src/index.crates.io-6f17d22bba15001f/sasl2-sys-0.1.20+2.1.28/build.rs:342:5:
  Unable to find libsasl2 on your system. Hints:

    * Have you installed the libsasl2 development package for your platform?
      On Debian-based systems, try libsasl2-dev. On RHEL-based systems, try
      cyrus-sasl-devel. On macOS with Homebrew, try cyrus-sasl.

    * Have you incorrectly set the SASL2_STATIC environment variable when your
      system only supports dynamic linking?

    * Are you willing to enable the `vendored` feature to instead build and link
      against a bundled copy of libsasl2?

  note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
warning: build failed, waiting for other jobs to finish...
```

交叉编译失败。

使用 `cargo install cross --git https://github.com/cross-rs/cross` 进行交叉编译


交叉编译到 Windows:

```bash
cargo install cargo-xwin
rustup target add x86_64-pc-windows-msvc
cargo xwin build --target x86_64-pc-windows-msvc --release
```

或使用 zigbuild:

```bash
cargo install cargo-zigbuild
rustup target add x86_64-pc-windows-msvc
cargo zigbuild --target x86_64-pc-windows-msvc --release
```

```bash
ln -s /usr/include/x86_64-linux-gnu/asm /usr/include/x86_64-linux-musl/asm &&
ln -s /usr/include/asm-generic /usr/include/x86_64-linux-musl/asm-generic &&
ln -s /usr/include/linux /usr/include/x86_64-linux-musl/linux

mkdir /musl


wget https://github.com/openssl/openssl/archive/OpenSSL_1_1_1f.tar.gz
tar zxvf OpenSSL_1_1_1f.tar.gz
cd openssl-OpenSSL_1_1_1f/

CC="musl-gcc -fPIE -pie" ./Configure no-shared no-async --prefix=/musl --openssldir=/musl/ssl linux-x86_64
make depend
make -j$(nproc)
make install
```

`openssl` 安装完成后, 导出环境配置:

```bash
sudo apt install musl-tools

export PKG_CONFIG_ALLOW_CROSS=1
export OPENSSL_STATIC=true
export OPENSSL_DIR=/musl
cargo build --target x86_64-unknown-linux-musl --release
```

或

```bash
wget https://www.openssl.org/source/openssl-1.0.1t.tar.gz
tar xzf openssl-1.0.1t.tar.gz
export MACHINE=armv7
export ARCH=arm
export CC=arm-linux-gnueabihf-gcc
cd openssl-1.0.1t && ./config shared && make && cd -

export OPENSSL_LIB_DIR=~/openssl-1.0.1t/
export OPENSSL_INCLUDE_DIR=~/openssl-1.0.1t/include
cargo build --target=armv7-unknown-linux-gnueabi --release
```

参考了以下链接:

- https://users.rust-lang.org/t/cant-cross-compile-project-with-openssl/70922
- https://blog.xco.moe/posts/rust_build_musl/
- https://xuanwo.io/2021/10-rust-cross-aarch64/
