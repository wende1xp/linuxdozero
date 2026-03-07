Fluxo vai mudar para:
- 1  Linux headers
- 2  clang minimal
- 3  compiler-rt builtins
- 4  musl
- 5  clang completo + runtimes


Este capítulo mostra como construir um compilador cruzado e as ferramentas associadas usando o ambiente do sistema host. Os programas compilados neste capítulo serão instalados sob o diretório $LFS/tools para mantê-los separados dos arquivos instalados nos capítulos seguintes.

Entre no diretório que contém os pacotes:

```
cd $BUILDDIR/sources/pkgs
```

# • Cabeçalhos da API do Linux 6.19.5

Certifique-se de que não existem arquivos obsoletos embutidos no pacote: 

```
make mrproper CC=clang
```

Compile os cabeçalhos e os instale na raiz do sistema que estamos construindo:

```
make headers_install HOSTCC=/usr/bin/clang ARCH=x86_64 INSTALL_HDR_PATH=$STAGE1/usr
```

# • Clang (Fase 1)

Esse vai ser o nosso compilador inicial, necessário para construir as dependências do compilador da fase 2.

Aplique as correções necessárias usando esse loop que aplica todas elas automaticamente:

```
for patch in $BUILDDIR/sources/patches/llvm/*.patch; do
    echo "Aplicando $patch..."
    patch -Np1 --quiet < "$patch" || exit 1
done
```

Configure a compilação:

```
cmake -G Ninja -S llvm -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$STAGE1 \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_ENABLE_RUNTIMES="" \
  -DLLVM_DEFAULT_TARGET_TRIPLE=$SYSTARGET \
  -DCLANG_DEFAULT_LINKER=lld \
  -DCLANG_DEFAULT_RTLIB=compiler-rt \
  -DCLANG_DEFAULT_UNWINDLIB=libunwind \
  -DLLVM_INCLUDE_TESTS=OFF \
  -DLLVM_INCLUDE_EXAMPLES=OFF \
  -DLLVM_ENABLE_BINDINGS=OFF \
  -DLLVM_BUILD_TOOLS=ON \
  -DLLVM_BUILD_UTILS=ON
```

Compile e instale:

```
ninja -C build
ninja -C build install
```

# • compiler-rt (bultins + crts)

Use a mesma árvore de diretórios do clang.

Configure a compilação:

```
cmake -G Ninja -S compiler-rt -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=$STAGE1 \
 -DCMAKE_C_COMPILER=$STAGE1/bin/clang \
 -DCMAKE_AR=$STAGE1/bin/llvm-ar \
 -DCMAKE_RANLIB=$STAGE1/bin/llvm-ranlib \
 -DCOMPILER_RT_BUILD_BUILTINS=ON \
 -DCOMPILER_RT_BUILD_CRT=ON \
 -DCOMPILER_RT_BUILD_SANITIZERS=OFF \
 -DCOMPILER_RT_BUILD_XRAY=OFF \
 -DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
 -DCOMPILER_RT_BUILD_PROFILE=OFF \
 -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON \
 -DCMAKE_C_COMPILER_TARGET=$SYSTARGET \
 -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY
```

Compile e instale:

```
ninja -C build install-builtins
ninja -C build install-crt
```

Após isso remova o diretório de compilação para liberar armazenamento:

```
rm -rf build
```

Faça links simbólicos para corrigir futuros problemas de compatibilidade ao compilar o musl:

```
mkdir -p $STAGE1/usr/lib/clang/21/lib/x86_64-alpes-linux-musl

ln -sv $STAGE1/usr/lib/linux/libclang_rt.builtins-x86_64.a $STAGE1/usr/lib/clang/21/lib/x86_64-alpes-linux-musl/libclang_rt.builtins.a
ln -sv $STAGE1/usr/lib/linux/clang_rt.crtbegin-x86_64.o $STAGE1/usr/lib/clang/21/lib/x86_64-alpes-linux-musl/clang_rt.crtbegin.o
ln -sv $STAGE1/usr/lib/linux/clang_rt.crtend-x86_64.o $STAGE1/usr/lib/clang/21/lib/x86_64-alpes-linux-musl/clang_rt.crtend.o
```

Não remova a árvore de diretórios que usamos pois ainda vamos usar ela para recompilar o clang (fase 2).

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
CC="$STAGE1/bin/clang --target=$SYSTARGET --sysroot=$STAGE1" AR="$STAGE1/bin/llvm-ar" RANLIB="$STAGE1/bin/llvm-ranlib" ./configure --prefix=/usr --syslibdir=/lib --target=$SYSTARGET --disable-gcc-wrapper
```

Compile e instale:

```
make
make DESTDIR=$STAGE1 install
```

Agora que temos o musl podemos definir as flags necessárias para compilar os próximos pacotes:

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

# • libatomic-chimera (v0.90.0.tar.gz)

Compile e instale:

```
make CC="$STAGE1/bin/clang --target=x86_64-alpes-linux-musl --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld -nostdlib -nodefaultlibs" PREFIX=/usr LIBDIR=/usr/lib
make CC="$STAGE1/bin/clang --target=x86_64-alpes-linux-musl --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld -nostdlib -nodefaultlibs" PREFIX=/usr LIBDIR=/usr/lib DESTDIR=$STAGE1 install
```

# • libunwind

Utilize a árvore de diretórios que não removemos ao compilar compiler-rt.

Configure a compilação:

```
cmake -G Ninja -S libunwind -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_SYSROOT="$STAGE1" \
 -DCMAKE_C_COMPILER="$STAGE1/bin/clang" \
 -DCMAKE_CXX_COMPILER="$STAGE1/bin/clang++" \
 -DCMAKE_C_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_CXX_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_C_FLAGS="-fPIC -rtlib=compiler-rt -Wno-unused-command-line-argument" \
 -DCMAKE_CXX_FLAGS="-fPIC -rtlib=compiler-rt -nostdlib++ -Wno-unused-command-line-argument" \
 -DLIBUNWIND_INSTALL_HEADERS=ON \
 -DLIBUNWIND_ENABLE_STATIC=OFF \
 -DLIBUNWIND_ENABLE_SHARED=ON \
 -DLIBUNWIND_USE_COMPILER_RT=ON \
 -DLIBUNWIND_HIDE_SYMBOLS=ON \
 -DLIBUNWIND_ENABLE_THREADS=ON \
 -DLIBUNWIND_ENABLE_CROSS_UNWINDING=ON \
 -DLIBUNWIND_ENABLE_ASSERTIONS=OFF \
 -DLIBUNWIND_ENABLE_CXX_EXCEPTIONS=OFF \
 -DLIBUNWIND_ENABLE_SHARED=ON
```

Compile e instale:

```
ninja -C build
DESTDIR=$STAGE2 ninja -C build install
```

# • Clang (Fase 2)

Esse vai ser o nosso compilador final para o $STAGE1, esse será o compilador usado para compilar os próximos programas em $STAGE2.

Configure a compilação:

```
cmake -G Ninja -S llvm -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$STAGE1 \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_ENABLE_RUNTIMES="compiler-rt;libunwind;libcxxabi;libcxx" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_DEFAULT_TARGET_TRIPLE=$SYSTARGET \
  -DCLANG_DEFAULT_LINKER=lld \
  -DCLANG_DEFAULT_RTLIB=compiler-rt \
  -DCLANG_DEFAULT_CXX_STDLIB=libc++ \
  -DCLANG_DEFAULT_UNWINDLIB=libunwind \
  -DLIBCXX_USE_COMPILER_RT=ON \
  -DLIBCXXABI_USE_COMPILER_RT=ON \
  -DLIBUNWIND_USE_COMPILER_RT=ON \
  -DLLVM_INCLUDE_TESTS=OFF
```

TESTE:
```
cmake -G Ninja -S llvm -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$STAGE1 \
  -DCMAKE_SYSROOT=$STAGE1 \
  -DCMAKE_C_COMPILER=$STAGE1/bin/clang \
  -DCMAKE_CXX_COMPILER=$STAGE1/bin/clang++ \
  -DCMAKE_C_COMPILER_TARGET=$SYSTARGET \
  -DCMAKE_CXX_COMPILER_TARGET=$SYSTARGET \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_ENABLE_RUNTIMES="compiler-rt;libunwind;libcxxabi;libcxx" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_DEFAULT_TARGET_TRIPLE=$SYSTARGET \
  -DCLANG_DEFAULT_SYSROOT=$STAGE1 \
  -DCLANG_DEFAULT_LINKER=lld \
  -DCLANG_DEFAULT_RTLIB=compiler-rt \
  -DCLANG_DEFAULT_CXX_STDLIB=libc++ \
  -DCLANG_DEFAULT_UNWINDLIB=libunwind \
  -DLIBCXX_USE_COMPILER_RT=ON \
  -DLIBCXXABI_USE_COMPILER_RT=ON \
  -DLIBUNWIND_USE_COMPILER_RT=ON \
  -DLLVM_INCLUDE_TESTS=OFF
```

Compile e instale:

```
ninja -C build
ninja -C build install
```

Capítulo Anterior:
[Capítulo 1 - Preparação](cap1.md)

Próximo Capítulo:
[Capítulo 3 - Ferramentas de Compilação (Fase 2)](cap3.md)
