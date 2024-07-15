# Maintainer: Daniele Basso <d dot bass 05 at proton dot me>
pkgname=bun
pkgver=1.1.20
pkgrel=1
pkgdesc="Bun is a fast JavaScript all-in-one toolkit. This PKGBUILD builds from source, resulting into a smaller and faster binary depending on your CPU."
arch=(x86_64)
url="https://github.com/oven-sh/bun"
license=('GPL')
depends=(c-ares libarchive libuv mimalloc tcc zlib zstd)
makedepends=(
	clang cmake esbuild git go icu libiconv libtool lld llvm ninja pkg-config python ruby rust unzip zig
)
conflicts=(bun-bin)
source=(git+$url.git#tag=bun-v$pkgver
        bun-linux-x64-$pkgver.zip::https://github.com/oven-sh/bun/releases/download/bun-v$pkgver/bun-linux-x64.zip) # add "baseline" here to download the avx2-less build of bun!
b2sums=('983a199a6c2d3882f4da9652b922c90055d8bea0d58e1fe6695ef8cef4d786142b9882fb6b483b987e685692d2162d102987340208fe4d484d46e8ab89e07fe1'
        'ac42166a79191cf58869c72f98fdda6b7269ed25e79bd4262abb39da08277ccfe556815ebd61aefee1fa6d41871e56cb1921444ad13d26ceec854b167321aa12')

_j=6 #change for your system

prepare() {
  cd bun

  git submodule update --init --recursive --progress --depth=1 --checkout src/bun.js/WebKit src/deps/picohttpparser src/deps/boringssl src/deps/lol-html src/deps/ls-hpack

  bash ./scripts/set-webkit-submodule-to-cmake.sh
}

build() {
  export PATH="${srcdir}/bun-linux-x64:$PATH"

  cd bun

  mkdir -p ./build

  bun i

  bash ./scripts/build-lolhtml.sh
  bash ./scripts/build-lshpack.sh
  bash ./scripts/build-boringssl.sh

  ln -sf /usr/lib/libtcc.so ./build/bun-deps/libtcc.a

  make runtime_js fallback_decoder bun_error node-fallbacks

  cd src/bun.js/WebKit/

  # Adapted from https://github.com/oven-sh/WebKit/blob/main/Dockerfile#L84

  COMMON_FLAGS="-mno-omit-leaf-frame-pointer -fno-omit-frame-pointer -ffunction-sections -fdata-sections"
  CC="/usr/bin/clang" CXX="/usr/bin/clang++" CFLAGS="${DEFAULT_CFLAGS} $CFLAGS" CXXFLAGS="${DEFAULT_CFLAGS} $CXXFLAGS" cmake \
      -S . \
      -B ./build \
      -Wno-dev \
      -DPORT="JSCOnly" \
      -DENABLE_STATIC_JSC=ON \
      -DENABLE_BUN_SKIP_FAILING_ASSERTIONS=ON \
      -DCMAKE_BUILD_TYPE=Release \
      -DUSE_THIN_ARCHIVES=OFF \
      -DUSE_BUN_JSC_ADDITIONS=ON \
      -DENABLE_FTL_JIT=ON \
      -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
      -DALLOW_LINE_AND_COLUMN_NUMBER_IN_BUILTINS=ON \
      -DENABLE_SINGLE_THREADED_VM_ENTRY_SCOPE=ON \
      -DENABLE_REMOTE_INSPECTOR=ON \
      -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" \
      -DCMAKE_AR="/usr/bin/llvm-ar" \
      -DCMAKE_RANLIB="/usr/bin/llvm-ranlib" \
      -GNinja

  ninja -C ./build jsc -j$_j

  mkdir -p ./output/{lib,include/JavaScriptCore,Source/JavaScriptCore}

  cp -r ./build/lib/*.a ./output/lib
  cp ./build/*.h ./output/include
  find ./build/JavaScriptCore/Headers/JavaScriptCore/ -name "*.h" -exec cp {} ./output/include/JavaScriptCore/ \;
  find ./build/JavaScriptCore/PrivateHeaders/JavaScriptCore/ -name "*.h" -exec cp {} ./output/include/JavaScriptCore/ \;
  cp -r ./build/WTF/Headers/wtf/ ./output/include
  cp -r ./build/bmalloc/Headers/bmalloc/ ./output/include
  cp -r ./Source/JavaScriptCore/Scripts ./output/Source/JavaScriptCore
  cp ./Source/JavaScriptCore/create_hash_table ./output/Source/JavaScriptCore
  rm -rf ./output/include/unicode
  cp -r /usr/include/unicode ./output/include/unicode

  ln -sf /lib/libicudata.so ./output/lib/libicudata.a
  ln -sf /lib/libicui18n.so ./output/lib/libicui18n.a
  ln -sf /lib/libicuuc.so ./output/lib/libicuuc.a

  cd $srcdir

  CXXFLAGS="-Wno-unused-result -Wno-error=format-truncation ${CXXFLAGS}" cmake -B $srcdir/build -S $srcdir/bun -Wno-dev -DCMAKE_BUILD_TYPE=Release -GNinja -DUSE_STATIC_LIBATOMIC=OFF -DUSE_SYSTEM_ICU=OFF \
        -DLLVM_PREFIX=/usr -DWEBKIT_DIR=$srcdir/bun/src/bun.js/WebKit/output -DUSE_DEBUG_JSC=OFF -DUSE_LTO=ON -DZIG_COMPILER=system -DCPU_TARGET=native -DZIG_LIB_DIR=/usr/lib/zig/ \
        -DUSE_CUSTOM_ZSTD=OFF -DUSE_CUSTOM_ZLIB=OFF -DUSE_CUSTOM_LIBARCHIVE=OFF -DUSE_CUSTOM_CARES=OFF -DUSE_CUSTOM_MIMALLOC=OFF -DUSE_UNIFIED_SOURCES=ON
  ninja -C ./build -j$_j
}

package() {
  install -Dm755 $srcdir/build/bun $pkgdir/usr/bin/bun
  ln -s /usr/bin/bun $pkgdir/usr/bin/bunx

  SHELL=zsh $pkgdir/usr/bin/bun completions > bun.zsh
  SHELL=bash $pkgdir/usr/bin/bun completions > bun.bash
  SHELL=fish $pkgdir/usr/bin/bun completions > bun.fish

  install -Dm644 bun.zsh $pkgdir/usr/share/zsh/site-functions/_bun
  install -Dm644 bun.bash $pkgdir/usr/share/bash-completion/completions/bun
  install -Dm644 bun.fish $pkgdir/usr/share/fish/vendor_completions.d/bun.fish
}
