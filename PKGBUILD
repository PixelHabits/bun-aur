# Maintainer: Daniele Basso <d dot bass 05 at proton dot me>
pkgname=bun
pkgver=1.0.32
#_zigver=0.12.0-dev.1828+225fe6ddb #https://github.com/oven-sh/bun/blob/bun-v1.0.28/build.zig#L9
pkgrel=1
pkgdesc="Bun is a fast JavaScript all-in-one toolkit. This PKGBUILD builds from source, resulting into a smaller and faster binary depending on your CPU."
arch=(x86_64)
url="https://github.com/oven-sh/bun"
license=('GPL')
makedepends=(
	clang16 cmake esbuild git go icu libiconv libtool lld16 llvm16 ninja pkg-config python ruby rust unzip
)
conflicts=(bun-bin)
source=(git+$url.git#tag=bun-v$pkgver
        bun-linux-x64-$pkgver.zip::https://github.com/oven-sh/bun/releases/download/bun-v$pkgver/bun-linux-x64.zip) # add "baseline" here to download the avx2-less build of bun!
b2sums=('84779f74d627def435019edaebc433da7cffdd37e3427d1fda71276b16dbdc5749dbbd4d577fcc316c6d2ee66604dff263299f8aa6042e4882f980e188f06b22'
        'bb56953dfdaac24f5b5c870cee97e29df60ad83604ddebb7530bb4c1caad6354ee0d1aaaf0dc06bacc376d3c42e33f265ad6cf115297da576837a4bccbe78e52')

_j=$(($(nproc)/2)) #change for your system

prepare() {
  export PATH="${srcdir}/bun-linux-x64:$PATH"

  cd "$pkgname"

# 
#   export PATH="$PATH":$srcdir/zig-linux-x86_64-$_zigver
# 
#   mkdir -p $srcdir/bun/.cache/
#   # ln -sf $srcdir/zig-linux-$_zigver $srcdir/bun/.cache/zig
#   # ln -sf $srcdir/zig-linux-$_zigver.tar.xz $srcdir/bun/.cache/
  git -c submodule.src/javascript/jsc/WebKit.update=checkout submodule update --init --recursive --depth=1 --progress

  bun i
  #cd test; bun i; cd ..
  
  bash ./scripts/all-dependencies.sh
  bash ./scripts/download-zig.sh
}

build() {
  export PATH="${srcdir}/bun-linux-x64:$PATH"

  cd $srcdir/bun/

  make runtime_js fallback_decoder bun_error node-fallbacks

  cd src/bun.js/WebKit/

  # Adapted from https://github.com/oven-sh/WebKit/blob/main/Dockerfile#L60
  #export CFLAGS="$CFLAGS -ffat-lto-objects"
  #export CXXFLAGS="$CXXFLAGS -ffat-lto-objects"

  CC="clang" CXX="clang++" cmake \
      -S . \
      -DCMAKE_CXX_FLAGS="-fuse-ld=lld" \
      -B ./build \
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

  ln -sf /lib/libicudata.so ./output/lib/libicudata.a
  ln -sf /lib/libicui18n.so ./output/lib/libicui18n.a
  ln -sf /lib/libicuuc.so ./output/lib/libicuuc.a

  # make jsc

  cmake -B $srcdir/$pkgname/build -S $srcdir/$pkgname -DCMAKE_BUILD_TYPE=Release -GNinja -DUSE_STATIC_LIBATOMIC=OFF -DWEBKIT_DIR=$srcdir/bun/src/bun.js/WebKit/output -DUSE_DEBUG_JSC=OFF -DZIG_OPTIMIZE=ReleaseFast
  ninja -C $srcdir/$pkgname/build -j$_j
}

package() {
  install -Dm755 $srcdir/$pkgname/build/bun $pkgdir/usr/bin/bun
  ln -s /usr/bin/bun $pkgdir/usr/bin/bunx

  SHELL=zsh $pkgdir/usr/bin/bun completions > bun.zsh
  SHELL=bash $pkgdir/usr/bin/bun completions > bun.bash
  SHELL=fish $pkgdir/usr/bin/bun completions > bun.fish

  install -Dm644 bun.zsh $pkgdir/usr/share/zsh/site-functions/_bun
  install -Dm644 bun.bash $pkgdir/usr/share/bash-completion/completions/bun
  install -Dm644 bun.fish $pkgdir/usr/share/fish/vendor_completions.d/bun.fish
}
