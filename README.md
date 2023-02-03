# eunomia-bpf: simplify and enhance eBPF with CO-RE[^1] and WebAssembly[^2]

[![Actions Status](https://github.com/eunomia-bpf/eunomia-bpf/workflows/Ubuntu/badge.svg)](https://github.com/eunomia-bpf/eunomia-bpf/actions)
[![GitHub release (latest by date)](https://img.shields.io/github/v/release/eunomia-bpf/eunomia-bpf)](https://github.com/eunomia-bpf/eunomia-bpf/releases)
[![codecov](https://codecov.io/gh/eunomia-bpf/eunomia-bpf/branch/master/graph/badge.svg?token=YTR1M16I70)](https://codecov.io/gh/eunomia-bpf/eunomia-bpf)
[![DeepSource](https://deepsource.io/gh/eunomia-bpf/eunomia-bpf.svg/?label=active+issues&show_trend=true&token=rcSI3J1-gpwLIgZWtKZC-N6C)](https://deepsource.io/gh/eunomia-bpf/eunomia-bpf/?ref=repository-badge)
[![CodeFactor](https://www.codefactor.io/repository/github/eunomia-bpf/eunomia-bpf/badge)](https://www.codefactor.io/repository/github/eunomia-bpf/eunomia-bpf)

**A compiler and runtime framework to help you build and distribute eBPF program easier.**

## Introduction

`eunomia-bpf` is a dynamic loading library/runtime and a compile toolchain framework, aim at helping you build and distribute eBPF programs easier.

With eunnomia-bpf, you can:

- simplify and enhance the process of `writing` eBPF programs:
  - simplify building CO-RE[^1] `libbpf` eBPF applications: [write eBPF kernel code only](#simplify-building-co-re-libbpf-ebpf-applications) and automatically exposing your data with `perf event` or `ring buffer` from kernel.
  - [Automatically sample the data](#automatically-sample-the-data-and-print-hists-in-userspace) from hash maps and print `hists` in userspace.
  - [Automatically generate](#automatically-generate-and-config-command-line-arguments) and config `command line arguments` for eBPF programs.
  - You can writing the kernel part in both `BCC` and `libbpf` styles:
    - enable CO-RE[^1] to BCC-style eBPF programs without depending on the LLVM library.
    - a [converter](https://github.com/eunomia-bpf/bcc/tree/master/src/cc/converter) to convert BCC stype source code to libbpf style source code.
- Build eBPF programs with `Wasm`[^2]:
  - [Write eBPF with Wasm](https://github.com/eunomia-bpf/wasm-bpf) in C/C++, Rust, Go...Any language you like in WebAssembly[^2], see the [`Wasm-bpf`](https://github.com/eunomia-bpf/wasm-bpf) project.
  - A runtime module for providing most abilities from kernel eBPF to userspace Wasm, covering the use cases from `tracing`, `networking`, `security`.
  - Toolchains and libraries for building eBPF programs to Wasm, provide a similar developing experience as the [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap).
- simplify the process of `distributing` eBPF programs:
  - A [tool](ecli/ecli-rs/) for push, pull and run pre-compiled eBPF programs as `OCI` images in Wasm module
  - Run eBPF programs from `cloud` or `URL` within [`1` line of bash](#dynamic-load-and-run-co-re-ebpf-kernel-code-from-the-cloud-with-url-or-oci-image) without recompiling, kernel version and architecture independent.
  - [Dynamically load](bpf-loader) eBPF programs with `JSON` config file or `Wasm` module.

[^1]: CO-RE: [Compile Once – Run Everywhere](https://facebookmicrosites.github.io/bpf/blog/2020/02/19/bpf-portability-and-co-re.html)
[^2]: WebAssembly or Wasm: https://webassembly.org/

## Getting Started

- Github Template：[eunomia-bpf/ebpm-template](https://github.com/eunomia-bpf/ebpm-template)
- example bpf programs: [examples/bpftools](examples/bpftools/)
- tutorial: [eunomia-bpf/bpf-developer-tutorial](https://github.com/eunomia-bpf/bpf-developer-tutorial)

## build or install the project

- Install the `ecli` tool for running eBPF program from the cloud:

    ```console
    $ wget https://aka.pw/bpf-ecli -O ecli && chmod +x ./ecli
    $ ./ecli -h
    Usage: ecli [--help] [--version] [--json] [--no-cache] url-and-args
    ....
    ```

- Install the `ecc` compiler-toolchain for compiling eBPF kernel code to a `config` file or `WASM` module(`clang`, `llvm`, and `libclang` should be installed for compiling):

    ```console
    $ wget https://github.com/eunomia-bpf/eunomia-bpf/releases/latest/download/ecc && chmod +x ./ecc
    $ ./ecc -h
    eunomia-bpf compiler
    Usage: ecc [OPTIONS] <SOURCE_PATH> [EXPORT_EVENT_HEADER]
    ....
    ```

  or use the docker image for compile:

    ```bash
    # for x86_64 and aarch64
    docker run -it -v `pwd`/:/src/ ghcr.io/eunomia-bpf/ecc-`uname -m`:latest # compile with docker. `pwd` should contains *.bpf.c files and *.h files.
    ```

- build the compiler, runtime library and tools:

  see [build](documents/build.md) for building details.

### Simplify building CO-RE libbpf eBPF applications

Just Write libbpf eBPF kernel code only, auto config the userspace part!

#### Automatically exposing your data from kernel

- Get data automatically from `perf event` or `ring buffer` to userspace:

    ```c
    struct {
      __uint(type, BPF_MAP_TYPE_RINGBUF);
      __uint(max_entries, 256 * 1024);
    } rb SEC(".maps");

    SEC("tp/sched/sched_process_exec")
    int handle_exec(struct trace_event_raw_sched_process_exec *ctx)
    {
      ....
      e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
      ....
      bpf_ringbuf_submit(e, 0);
      return 0;
    }
    ```

    Compile and Run the program:

    ```console
    $ ecc bootstrap.bpf.c bootstrap.h
    Compiling bpf object...
    Generating export types...
    Packing ebpf object and config into package.json...
    $ sudo ./ecli examples/bpftools/bootstrap/package.json
    TIME     PID     PPID    EXIT_CODE  DURATION_NS  COMM    FILENAME  EXIT_EVENT  
    22:01:04  46310  2915    0          0            sh      /bin/sh   0
    22:01:04  46311  46310   0          0            which   /usr/bin/which 0
    22:01:04  46311  46310   0          2823776      which             1
    22:01:04  46310  2915    0          6288891      sh                1
    22:01:04  46312  2915    0          0            sh      /bin/sh   0
    22:01:04  46313  46312   0          0            ps      /usr/bin/ps 0
    ```

    see [bootstrap](examples/bpftools/bootstrap/bootstrap.bpf.c) for example. This is exactly the same as [bootstrap.bpf.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/bootstrap.bpf.c) in [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) project, but only kernel code is needed.

#### Automatically sample the data and print `hists` in userspace

- Sample the data from hash maps and print them in human readable format with comments:

    ```c
    /// @sample {"interval": 1000, "type" : "log2_hist"}
    struct {
        __uint(type, BPF_MAP_TYPE_HASH);
        __uint(max_entries, MAX_ENTRIES);
        __type(key, u32);
        __type(value, struct hist);
    } hists SEC(".maps");
    ```

    and Get `hist` data from `hists` map and print them in human readable format:

    ```bash
    $ sudo ecli  examples/bpftools/runqlat/package.json --targ_per_process
    key = 8326
    comm = containerd

          usec              : count    distribution
            0 -> 1          : 0        |                                        |
            2 -> 3          : 0        |                                        |
            4 -> 7          : 0        |                                        |
            8 -> 15         : 0        |                                        |
           16 -> 31         : 2        |*************                           |
           32 -> 63         : 2        |*************                           |
           64 -> 127        : 6        |****************************************|
          128 -> 255        : 0        |                                        |
          256 -> 511        : 2        |*************                           |
    ```

    see [examples/bpftools/mdflush.bpf.c](examples/bpftools/runqlat/runqlat.bpf.c) for example.

#### Automatically generate and config command line arguments

- Automatically generate and config command line arguments for your eBPF program from the comments in your kernel code:

    ```c
    /// Process ID to trace
    const volatile pid_t pid_target = 0;
    /// Thread ID to trace
    const volatile pid_t tgid_target = 0;
    /// @description User ID to trace
    const volatile uid_t uid_target = 0;
    /// @cmdarg {"default": false, "short": "f", "long": "failed"}
    /// @description target pid to trace
    const volatile bool targ_failed = false;
    ```

    and Get:

    ```console
    $ sudo ecli  examples/bpftools/opensnoop/package.json -h
    Usage: opensnoop_bpf [--help] [--version] [--verbose] [--pid_target VAR] [--tgid_target VAR] [--uid_target VAR] [--failed]

    Trace open family syscalls.

    Optional arguments:
      -h, --help    shows help message and exits 
      -v, --version prints version information and exits 
      --verbose     prints libbpf debug information 
      --pid_target  Process ID to trace 
      --tgid_target Thread ID to trace
    ```

    see [examples/bpftools/opensnoop/opensnoop.bpf.c](examples/bpftools/opensnoop/opensnoop.bpf.c) for example.

- `100%` compatible with `libbpf`, [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap/tree/master/examples/c) and `libbpf-rs`, etc: you can compile [libbpf-tools](https://github.com/iovisor/bcc/blob/master/libbpf-tools) kernel code with `eunomia-bpf` and run them without many modification!
- Not limited to tracing: support `tracepoints`, `kprobe`, `uprobe`, `lsm`, `xdp`, `tc` etc...

### Compile and pack CO-RE eBPF kernel code to a config file

- Compile and pack CO-RE eBPF kernel code to a `JSON` or `YAML` config file:

    ```console
    $ ecc cmd/test/opensnoop.bpf.c opensnoop.h
    Compiling bpf object...
    Generating export types...
    Packing ebpf object and config into package.json...
    $ docker run -it -v `pwd`/:/src/ ghcr.io/eunomia-bpf/ecc-`uname -m`:latest # build with docker for x86_64 and aarch64
    Packing ebpf object and config into package.json...
    ```

    You can modify the config file and config the eBPF program behavior to your need.

### Dynamic load and run CO-RE eBPF kernel code from the cloud with URL or OCI image

- you can dynamically load it on different kernel version without recompile, and without clang/llvm dependency:

    ```console
    $ sudo ecli opensnoop.json
    TIME     PID     TPID    SIG     RET     COMM    
    22:58:28  77121  3168    0       0       cpptools-srv
    22:58:29  69044  3168    0       0       cpptools-srv
    22:58:29  3014   2906    0       0       code
    22:58:29  6952   4061    0       0       node
    22:58:29  4061   3937    0       0       node
    22:58:29  75263  3168    0       0       cpptools-srv
    22:58:29  2906   2488    0       0       code
    22:58:29  69149  3168    0       0       cpptools-srv
    22:58:29  73541  3168    0       0       cpptools-srv
    22:58:29  73468  3168    0       0       cpptools-srv
    22:58:29  2906   2488    0       0       code
    22:58:29  69094  3168    0       0       cpptools-srv
    ```

- Get pre-compiled eBPF programs running from the cloud to the kernel in `1` line of bash, kernel version and architecture independent:

    ```bash
    # download the release from https://github.com/eunomia-bpf/eunomia-bpf/releases/latest/download/ecli
    $ wget https://aka.pw/bpf-ecli -O ecli && chmod +x ./ecli
    $ sudo ./ecli https://eunomia-bpf.github.io/eunomia-bpf/sigsnoop/package.json # simply run a pre-compiled ebpf code from a url
    $ sudo ./ecli sigsnoop:latest # run with a name and download the latest version bpf tool from our repo
    ```

- very small and simple! The library itself `<1MB` and no `LLVM/Clang` dependence, can be embedded easily in you project
- as fast as `<100ms` and little resource need to dynamically load and run eBPF program

Base on `eunomia-bpf`, we have an eBPF pacakge manager in [LMP](https://github.com/linuxkerneltravel/lmp) project, with OCI images and [ORAS](https://github.com/oras-project/oras) for distribution.

### Write user space code for your eBPF program in WebAssembly

You can write and compile user space code for your eBPF program in WebAssembly, and run it with `eunomia-bpf`:

```bash
$ sudo ./ecli run app.wasm -h
Usage: sigsnoop [-h] [-x] [-k] [-n] [-p PID] [-s SIGNAL]
Trace standard and real-time signals.


    -h, --help  show this help message and exit
    -x, --failed  failed signals only
    -k, --killed  kill only
    -p, --pid=<int>  target pid
    -s, --signal=<int>  target signal
```

See [sigsnoop](examples/bpftools/sigsnoop/app.c), we port the user space code of `sigsnoop` tool from [BCC](https://github.com/iovisor/bcc/blob/master/libbpf-tools/sigsnoop.c) to our project as an example.

You can operate the eBPF kernel program or process the data in user space `WASM` runtime.

Powered by WASM, an eBPF program may be able to:

- have isolation and protection for operating system resources, both user-space and kernel-space
- safely execute user-defined or community-contributed eBPF code as plug-ins in a software product
- Write eBPF programs with the language you favor, distribute and run the programs on another kernel or arch.

We have tested on `x86` and `arm` platform, more Architecture tests will be added soon.

## Project Architecture

we have a loader library, a compile toolchain, and some additional tools like cli and a custom metrics exporter.

![eunomia-arch.png](documents/images/eunomia-arch.png)

### An bpf-loader library

A wrapper of main functions of libbpf, provide the ability to dynamically load eBPF code to the kernel and run it with a simple JSON and a few API.

see [bpf-loader](eunomia-bpf) for details.

A [simple cli interface](ecli) is provided for bpf-loader library, which you can use it to start any eBPF program from a url in a command. You can download it from [release](https://github.com/eunomia-bpf/eunomia-bpf/releases/).

see [examples](examples) for more examples.

### A library to load and operate eBPF program from a WASM module

Use the `eunomia-bpf` library to load `eBPF` program from a `WASM` module, you can write a WASM module to operate the eBPF program or process the data in user space `WASM` runtime. The idea is simple:

1. compile the kernel eBPF code skeleton to the `JSON` format with `eunomia-cc` toolchain
2. embed the `JSON` data in the `WASM` module, and provide some API for operating the eBPF program skeleton
3. load the `JSON` data from the `WASM` module and run the eBPF program skeleton with `eunomia-bpf` library

You can have multiple `eBPF` program in a single `WASM` module.

See [ewasm](wasm-runtime) for details. In fact, `ewasm` library only exports a few functions from `bpf-loader` library to the `VM`, so you can replace the `WASM` runtime with your own easily.

For example, you can run an eBPF program with a WASM module for an URL:

```bash
sudo ./ecli run https://eunomia-bpf.github.io/eunomia-bpf/sigsnoop/app.wasm
```

You can also generate a WASM program template for eBPF or build WASM module with `compiler` container:

```shell
# for x86_64 and aarch64
docker run -it -v `pwd`/:/src/ ghcr.io/eunomia-bpf/ecc-`uname -m`:latest gen-wasm-skel # generate WASM app template for eBPF
docker run -it -v `pwd`/:/src/ ghcr.io/eunomia-bpf/ecc-`uname -m`:latest build-wasm    # Build WASM module
```

see [sigsnoop example](examples/bpftools/sigsnoop) for more detail.

### A compile toolchain to help you generate pre compiled eBPF data

The toolchain can be used as a docker to generate pre-compiled eBPF data in one command:

see the compile toolchains [compiler](compiler) for details.

you can also simply use the [ebpm-template](https://github.com/eunomia-bpf/ebpm-template) repo as a template in github, just push to it and github action can help you compile CO-RE ebpf code!

### other related projects

- LMP eBPF Hub: [github.com/linuxkerneltravel/lmp](https://github.com/linuxkerneltravel/lmp)

    > a package manager for eBPF based on wasm modules

- bolipi online compiler & runner: [https://bolipi.com/ebpf/home/online](https://bolipi.com/ebpf/home/online)

    > an online compiler and runner for eBPF program newbies

- An Observability tool

    > An prometheus and OpenTelemetry exporter for custom eBPF metrics, written in async rust: [eunomia-exporter](eunomia-exporter). You can compile it or download from [release](https://github.com/eunomia-bpf/eunomia-bpf/releases/)

## build the project

see [build](documents/build.md) for details.

## examples

see [examples](examples) for details about eBPF tools and library usage.

We also have a prove of concept video: [Writing eBPF programs in WASM](https://www.bilibili.com/video/BV1JN4y1A76k/).

## benchmark

see [benchmark](documents/benchmark.md) for details.

## Road-map

- [ ] Rust SDK in Wasm
- [ ] Go SDK in Wasm
- [ ] Python SDK with BCC
- [ ] `riscv` and other arch support
- [ ] Android support

## License

MIT LICENSE
