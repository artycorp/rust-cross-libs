# rust-cross-libs

Cross-compile the Rust standard library for unsupported targets without a
full bootstrap build.

Thanks to Kevin Mehall: https://gist.github.com/kevinmehall/16e8b3ea7266b048369d

## Introduction

This guide assumes you are using a x64_86 host to cross-compile the Rust
`std` library to an unsupported target.

### Using custom targets

While it not possible to cross-compile Rust for an unsupported target unless
you hack it, Rust offers the possibility to use custom targets with `rustc`:

From the Rust docs:

> 
A target triple, as passed via `rustc --target=TRIPLE`, will first be
compared against the list of built-in targets. This is to ease distributing
rustc (no need for configuration files) and also to hold these built-in
targets as immutable and sacred. If `TRIPLE` is not one of the built-in
targets, rustc will check if a file named `TRIPLE` exists. If it does, it
will be loaded as the target configuration. If the file does not exist,
rustc will search each directory in the environment variable
`RUST_TARGET_PATH` for a file named `TRIPLE.json`. The first one found will
be loaded. If no file is found in any of those directories, a fatal error
will be given. `RUST_TARGET_PATH` includes `/etc/rustc` as its last entry,
to be searched by default.

> 
Projects defining their own targets should use
`--target=path/to/my-awesome-platform.json` instead of adding to
`RUST_TARGET_PATH`.

Unfortunatly, passing the JSON file path instead of using `RUST_TARGET_PATH`
does not work, so the script internally uses `RUST_TARGET_PATH` to define
the target specification.

## Preparation

### Define your custom target

I will use a custom target `armv5te-unknown-linux-gnueabi` to build a
cross-compiled *"Hello, World!"* for an ARMv5TE soft-float target. Note that
the provided JSON file defines every possible value you can with the current
Rust nightly version.

### Get Rust sources and binaries

We fetch the Rust sources from github and get the binaries from the latest
snapshot to run on your host, e.g. for x86_64-unknown-linux-gnu:

    $ mkdir rust-cross-libs
    $ cd rust-cross-libs
    $ git clone https://github.com/rust-lang/rust rust-git
    $ wget https://static.rust-lang.org/dist/rust-nightly-x86_64-unknown-linux-gnu.tar.gz
    $ tar xf rust-nightly-x86_64-unknown-linux-gnu.tar.gz
    $ rust-nightly-x86_64-unknown-linux-gnu/install.sh --prefix=$PWD/rust

If you're on Arch Linux or any other systen where `cpp` is not in `/lib`:

    $ export CPP=/usr/bin/cpp

### Define the cross toolchain environment

Define your host triple:

    $ export HOST=x86_64-unknown-linux-gnu

Define your cross compiler and linker:

    $ export CC=$HOME/cross/usr/bin/arm-linux-gcc
    $ export AR=$HOME/cross/usr/bin/arm-linux-ar

Define the `CFLAGS` to build the compiler-rt and libbacktrace libraries with:

    $ export CFLAGS="-Wall -Os -fPIC -D__arm__ -mfloat-abi=soft"

Adjust these flags depending on your target.

### Run the script

    $ ./rust-cross-libs.sh --rust-prefix=$PWD/rust --rust-git=$PWD/rust-git --target=$PWD/cfg/armv5te-unknown-linux-gnueabi.json
    [..]
    Libraries are in /home/joerg/rust-cross-libs/rust/lib/rustlib/armv5te-unknown-linux-gnueabi/lib

## Hello, world!

Export path to your host Rust binaries and libraries as well as the path to your
custom target JSON file:

    $ export PATH=$PWD/rust/bin:$PATH
    $ export LD_LIBRARY_PATH=$PWD/rust/lib
    $ export RUST_TARGET_PATH=$PWD/cfg

Cargo the hello example app:

    $ cargo new --bin hello
    $ cd hello
    $ cargo build --target=armv5te-unknown-linux-gnueabi --release

Check:

    $ file target/armv5te-unknown-linux-gnueabi/release/hello
    target/armv5te-unknown-linux-gnueabi/release/hello: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.3, for GNU/Linux 2.6.32, not stripped

    $ size target/armv5te-unknown-linux-gnueabi/release/hello
      text	   data	    bss	    dec	   hex	filename
    116935	   2308	     68	 119255  1d1d7	target/armv5te-unknown-linux-gnueabi/release/hello
