# Maintainer: Daniele Basso <d dot bass 05 at proton dot me>
pkgname=bun
pkgver=1.2.1
_webkitver=9e3b60e4a6438d20ee6f8aa5bec6b71d2b7d213f #https://github.com/oven-sh/bun/blob/main/cmake/tools/SetupWebKit.cmake#L5
pkgrel=1
pkgdesc="Bun is a fast JavaScript all-in-one toolkit. This PKGBUILD builds from source, resulting into a smaller and faster binary depending on your CPU."
arch=(x86_64)
url="https://github.com/oven-sh/bun"
license=('GPL')
#depends=(c-ares libarchive libuv mimalloc tcc zlib zstd)
makedepends=(
	ccache clang18 cmake git go icu libdeflate libiconv libtool lld18 llvm18 mold ninja pkg-config python ruby rust unzip
)
conflicts=(bun-bin)
source=(git+$url.git#tag=bun-v$pkgver
        bun-linux-x64-$pkgver.zip::https://github.com/oven-sh/bun/releases/download/bun-v$pkgver/bun-linux-x64.zip # add "baseline" here to download the avx2-less build of bun!
        brotliFlag.patch)
b2sums=('b196a9e7ed186a4476ff6485083600e0a00b09799f321106c38d98e8320b0ccf0480c7ec2176f89f6566ab05cf06a345fdad08bc1e48434d2600f07c916bfb9d'
        '2e7c45784ac06313713cc009d16b071f86f58c52c859202c2ba974fda7d4d0e4efa92df74ae7d412ea49750ff81d357d99245672a215726b3ccae1a6395f9327'
        'ba86bf7d8ff3c6b0aa1b26a2eaf7d0ca480ff42fde59b75f3290de3f197a07ec8fd926c96287436e29d5dedb9632ffe9e1f8d44ebfa7f9df804874bc889afc2d')
options=(ccache lto)

_j=16 #change for your system

prepare() {
  export PATH="${srcdir}/bun-linux-x64:$PATH"

  # rm -rf WebKit
  if ! [[ -d WebKit ]]; then
      git clone --depth=1 https://github.com/oven-sh/WebKit.git -b autobuild-$_webkitver
  fi

  cd bun

  # mkdir -p ./vendor
  # ln -sf $srcdir/WebKit ./vendor/WebKit

  patch -Np1 -i ../brotliFlag.patch

  bun i
}

build() {
  export PATH="$/usr/lib/llvm18/bin/:$PATH"
  mkdir -p ./build

  build_webkit

  CXXFLAGS="-Wno-unused-result ${CXXFLAGS}" cmake -GNinja -B $srcdir/build -S $srcdir/bun -Wno-dev -DCMAKE_BUILD_TYPE=Release -DUSE_STATIC_LIBATOMIC=OFF -DUSE_WEBKIT_ICU=ON \
        -DENABLE_CCACHE=ON -DLLVM_VERSION=18.1.8 -DENABLE_LTO=ON -DWEBKIT_LOCAL=ON -DWEBKIT_PATH=$srcdir/WebKit/WebKitBuild/Release/output \
        -DCMAKE_C_COMPILER=/usr/lib/llvm18/bin/clang -DCMAKE_CXX_COMPILER=/usr/lib/llvm18/bin/clang++

  ninja -C ./build -j$_j
}

build_webkit(){

  pushd $srcdir/WebKit/

#   cd $srcdir/bun
# 
#   WEBKIT_DIR=$srcdir/WebKit make jsc-copy-headers
# 
#   cd ..

  mkdir -p WebKitBuild/Release

  # Adapted from https://github.com/oven-sh/WebKit/blob/main/Dockerfile#L109

  export DEFAULT_CFLAGS="-mno-omit-leaf-frame-pointer -g -fno-omit-frame-pointer -ffunction-sections -fdata-sections -faddrsig -fno-unwind-tables -fno-asynchronous-unwind-tables -DU_STATIC_IMPLEMENTATION=1 "
  export LTO_FLAG="-flto=full -fwhole-program-vtables -fforce-emit-vtables "

  export CFLAGS="${DEFAULT_CFLAGS} $CFLAGS $LTO_FLAG "
  export CXXFLAGS="${DEFAULT_CFLAGS} $CXXFLAGS $LTO_FLAG -fno-c++-static-destructors "
  export LDFLAGS="-fuse-ld=lld $LDFLAGS "

  CC="/usr/bin/clang" CXX="/usr/bin/clang++" cmake \
      -S . \
      -B ./WebKitBuild/Release \
      -Wno-dev \
      -DPORT="JSCOnly" \
      -DENABLE_STATIC_JSC=ON \
      -DENABLE_BUN_SKIP_FAILING_ASSERTIONS=ON \
      -DCMAKE_BUILD_TYPE=Release \
      -DUSE_THIN_ARCHIVES=OFF \
      -DUSE_BUN_JSC_ADDITIONS=ON \
      -DUSE_BUN_EVENT_LOOP=OFF \
      -DENABLE_FTL_JIT=ON \
      -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
      -DALLOW_LINE_AND_COLUMN_NUMBER_IN_BUILTINS=ON \
      -DENABLE_REMOTE_INSPECTOR=ON \
      -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" \
      -DCMAKE_AR="/usr/bin/llvm-ar" \
      -DCMAKE_RANLIB="/usr/bin/llvm-ranlib" \
      -GNinja


  cd ./WebKitBuild/Release
  ninja jsc -j$_j

  mkdir -p ./output/{lib,include/JavaScriptCore,Source/JavaScriptCore}

  cp -r ./lib/*.a ./output/lib
  cp ./*.h ./output/include
  cp -r ./bin ./output/bin
  cp ./*.json ./output

  find ./JavaScriptCore/DerivedSources/ -name "*.h" -exec sh -c 'cp "$1" "./output/include/JavaScriptCore/$(basename "$1")"' sh {} \;
  find ./JavaScriptCore/DerivedSources/ -name "*.json" -exec sh -c 'cp "$1" "./output/$(basename "$1")"' sh {} \;
  find ./JavaScriptCore/Headers/JavaScriptCore/ -name "*.h" -exec cp {} ./output/include/JavaScriptCore/ \;
  find ./JavaScriptCore/PrivateHeaders/JavaScriptCore/ -name "*.h" -exec cp {} ./output/include/JavaScriptCore/ \;
  cp -r ./WTF/Headers/wtf/ ./output/include
  cp -r ./bmalloc/Headers/bmalloc/ ./output/include
  mkdir -p ./output/Source/JavaScriptCore
  cp -r ../../Source/JavaScriptCore/Scripts ./output/Source/JavaScriptCore
  cp ../../Source/JavaScriptCore/create_hash_table ./output/Source/JavaScriptCore

  ln -sf /lib/libicudata.so ./lib/libicudata.a
  ln -sf /lib/libicui18n.so ./lib/libicui18n.a
  ln -sf /lib/libicuuc.so ./lib/libicuuc.a

  popd
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
