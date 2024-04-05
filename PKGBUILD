# Maintainer: Daniele Basso <d dot bass 05 at proton dot me>
pkgname=bun
pkgver=1.1.1
pkgrel=1
pkgdesc="Bun is a fast JavaScript all-in-one toolkit. This PKGBUILD builds from source, resulting into a smaller and faster binary depending on your CPU."
arch=(x86_64)
url="https://github.com/oven-sh/bun"
license=('GPL')
depends=(zlib libarchive mimalloc libuv sqlite)
makedepends=(
	clang16 cmake esbuild git go icu libiconv libtool lld16 llvm16 ninja pkg-config python ruby rust unzip
)
conflicts=(bun-bin)
source=(git+$url.git#tag=bun-v$pkgver
        bun-linux-x64-$pkgver.zip::https://github.com/oven-sh/bun/releases/download/bun-v$pkgver/bun-linux-x64.zip) # add "baseline" here to download the avx2-less build of bun!
b2sums=('7421951a3b1cf33ea4874b7513e5361f321b31adcde0b70eef49ce61516bcf65ca02fed40c828d34e3b581defdb43846e5ca7bdb5194df8534f0f5cb440fd563'
        '4bbcda0229e68dc511c7d8712533abd7376e809ff9f013721ed99b5da8c2228de0ae868a5164c43cfdcdc6efe78a99c7b56ba7effab7444e702b4d2a1fa4fd60')

_j=$(($(nproc)/2)) #change for your system

prepare() {
  export PATH="${srcdir}/bun-linux-x64:$PATH"

  cd bun

  bun i
  bash ./scripts/update-submodules.sh --webkit
  bash ./scripts/set-webkit-submodule-to-cmake.sh
}

build() {
  export PATH="${srcdir}/bun-linux-x64:$PATH"

  cd bun

  bash ./scripts/all-dependencies.sh

  make runtime_js fallback_decoder bun_error node-fallbacks

  cd src/bun.js/WebKit/

  # Adapted from https://github.com/oven-sh/WebKit/blob/main/Dockerfile#L60
  #export CFLAGS="$CFLAGS -ffat-lto-objects"
  #export CXXFLAGS="$CXXFLAGS -ffat-lto-objects"

  CC="/usr/lib/llvm16/bin/clang" CXX="/usr/lib/llvm16/bin/clang++" cmake \
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

  cmake -B $srcdir/build -S $srcdir/$pkgname -Wno-dev -DCMAKE_BUILD_TYPE=Release -GNinja -DUSE_STATIC_LIBATOMIC=OFF \
        -DWEBKIT_DIR=$srcdir/bun/src/bun.js/WebKit/output -DUSE_DEBUG_JSC=OFF -DZIG_OPTIMIZE=ReleaseFast \
        -DLLVM_PREFIX=/usr/lib/llvm16 -DCMAKE_CXX_COMPILER=/usr/lib/llvm16/bin/clang++ -DCMAKE_C_COMPILER=/usr/lib/llvm16/bin/clang \
        -DUSE_LTO=ON -DUSE_CUSTOM_ZLIB=OFF -DUSE_CUSTOM_LIBARCHIVE=OFF -DUSE_CUSTOM_MIMALLOC=OFF -DUSE_CUSTOM_LIBUV=OFF -DUSE_STATIC_SQLITE=OFF 
  ninja -C $srcdir/build -j$_j
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
