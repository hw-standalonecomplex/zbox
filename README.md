<img src="https://www.zbox.io/svg/logo.svg" alt="Zbox Logo" height="96" /> Zbox
======
[![Travis](https://img.shields.io/travis/zboxfs/zbox.svg?style=flat-square)](https://travis-ci.org/zboxfs/zbox)
[![Crates.io](https://img.shields.io/crates/d/zbox.svg?style=flat-square)](https://crates.io/crates/zbox)
[![Crates.io](https://img.shields.io/crates/v/zbox.svg?style=flat-square)](https://crates.io/crates/zbox)
[![GitHub last commit](https://img.shields.io/github/last-commit/zboxfs/zbox.svg?style=flat-square)](https://github.com/zboxfs/zbox)
[![license](https://img.shields.io/github/license/zboxfs/zbox.svg?style=flat-square)](https://github.com/zboxfs/zbox)
[![GitHub stars](https://img.shields.io/github/stars/zboxfs/zbox.svg?style=social&label=Stars)](https://github.com/zboxfs/zbox)

Zbox is a zero-details, privacy-focused embeddable file system. Its goal is
to help application store files securely, privately and reliably. By
encapsulating files and directories into an encrypted repository, it provides
a virtual file system and exclusive access to authorised application.

Unlike other system-level file systems, such as [ext4], [XFS] and [Btrfs], which
provide shared access to multiple processes, Zbox is a file system that runs
in the same memory space as the application. It only provides access to one
process at a time.

By abstracting IO access, Zbox supports a variety of underlying storage layers,
including memory, OS file system, RDBMS and key-value object store.

## Disclaimer

Zbox is under active development, we are not responsible for any data loss
or leak caused by using it. Always back up your files and use at your own risk!

Features
========
- Everything is encrypted :lock:, including metadata and directory structure,
  no knowledge can be leaked to underlying storage
- State-of-the-art cryptography: AES-256-GCM (hardware), XChaCha20-Poly1305,
  Argon2 password hashing and etc., empowered by [libsodium]
- Support multiple storages, including memory, OS file system, RDBMS, Key-value
  object store and more
- Content-based data chunk deduplication and file-based deduplication
- Data compression using [LZ4] in fast mode, optional
- Data integrity is guaranteed by authenticated encryption primitives (AEAD
  crypto)
- File contents versioning
- Copy-on-write (COW :cow:) semantics
- ACID transactional operations
- Built with [Rust] :hearts:

## Comparison

Many OS-level file systems support encryption, such as [EncFS], [APFS] and
[ZFS]. Some disk encryption tools also provide virtual file system, such as
[TrueCrypt], [LUKS] and [VeraCrypt].

This diagram shows the difference between Zbox and them.

![Comparison](https://www.zbox.io/svg/zbox-compare.svg)

Below is the feature comparison list.

|                             | Zbox                     | OS-level File Systems    | Disk Encryption Tools    |
| --------------------------- | ------------------------ | ------------------------ | ------------------------ |
| Encrypts file contents      | :heavy_check_mark:       | partial                  | :heavy_check_mark:       |
| Encrypts file metadata      | :heavy_check_mark:       | partial                  | :heavy_check_mark:       |
| Encrypts directory          | :heavy_check_mark:       | partial                  | :heavy_check_mark:       |
| Data integrity              | :heavy_check_mark:       | partial                  | :heavy_multiplication_x: |
| Shared access for processes | :heavy_multiplication_x: | :heavy_check_mark:       | :heavy_check_mark:       |
| Deduplication               | :heavy_check_mark:       | :heavy_multiplication_x: | :heavy_multiplication_x: |
| Compression                 | :heavy_check_mark:       | partial                  | :heavy_multiplication_x: |
| Versioning                  | :heavy_check_mark:       | :heavy_multiplication_x: | :heavy_multiplication_x: |
| COW semantics               | :heavy_check_mark:       | partial                  | :heavy_multiplication_x: |
| ACID Transaction            | :heavy_check_mark:       | :heavy_multiplication_x: | :heavy_multiplication_x: |
| Multiple storage layers     | :heavy_check_mark:       | :heavy_multiplication_x: | :heavy_multiplication_x: |
| API access                  | :heavy_check_mark:       | through VFS              | through VFS              |
| Symbolic links              | :heavy_multiplication_x: | :heavy_check_mark:       | depends on inner FS      |
| Users and permissions       | :heavy_multiplication_x: | :heavy_check_mark:       | :heavy_check_mark:       |
| FUSE support                | :heavy_multiplication_x: | :heavy_check_mark:       | :heavy_check_mark:       |
| Linux and macOS support     | :heavy_check_mark:       | :heavy_check_mark:       | :heavy_check_mark:       |
| Windows support             | :heavy_check_mark:       | partial                  | :heavy_check_mark:       |

## Supported Storage

By now, Zbox supports a variety of underlying storage, which are listed below.
Memory and OS file storage are enabled by default, all the others can be
enabled individually by specifying its feature when build.

| Storage        | URI identifier  | Feature        |
| -------------- | --------------- | -------------- |
| Memory         | "mem://"        | N/A            |
| OS file system | "file://"       | N/A            |
| SQLite         | "sqlite://"     | storage-sqlite |
| Redis          | "redis://"      | storage-redis  |

There is another special storage `Faulty` ("faulty://"), which is based on
memory storage and can simulate random IO error. It is used internally to
facilitate random IO error test.

How to use
==========
For reference documentation, please visit [documentation](https://docs.rs/zbox).

## Requirements

- [Rust] stable >= 1.21
- [libsodium] >= 1.0.16

## Supported Platforms

- 64-bit Debian-based Linux, such as Ubuntu
- 64-bit macOS
- 64-bit Windows
- 64-bit Android, API level >= 21 (in progress)

32-bit and other OS are `NOT` supported yet.

## Usage

Add the following dependency to your `Cargo.toml`:

```toml
[dependencies]
zbox = "0.6.1"
```

## Example

```rust
extern crate zbox;

use std::io::{Read, Write};
use zbox::{init_env, RepoOpener, OpenOptions};

fn main() {
    // initialise zbox environment, called first
    init_env();

    // create and open a repository in current OS directory
    let mut repo = RepoOpener::new()
        .create(true)
        .open("file://./my_repo", "your password")
        .unwrap();

    // create and open a file in repository for writing
    let mut file = OpenOptions::new()
        .create(true)
        .open(&mut repo, "/my_file.txt")
        .unwrap();

    // use std::io::Write trait to write data into it
    file.write_all(b"Hello, world!").unwrap();

    // finish writing to make a permanent content version
    file.finish().unwrap();

    // read file content using std::io::Read trait
    let mut content = String::new();
    file.read_to_string(&mut content).unwrap();
    assert_eq!(content, "Hello, world!");
}
```

## Build with Docker

Zbox comes with [Docker] support, it is easier to build Zbox with [zboxfs/base]
image. This image is based on Ubuntu 16.04 and has Rust stable and libsodium
included. Check more details in the [Dockerfile](docker/base.docker).

You can also use image [zboxfs/android] to build Zbox for Android. It is based
on [zboxfs/base] image and has Android NDK included. Check more details in the
[Dockerfile](docker/android.docker).

To build for Linux x86_64:
```bash
docker run --rm -v $PWD:/root/zbox zboxfs/base cargo build
```

To build for Android x86_64:
```bash
docker run --rm -v $PWD:/root/zbox zboxfs/android cargo build --target x86_64-linux-android
```

To build for Android arm64:
```bash
docker run --rm -v $PWD:/root/zbox zboxfs/android cargo build --target aarch64-linux-android
```

Or run the test suite.
```bash
docker run --rm -v $PWD:/root/zbox zboxfs/base cargo test
```

## Static linking with libsodium

By default, Zbox uses dynamic linking when it is linked with libsodium. If you
want to change this behavior and use static linking, you can enable below two
environment variables.

On Linux/macOS,

```bash
export SODIUM_LIB_DIR=/path/to/your/libsodium/lib
export SODIUM_STATIC=true
```

On Windows,

```bash
set SODIUM_LIB_DIR=C:\path\to\your\libsodium\lib
set SODIUM_STATIC=true
```

And then re-build the code.

```bash
cargo build
```

Performance
============

The performance test is run on a Macbook Pro 2017 laptop with spec as below.

| Spec                    | Value                       |
| ----------------------- | --------------------------- |
| Processor Name:         | Intel Core i7               |
| Processor Speed:        | 3.5 GHz                     |
| Number of Processors:   | 1                           |
| Total Number of Cores:  | 2                           |
| L2 Cache (per Core):    | 256 KB                      |
| L3 Cache:               | 4 MB                        |
| Memory:                 | 16 GB                       |
| OS Version:             | macOS High Sierra 10.13.6   |

Test result:

|                               | Read            | Write          | TPS          |
| ----------------------------- | --------------- | -------------- | ------------ |
| Baseline (memcpy):            | 3658.23 MB/s    | 3658.23 MB/s   | N/A          |
| Baseline (file):              | 1307.97 MB/s    | 2206.30 MB/s   | N/A          |
| Memory storage (no compress): | 605.01 MB/s     | 186.20 MB/s    | 267 tx/s     |
| Memory storage (compress):    | 505.04 MB/s     | 161.11 MB/s    | 263 tx/s     |
| File storage (no compress):   | 435.66 MB/s     | 147.44 MB/s    | 117 tx/s     |
| File storage (compress):      | 372.60 MB/s     | 124.92 MB/s    | 106 tx/s     |

To run the performance test on your own computer, please follow the
instructions in [CONTRIBUTING.md](CONTRIBUTING.md#run-performance-test).

Contribution
============

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall
be licensed as above, without any additional terms of conditions.

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of
conduct, and the process for submitting pull requests to us.

Community
=========

- [Gitter Chat Room](https://gitter.im/zboxfs/zbox)
- [Twitter](https://twitter.com/ZboxFS)

License
=======
`Zbox` is licensed under the Apache 2.0 License - see the [LICENSE](LICENSE)
file for details.

[ext4]: https://en.wikipedia.org/wiki/Ext4
[xfs]: http://xfs.org
[btrfs]: https://btrfs.wiki.kernel.org
[Rust]: https://www.rust-lang.org
[libsodium]: https://libsodium.org
[LZ4]: http://www.lz4.org
[EncFS]: https://vgough.github.io/encfs/
[APFS]: https://en.wikipedia.org/wiki/Apple_File_System
[ZFS]: https://en.wikipedia.org/wiki/ZFS
[TrueCrypt]: http://truecrypt.sourceforge.net
[LUKS]: https://gitlab.com/cryptsetup/cryptsetup/
[VeraCrypt]: https://veracrypt.codeplex.com
[rust:latest]: https://hub.docker.com/_/rust/
[Docker]: https://www.docker.com
[zboxfs/base]: https://hub.docker.com/r/zboxfs/base/
[zboxfs/android]: https://hub.docker.com/r/zboxfs/android/
