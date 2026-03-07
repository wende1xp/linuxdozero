É aqui que iremos compilar as ferramentas necessárias para tornar o sistema independente do host usando nosso compilador inicial.

Antes de começarmos a compilar qualquer pacote, defina as variáveis com esses comandos:

```
export CC="$STAGE1/bin/clang --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld"
export CXX="$STAGE1/bin/clang++ --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld"

export AR="$STAGE1/bin/llvm-ar"
export RANLIB="$STAGE1/bin/llvm-ranlib"
export LD="$STAGE1/bin/ld.lld"
export NM="$STAGE1/bin/llvm-nm"

export CFLAGS="-fPIC -Wno-unused-command-line-argument"
export CXXFLAGS="$CFLAGS"
```

# • Cabeçalhos da API do Linux 6.19.5

Certifique-se de que não existem arquivos obsoletos embutidos no pacote: 

```
make mrproper CC=clang
```

Compile os cabeçalhos e os instale na raiz do sistema que estamos construindo:

```
make headers_install HOSTCC="$STAGE1/bin/clang --rtlib=compiler-rt -fuse-ld=lld" HOSTCFLAGS="--sysroot=$STAGE1" ARCH=x86_64 INSTALL_HDR_PATH=$STAGE2/usr
```

# • Musl

Aplique as correções de segurança usando esse loop que aplica todas elas automaticamente:

```
for patch in $BUILDDIR/sources/patches/musl/*.patch; do
    echo "Aplicando $patch..."
    patch -Np1 --quiet < "$patch" || exit 1
done
```

Configure a compilação:

```
./configure --prefix=/usr --syslibdir=/lib --target=$SYSTARGET --disable-gcc-wrapper
```

Compile e instale:

```
make
make DESTDIR=$STAGE2 install
```

Agora que temos uma libc, então, agora podemos compilar apontando a raíz para $STAGE2.

```
export CC="$STAGE1/bin/clang --target=$SYSTARGET --sysroot=$STAGE2 -rtlib=compiler-rt -fuse-ld=lld"
export CXX="$STAGE1/bin/clang++ --target=$SYSTARGET --sysroot=$STAGE2 -rtlib=compiler-rt -fuse-ld=lld"

export AR="$STAGE1/bin/llvm-ar"
export RANLIB="$STAGE1/bin/llvm-ranlib"
export LD="$STAGE1/bin/ld.lld"
export NM="$STAGE1/bin/llvm-nm"

export CFLAGS="-fPIC -Wno-unused-command-line-argument"
export CXXFLAGS="$CFLAGS"
```

# • zlib-ng-compat (2.3.3.tar.gz)

Configure a compilação:

```
./configure --prefix=/usr --libdir=/lib --zlib-compat --shared
```

Compile e instale:

```
make
make DESTDIR=$STAGE2 install
```

# • libatomic-chimera (v0.90.0.tar.gz)

Compile e instale:

```
make PREFIX=/usr LIBDIR=/usr/lib
make PREFIX=/usr LIBDIR=/usr/lib DESTDIR=$STAGE2 install
```

# • compiler-rt (llvm-project-21.1.8.src.tar.xz)

Aplique as correções necessárias usando esse loop que aplica todas elas automaticamente:

```
for patch in $BUILDDIR/sources/patches/llvm/*.patch; do
    echo "Aplicando $patch..."
    patch -Np1 --quiet < "$patch" || exit 1
done
```

Configure a compilação:

```
cd compiler-rt

cmake -G Ninja -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_C_COMPILER="$STAGE1/bin/clang" \
 -DCMAKE_CXX_COMPILER="$STAGE1/bin/clang++" \
 -DCMAKE_C_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_CXX_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_C_FLAGS="-fPIC -rtlib=compiler-rt -Wno-unused-command-line-argument" \
 -DCMAKE_CXX_FLAGS="-fPIC -rtlib=compiler-rt -nostdlib++ -Wno-unused-command-line-argument" \
 -DCMAKE_SYSROOT="$STAGE2" \
 -DLIBUNWIND_INSTALL_HEADERS=ON \
 -DLIBUNWIND_ENABLE_STATIC=OFF \
 -DLIBUNWIND_USE_COMPILER_RT=ON \
 -DLIBUNWIND_HIDE_SYMBOLS=ON \
 -DLIBUNWIND_ENABLE_THREADS=ON \
 -DLIBUNWIND_ENABLE_CROSS_UNWINDING=ON \
 -DLIBUNWIND_ENABLE_ASSERTIONS=OFF \
 -DLIBUNWIND_ENABLE_SHARED=ON
```

# • libunwind (llvm-project-21.1.8.src.tar.xz)

Aplique as correções necessárias usando esse loop que aplica todas elas automaticamente:

```
for patch in $BUILDDIR/sources/patches/llvm/*.patch; do
    echo "Aplicando $patch..."
    patch -Np1 --quiet < "$patch" || exit 1
done
```

Configure a compilação:

```
cd libunwind

cmake -G Ninja -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_C_COMPILER="$STAGE1/bin/clang" \
 -DCMAKE_CXX_COMPILER="$STAGE1/bin/clang++" \
 -DCMAKE_C_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_CXX_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_C_FLAGS="-fPIC -rtlib=compiler-rt -Wno-unused-command-line-argument" \
 -DCMAKE_CXX_FLAGS="-fPIC -rtlib=compiler-rt -nostdlib++ -Wno-unused-command-line-argument" \
 -DCMAKE_SYSROOT="$STAGE2" \
 -DLIBUNWIND_INSTALL_HEADERS=ON \
 -DLIBUNWIND_ENABLE_STATIC=OFF \
 -DLIBUNWIND_USE_COMPILER_RT=ON \
 -DLIBUNWIND_HIDE_SYMBOLS=ON \
 -DLIBUNWIND_ENABLE_THREADS=ON \
 -DLIBUNWIND_ENABLE_CROSS_UNWINDING=ON \
 -DLIBUNWIND_ENABLE_ASSERTIONS=OFF \
 -DLIBUNWIND_ENABLE_SHARED=ON
```

Compile e instale:

```
ninja -C build
DESTDIR=$STAGE2 ninja -C build install
```

Não apague a árvore de diretórios atual pois ainda usaremos na compilação dos próximos programas.

# • libc++ (llvm-project-21.1.8.src.tar.xz)

Assumindo que você ainda esteja no diretório anterior (libunwind), basta entrar no diretório com esse comando:

```
cd ../libcxx
```

Configure a compilação:

headers

```
cmake -G Ninja -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_C_COMPILER="$STAGE1/bin/clang" \
 -DCMAKE_CXX_COMPILER="$STAGE1/bin/clang++" \
 -DCMAKE_C_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_CXX_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_C_FLAGS="-fPIC -rtlib=compiler-rt -Wno-unused-command-line-argument" \
 -DCMAKE_CXX_FLAGS="-fPIC -rtlib=compiler-rt -unwindlib=libunwind -nostdlib++ -Wno-unused-command-line-argument" \
 -DLIBCXX_ENABLE_SHARED=ON \
 -DLIBCXX_ENABLE_STATIC=OFF \
 -DLIBCXX_CXX_ABI=none \
 -DLIBCXX_INSTALL_HEADERS=ON
```

```
DESTDIR=$STAGE2 ninja -C build install-cxx-headers
```




```
cmake -G Ninja -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_C_COMPILER="$STAGE1/bin/clang" \
 -DCMAKE_CXX_COMPILER="$STAGE1/bin/clang++" \
 -DCMAKE_C_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_CXX_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_C_FLAGS="-fPIC -rtlib=compiler-rt -unwindlib=libunwind -Wno-unused-command-line-argument" \
 -DCMAKE_CXX_FLAGS="-fPIC -rtlib=compiler-rt -unwindlib=libunwind -nostdlib++ -Wno-unused-command-line-argument" \
 -DCMAKE_SYSROOT="$STAGE2" \
 -DLIBCXX_USE_COMPILER_RT=ON \
 -DLIBCXX_ENABLE_SHARED=ON \
 -DLIBCXX_ENABLE_STATIC=OFF \
 -DLIBCXX_HAS_MUSL_LIBC=ON \
 -DLIBCXX_INCLUDE_TESTS=OFF
```



cmake -G Ninja -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_C_COMPILER="$STAGE1/bin/clang" \
 -DCMAKE_CXX_COMPILER="$STAGE1/bin/clang++" \
 -DCMAKE_C_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_CXX_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_C_FLAGS="-fPIC -rtlib=compiler-rt -unwindlib=libunwind -Wno-unused-command-line-argument" \
 -DCMAKE_CXX_FLAGS="-fPIC -rtlib=compiler-rt -unwindlib=libunwind -Wno-unused-command-line-argument" \
 -DLIBCXX_USE_COMPILER_RT=ON \
 -DLIBCXX_ENABLE_SHARED=ON \
 -DLIBCXX_ENABLE_STATIC=OFF \
 -DLIBCXX_HAS_MUSL_LIBC=ON \
 -DLIBCXX_INCLUDE_TESTS=OFF


Compile e instale:

```
ninja -C build
DESTDIR=$STAGE2 ninja -C build install
```

# • libc++abi (llvm-project-21.1.8.src.tar.xz)

Assumindo que você ainda esteja no diretório anterior (libunwind), basta entrar no diretório com esse comando:

```
cd ../libcxxabi
```

Configure a compilação:

```
cmake -G Ninja -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_C_COMPILER="$STAGE1/bin/clang" \
 -DCMAKE_CXX_COMPILER="$STAGE1/bin/clang++" \
 -DCMAKE_C_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_CXX_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_C_FLAGS="-fPIC -rtlib=compiler-rt -unwindlib=libunwind -Wno-unused-command-line-argument" \
 -DCMAKE_CXX_FLAGS="-fPIC -rtlib=compiler-rt -unwindlib=libunwind -nostdlib++ -Wno-unused-command-line-argument" \
 -DLLVM_ENABLE_RUNTIMES="libunwind" \
 -DLIBCXXABI_USE_LLVM_UNWINDER=ON \
 -DLIBUNWIND_INCLUDE_DIR=$STAGE2/usr/include \
 -DLIBUNWIND_LIBRARY=$STAGE2/usr/lib/libunwind.so \
 -DLIBCXXABI_ENABLE_STATIC_UNWINDER=OFF \
 -DLIBCXXABI_USE_COMPILER_RT=ON \
 -DLIBCXXABI_ENABLE_SHARED=ON \
 -DLIBCXXABI_ENABLE_STATIC=OFF \
 -DLIBCXXABI_LIBCXX_INCLUDES="$BUILDDIR/sources/pkgs/llvm-project-21.1.8.src/libcxx/include" \
 -DLIBCXXABI_INCLUDE_TESTS=OFF
```

Compile e instale:

```
ninja -C build
DESTDIR=$STAGE2 ninja -C build install
```

Capítulo Anterior:
[Capítulo 2 - Ferramentas de Compilação (Fase 1)](cap2.md)

Próximo Capítulo:
[Capítulo 4 - Ferramentas de Compilação (Fase 3)](cap4.md)
