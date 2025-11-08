# Maintainer: Johannes Löthberg <johannes@kyriasis.com>
# Maintainer: Jan Alexander Steffens (heftig) <heftig@archlinux.org>
# Contributor: Alexander F Rødseth <xyproto@archlinux.org>
# Contributor: Daniel Micay <danielmicay@gmail.com>
# Contributor: userwithuid <userwithuid@gmail.com>

# ALARM: Kevin Mihelich <kevin@archlinuxarm.org>
#  - remove lib32, musl, and wasm packages and related bits
#  - remove wasm-component-ld from makedepends
#  - add a link to g++ to compensate for broken cross-compiler decisions
#  - build v6/v7 with -j2 - RAM constraints
#  - set llvm-config in config.toml for ARM architectures
#  - set debuginfo-level = 0 in config.toml - RAM constraints
#  - build aarch64 with 16k page support

highmem=1

pkgbase=rust
pkgname=(
  rust
  rust-src
)
pkgver=1.91.0
pkgrel=1
epoch=1
pkgdesc="Systems programming language focused on safety, speed and concurrency"
url=https://www.rust-lang.org/
arch=(x86_64)
license=("Apache-2.0 OR MIT")
options=(
  !emptydirs
  !lto
)
depends=(
  bash
  compiler-rt
  curl
  gcc
  gcc-libs
  glibc
  libssh2
  lld
  llvm-libs
  openssl
  zlib
)
makedepends=(
  clang
  cmake
  libffi
  llvm
  musl
  ninja
  perl
  python
  git
  rust
  rustup
)
checkdepends=(
  gdb
  procps-ng
)
source=(
  "https://static.rust-lang.org/dist/rustc-$pkgver-src.tar.gz"

  # Patch bootstrap so that rust-analyzer-proc-macro-srv
  # is in /usr/lib instead of /usr/libexec
  0001-bootstrap-Change-libexec-dir.patch

  # Put bash completions where they belong
  0002-bootstrap-Change-bash-completion-dir.patch

  # Fix build with system rustc
  # https://github.com/rust-lang/rust/issues/143735
  0003-bootstrap-Workaround-for-system-stage0.patch

  # Use our *-pc-linux-gnu targets, making LTO with clang simpler
  0004-compiler-Change-LLVM-targets.patch

  # Prefer "lib" over "lib64"
  0006-compiler-Swap-primary-and-secondary-lib-dirs.patch
)
b2sums=('abd675fda0e147735f8f665059826d2e5e2fd334d3b60c10197838553a91bb04c9df1f84062232d4fe71400443cd9ca6e233faf7bd18abd19776d63d6aa388a9'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP')

# Make sure the duplication in rust-wasm is found
COMPRESSZST+=(--long)

prepare() {
  cd rustc-$pkgver-src

  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    src="${src%.zst}"
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  local clangdir
  clangdir="$(clang -print-resource-dir)"

  cat >bootstrap.toml <<END
# see src/bootstrap/defaults/
profile = "dist"

# see src/bootstrap/src/utils/change_tracker.rs
change-id = 146435


[llvm]
download-ci-llvm = false
link-shared = true

[build]
description = "Arch Linux $pkgbase $epoch:$pkgver-$pkgrel"
cargo = "/usr/bin/cargo"
rustc = "/usr/bin/rustc"
rustfmt = "/usr/bin/rustfmt"
locked-deps = true
vendor = true
tools = [
  "cargo",
  "clippy",
  "rustdoc",
  "rustfmt",
  "rust-analyzer-proc-macro-srv",
  "analysis",
  "src",
]
sanitizers = false
profiler = true

# Generating docs fails with the wasm32-* targets
docs = false

[install]
prefix = "/usr"

[rust]
codegen-units = 16
codegen-units-std = 1
debuginfo-level = 0
debuginfo-level-std = 0
channel = "stable"
rpath = false
lld = false
use-lld = false
llvm-bitcode-linker = false
deny-warnings = false
backtrace-on-ice = true

[dist]
compression-formats = ["gz"]
compression-profile = "fast"

[target.x86_64-unknown-linux-gnu]
cc = "/usr/bin/gcc"
cxx = "/usr/bin/g++"
ar = "/usr/bin/gcc-ar"
ranlib = "/usr/bin/gcc-ranlib"
llvm-config = "/usr/bin/llvm-config"
optimized-compiler-builtins = "$clangdir/lib/linux/libclang_rt.builtins-x86_64.a"

[target.i686-unknown-linux-gnu]
cc = "/usr/bin/gcc"
cxx = "/usr/bin/g++"
ar = "/usr/bin/gcc-ar"
ranlib = "/usr/bin/gcc-ranlib"
optimized-compiler-builtins = "$clangdir/lib/linux/libclang_rt.builtins-i386.a"

[target.aarch64-unknown-linux-gnu]
cc = "/usr/bin/gcc"
cxx = "/usr/bin/g++"
ar = "/usr/bin/gcc-ar"
ranlib = "/usr/bin/gcc-ranlib"
llvm-config = "/usr/bin/llvm-config"

[target.armv7-unknown-linux-gnueabihf]
cc = "/usr/bin/gcc"
cxx = "/usr/bin/g++"
ar = "/usr/bin/gcc-ar"
ranlib = "/usr/bin/gcc-ranlib"
llvm-config = "/usr/bin/llvm-config"

[target.arm-unknown-linux-gnueabihf]
cc = "/usr/bin/gcc"
cxx = "/usr/bin/g++"
ar = "/usr/bin/gcc-ar"
ranlib = "/usr/bin/gcc-ranlib"
llvm-config = "/usr/bin/llvm-config"
END

  if [[ $CARCH == armv7h ]]; then
    mkdir path
    ln -s /usr/bin/g++ path/arm-linux-gnueabihf-g++
    export PATH="$srcdir/path:$PATH"
    jobs="2"
  else
    jobs="$(nproc)"
  fi

  rustup toolchain install $pkgver
  rustup default $pkgver
}

_pick() {
  local p="$1" f d; shift
  for f; do
    d="$srcdir/$p/${f#$pkgdir/}"
    mkdir -p "$(dirname "$d")"
    mv "$f" "$d"
    rmdir -p --ignore-fail-on-non-empty "$(dirname "$f")"
  done
}

build() {
  cd rustc-$pkgver-src

  [[ $CARCH == "aarch64" ]] && export JEMALLOC_SYS_WITH_LG_PAGE=16
  export RUST_BACKTRACE=1
  unset CFLAGS CXXFLAGS LDFLAGS
  git config --global --add safe.directory '*'
  DESTDIR="$srcdir/dest-rust" python ./x.py install -j $jobs

  cd ../dest-rust

  # delete unnecessary files, e.g. files only used for the uninstall script
  rm -v etc/target-spec-json-schema.json
  rm -v usr/lib/rustlib/{components,install.log,rust-installer-version,uninstall.sh}
  rm -v usr/lib/rustlib/manifest-*

  # licenses for main rust package
  local ldir="usr/share/licenses/rust" f d
  mkdir -p "$ldir"
  for f in usr/share/doc/*/{COPYRIGHT,LICENSE}*; do
    d="$(dirname "$f")"
    case $f in
      */LICENSE-APACHE) rm -v "$f" ;;
      *) mv -v "$f" "$ldir/${f##*/}.${d##*/}" ;;
    esac
    rmdir -p --ignore-fail-on-non-empty "$d"
  done

  # rustbuild always installs copies of the shared libraries to /usr/lib,
  # overwrite them with symlinks to the per-architecture versions
  #mkdir -pv usr/lib32
  #ln -srvft usr/lib   usr/lib/rustlib/x86_64-unknown-linux-gnu/lib/*.so
  #ln -srvft usr/lib32 usr/lib/rustlib/i686-unknown-linux-gnu/lib/*.so

  #_pick dest-i686 usr/lib/rustlib/i686-unknown-linux-gnu usr/lib32
  #_pick dest-musl usr/lib/rustlib/x86_64-unknown-linux-musl
  #_pick dest-wasm usr/lib/rustlib/wasm32-*
  _pick dest-src  usr/lib/rustlib/src
}

package_rust() {
  optdepends=(
    'gdb: rust-gdb script'
    'lldb: rust-lldb script'
  )
  provides=(
    cargo
    rustfmt
  )
  conflicts=(
    cargo
    'rust-docs<1:1.56.1-3'
    rustfmt
  )
  replaces=(
    cargo
    cargo-tree
    'rust-docs<1:1.56.1-3'
    rustfmt
  )

  cp -a dest-rust/* "$pkgdir"
}

package_rust-src() {
  pkgdesc="Source code for the Rust standard library"
  depends=(rust)

  cp -a dest-src/* "$pkgdir"

  install -Dt "$pkgdir/usr/share/licenses/$pkgname" -m644 \
    rustc-$pkgver-src/{COPYRIGHT,LICENSE-MIT}
}

# vim:set ts=2 sw=2 et:
