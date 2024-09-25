# Maintainer: Daniele Basso <d dot bass 05 at proton dot me>
pkgname=bun
pkgver=1.1.29
pkgrel=1
pkgdesc="Bun is a fast JavaScript all-in-one toolkit. This PKGBUILD builds from source, resulting into a smaller and faster binary depending on your CPU."
arch=(x86_64)
url="https://github.com/oven-sh/bun"
license=('GPL')
depends=(c-ares libarchive libuv mimalloc tcc zlib zstd)
makedepends=(
	clang cmake esbuild git go icu libdeflate libiconv libtool lld llvm ninja pkg-config python ruby rust unzip zig
)
conflicts=(bun-bin)
source=(git+$url.git#tag=bun-v$pkgver
        bun-linux-x64-$pkgver.zip::https://github.com/oven-sh/bun/releases/download/bun-v$pkgver/bun-linux-x64.zip # add "baseline" here to download the avx2-less build of bun!
        git+https://github.com/oven-sh/WebKit.git#commit=4a2db3254a9535949a5d5380eb58cf0f77c8e15a
        describeFrame.patch)
b2sums=('2ea0c86bc5c120e2f452ca5614fe09c11f2b99c3747efe76e21764f25ed28fd6fee4cafc35ac8c44c5ab4aafc2be7494944e0be162bb322a3233cd6a9d712ccc'
        '05eaaa1369457ee47393a7e91d3ee74bccad11c39d83818034079e7e9b31b343a09996b36deac850eced8333e42b8c7cc6293d98a1ce8625aa54dffd60ee06d2'
        '82c6b8570b8fc7fd2dd030bf5fee70585b9cc82a1e6b096656e2fbfc6b2e8cbc81468a777b5f1daae874650de83f183f9e8f35170a2a14aa387d8dc6a0ebed74'
        '1f0c037df9ed2df72df9cb714843bc7f64cc6fd06482132d9b09846ab69db5cab6f5910c6e27d2b335af4b0fd7b2694f7fb27de1c4c34848b12cdcd9fd347f1f')
options=(!ccache lto)

_j=6 #change for your system

prepare() {
  export PATH="${srcdir}/bun-linux-x64:$PATH"

  cd bun
  git apply $srcdir/describeFrame.patch
  bun i
  rm -rf ./vendor/zig
  mkdir -p ./vendor/zig
  ln -sf /usr/bin ./vendor/zig/
  ln -sf /usr/lib/zig ./vendor/zig/lib
  cd ..
}


build() {
  mkdir -p ./build

  build_webkit

  cd $srcdir

  CXXFLAGS="-Wno-unused-result ${CXXFLAGS}" cmake -GNinja -B $srcdir/build -S $srcdir/bun -Wno-dev -DCMAKE_BUILD_TYPE=Release -DUSE_STATIC_LIBATOMIC=OFF -DUSE_SYSTEM_ICU=OFF \
        -DENABLE_CCACHE=OFF -DLLVM_PREFIX=/usr -DWEBKIT_PATH=$srcdir/WebKit/output -DWEBKIT_LOCAL=ON -DENABLE_LTO=ON -DCPU_TARGET=native -DLLVM_VERSION=18.1.8
  sed -i 's/lld-18/lld/g' ./build/build.ninja
  ninja -C ./build -j$_j
}

build_webkit(){

  cd $srcdir/WebKit/

  # Adapted from https://github.com/oven-sh/WebKit/blob/main/Dockerfile#L109

  COMMON_FLAGS="-mno-omit-leaf-frame-pointer -fno-omit-frame-pointer -ffunction-sections -fdata-sections -faddrsig -fno-unwind-tables -fno-asynchronous-unwind-tables -DU_STATIC_IMPLEMENTATION=1" \
  CC="/usr/bin/clang" CXX="/usr/bin/clang++" CFLAGS="${DEFAULT_CFLAGS} $CFLAGS" CXXFLAGS="${DEFAULT_CFLAGS} $CXXFLAGS -fno-c++-static-destructors" cmake \
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
