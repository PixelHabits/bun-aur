# Maintainer: Daniele Basso <d dot bass 05 at proton dot me>
pkgname=bun
pkgver=1.1.42
_webkitver=3845bf370ff4e9a5c0b96036255142c7904be963 #https://github.com/oven-sh/bun/blob/main/cmake/tools/SetupWebKit.cmake#L5
pkgrel=1
pkgdesc="Bun is a fast JavaScript all-in-one toolkit. This PKGBUILD builds from source, resulting into a smaller and faster binary depending on your CPU."
arch=(x86_64)
url="https://github.com/oven-sh/bun"
license=('GPL')
#depends=(c-ares libarchive libuv mimalloc tcc zlib zstd)
makedepends=(
	ccache clang cmake git go icu libdeflate libiconv libtool lld llvm mold ninja pkg-config python ruby rust unzip
)
conflicts=(bun-bin)
source=(git+$url.git#tag=bun-v$pkgver
        bun-linux-x64-$pkgver.zip::https://github.com/oven-sh/bun/releases/download/bun-v$pkgver/bun-linux-x64.zip) # add "baseline" here to download the avx2-less build of bun!
        # git+https://github.com/oven-sh/WebKit.git#commit=$_webkitver)
b2sums=('93b3b52f1f5283dc1babd7634e2a7a728c718740d7d4b19b78df6ae051b46aab62d52d81adbbc00f5aa89bc2de4d478dae0ac3a9ab179ffbcf5675527f029649'
        '2d81633fa49677b203663d547b0b35cf6e9ac78fbb1a25c56c786311e9e38c71e8e7cdd0260fada5502c04f3d486e11d4748366a6184d434b2dde6eacf625cfc')
options=(ccache lto)

_j=16 #change for your system

prepare() {
  export PATH="${srcdir}/bun-linux-x64:$PATH"

  cd bun
  bun i
  cd ..
}


build() {
  mkdir -p ./build

#   ln -sf $srcdir/Webkit ./bun/vendor/WebKit
# 
#   build_webkit

#   cd $srcdir/bun
# 
#   WEBKIT_DIR=$srcdir/WebKit make jsc-copy-headers
# 
#   cd ..

  CXXFLAGS="-Wno-unused-result ${CXXFLAGS}" cmake -GNinja -B $srcdir/build -S $srcdir/bun -Wno-dev -DCMAKE_BUILD_TYPE=Release -DUSE_STATIC_LIBATOMIC=OFF -DUSE_SYSTEM_ICU=OFF \
        -DENABLE_CCACHE=ON -DLLVM_PREFIX=/usr -DENABLE_LTO=ON -DENABLE_CANARY=OFF -DCPU_TARGET=native -DLLVM_VERSION=18.1.8 -DLLD_NAME="mold" #-DWEBKIT_LOCAL=ON -DWEBKIT_PATH=$srcdir/WebKit/WebKitBuild/Release

  ninja -C ./build -j$_j
}

build_webkit(){

  cd $srcdir/WebKit/

  mkdir -p WebKitBuild/Release

  # Adapted from https://github.com/oven-sh/WebKit/blob/main/Dockerfile#L109

  export DEFAULT_CFLAGS="-mno-omit-leaf-frame-pointer -fno-omit-frame-pointer -ffunction-sections -fdata-sections -faddrsig -fno-unwind-tables -fno-asynchronous-unwind-tables -DU_STATIC_IMPLEMENTATION=1 "
  # export LTO_FLAG="-flto=full -fwhole-program-vtables -fforce-emit-vtables "

  export CFLAGS="${DEFAULT_CFLAGS} $CFLAGS $LTO_FLAG -ffile-prefix-map=/webkit/Source=vendor/WebKit/Source  -ffile-prefix-map=/webkitbuild/=. "
  export CXXFLAGS="${DEFAULT_CFLAGS} $CXXFLAGS $LTO_FLAG -fno-c++-static-destructors -ffile-prefix-map=/webkit/Source=vendor/WebKit/Source -ffile-prefix-map=/webkitbuild/=. "
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
      -DENABLE_FTL_JIT=ON \
      -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
      -DALLOW_LINE_AND_COLUMN_NUMBER_IN_BUILTINS=ON \
      -DENABLE_REMOTE_INSPECTOR=ON \
      -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" \
      -DCMAKE_AR="/usr/bin/llvm-ar" \
      -DCMAKE_RANLIB="/usr/bin/llvm-ranlib" \
      -GNinja

  ninja -C ./WebKitBuild/Release jsc -j$_j

#   mkdir -p ./output/{lib,include/JavaScriptCore,Source/JavaScriptCore}
# 
#   cp -r ./build/lib/*.a ./output/lib
#   cp ./build/*.h ./output/include
#   find ./build/JavaScriptCore/Headers/JavaScriptCore/ -name "*.h" -exec cp {} ./output/include/JavaScriptCore/ \;
#   find ./build/JavaScriptCore/PrivateHeaders/JavaScriptCore/ -name "*.h" -exec cp {} ./output/include/JavaScriptCore/ \;
#   cp -r ./build/WTF/Headers/wtf/ ./output/include
#   cp -r ./build/bmalloc/Headers/bmalloc/ ./output/include
#   cp -r ./Source/JavaScriptCore/Scripts ./output/Source/JavaScriptCore
#   cp ./Source/JavaScriptCore/create_hash_table ./output/Source/JavaScriptCore
#   rm -rf ./output/include/unicode
#   cp -r /usr/include/unicode ./output/include/unicode
# 

  ln -sf /lib/libicudata.so ./WebKitBuild/Release/lib/libicudata.a
  ln -sf /lib/libicui18n.so ./WebKitBuild/Release/lib/libicui18n.a
  ln -sf /lib/libicuuc.so ./WebKitBuild/Release/lib/libicuuc.a
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
