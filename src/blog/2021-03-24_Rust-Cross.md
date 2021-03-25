# Painless Rust Cross-Compilation

### Rust's `cross`
Rust is big on developer productivity: The tooling provided to get started with Rust development _just works_, be it
`cargo`, `rustup`, or `clippy`. Another tool is in the toolbox is [cross](https://github.com/rust-embedded/cross), which
lets you "magically" run tests or compile for other [build
targets](https://rust-lang.github.io/rustup-components-history/). Under the hood, cross provides a set of docker
containers (one for each supported target) containing a properly set up cross compilation toolchain and execution
environment; for example for
[`arm-unknown-linux-gnueabihf`](https://github.com/rust-embedded/cross/blob/master/docker/Dockerfile.arm-unknown-linux-gnueabihf),
this includes the `arm-linux-gnueabihf-gcc` toolchain and `qemu-arm` for executing arm binaries.

Obviously, it would be a pain to set up each of those toolchains correctly on your host, and very hard to ensure build
reproducibility when collaborating.

### `docker buildx`
In another ecosystem, `docker` is pushing its refactoring of the `docker build` subsystem
[`moby/buildkit`](https://github.com/moby/buildkit) slowly to become the standard of recent `docker` versions. One of
the notable additions is the additional of the `--platform` arg to `docker buildx`. Images can be created for multiple
architectures like so[^0]:
```shell
docker buildx --platform linux/amd64,linux/aarch64,windows/amd64 .
```
This generates a so-called [multi-arch image](https://www.docker.com/blog/multi-arch-build-and-images-the-simple-way/),
where the image's manifest contains references to multiple images depending on the chosen architecture. Within the
`Dockerfile`, the following arguments are exposed: `BUILDPLATFORM` and `TARGETPLATFORM`.

### Bringing it together
Given a Cargo project, I'd like to cross-compile to multiple targets with a single call to `docker buildx`:
```shell
docker buildx build --platform linux/amd64,linux/aarch64,windows/amd64 -o dist .
```
And this should yield:
```shell
~ tree dist
dist
├── linux_amd64
│   ├── my_awesome_binary
├── linux_arm64
│   ├── my_awesome_binary
└── windows_amd64
    └── my_awesome_binary.exe
```

.. and here's the `Dockerfile` for this, with inlined comments:
```Dockerfile
# Usage from crate root:
#   docker buildx build \
#     --platform linux/amd64,linux/aarch64 \
#     -f Dockerfile \
#     -o dist .
# This will put the resulting artefacts inside `./dist`.
ARG CROSSVER=0.2.1

# Poor man's mapping from buildx arch scheme to cargo
FROM --platform=$BUILDPLATFORM rustembedded/cross:aarch64-unknown-linux-musl-${CROSSVER} AS build-linux-arm64
ENV CARGO_BUILD_TARGET=aarch64-unknown-linux-musl
FROM --platform=$BUILDPLATFORM rustembedded/cross:x86_64-unknown-linux-musl-${CROSSVER} AS build-linux-amd64
ENV CARGO_BUILD_TARGET=x86_64-unknown-linux-musl
FROM --platform=$BUILDPLATFORM rustembedded/cross:arm-unknown-linux-musleabi-${CROSSVER} AS build-linux-armv6
ENV CARGO_BUILD_TARGET=arm-unknown-linux-musleabi
FROM --platform=$BUILDPLATFORM rustembedded/cross:armv7-unknown-linux-musleabihf-${CROSSVER} AS build-linux-armv7
ENV CARGO_BUILD_TARGET=armv7-unknown-linux-musleabihf
FROM --platform=$BUILDPLATFORM rustembedded/cross:x86_64-pc-windows-gnu-${CROSSVER} AS build-windows-amd64
ENV CARGO_BUILD_TARGET=x86_64-pc-windows-gnu

# actual build image
FROM --platform=$BUILDPLATFORM build-${TARGETOS}-${TARGETARCH}${TARGETVARIANT} AS crossbuild
ENV     CARGO_HOME=/usr/local/cargo \
        PATH=/usr/local/cargo/bin:$PATH
ARG RUSTVER=1.51.0

RUN curl https://sh.rustup.rs -sSf | sh -s -- \
    --default-toolchain ${RUSTVER} \
    --profile minimal \
    --target ${CARGO_BUILD_TARGET} \
    -y 

# actual building
FROM --platform=$BUILDPLATFORM crossbuild AS build
ENV CARGO_BUILD_JOBS=8
ARG TARGETPLATFORM
ARG BUILDPLATFORM
RUN echo "Building on $BUILDPLATFORM for $TARGETPLATFORM"
ARG TARGET_BINARY

WORKDIR /src
COPY . .

RUN cargo build --release

# move created binary to a well known place, as there is no access to the
# CARGO_BUILD_TARGET env var in the later extraction stage
# .. would be great to have `cargo build --out-dir` already on stable ..
RUN mkdir /out && \
    mv /src/target/$CARGO_BUILD_TARGET/release/* /out

# create an empty image just with the built binary to export
FROM scratch

COPY --from=build /out/* .
```

Compared to other approaches, where one would use a shared cache for Cargo's intermediate artefacts (or sccache), I
found this approach to be surprisingly well suited for CI setups, as building the image is parallelized and heavily
cached by the Docker daemon, and quite maintainable, as there's only one build definition involved for all build
targets.

----

[^0]: Yes, you read that right, even for Windows.
