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

Remova o diretório de compilação:

```
rm -rf build
```

Guarde a árvore de diretórios para a compilação de compiler-rt, libunwind, libcxx, libcxxabi e clang (fase 2).

# • compiler-rt

Compilaremos somente os builtins e os crts do compiler-rt nessa fase.

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

Configure a compilação (TESTE):

```
cmake -G Ninja -S compiler-rt -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_SYSROOT="$STAGE1" \
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

Remova o diretório de compilação:

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

# • libatomic-chimera #teste compilar sem ele

Compile e instale:

```
make CC="$STAGE1/bin/clang --target=x86_64-alpes-linux-musl --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld -nostdlib -nodefaultlibs" PREFIX=/usr LIBDIR=/usr/lib
make CC="$STAGE1/bin/clang --target=x86_64-alpes-linux-musl --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld -nostdlib -nodefaultlibs" PREFIX=/usr LIBDIR=/usr/lib DESTDIR=$STAGE1 install
```

# • Runtimes C++ do Clang

Instalaremos os runtimes mínimos (libunwind, libcxx e libcxxabi) para daqui a pouco compilar um novo Clang mais independente do host.

Configure a compilação:

```
cmake -G Ninja -S runtimes -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_SYSROOT="$STAGE1" \
 -DCMAKE_C_COMPILER="$STAGE1/bin/clang" \
 -DCMAKE_CXX_COMPILER="$STAGE1/bin/clang++" \
 -DCMAKE_C_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_CXX_COMPILER_TARGET="$SYSTARGET" \
 -DCMAKE_C_FLAGS="$CFLAGS" \
 -DCMAKE_CXX_FLAGS="$CXXFLAGS -nostdlib++" \
 -DLLVM_ENABLE_RUNTIMES="libunwind;libcxx;libcxxabi" \
 -DLIBCXX_USE_COMPILER_RT=ON \
 -DLIBCXXABI_USE_COMPILER_RT=ON \
 -DLIBCXXABI_USE_LLVM_UNWINDER=ON \
 -DLIBCXX_HAS_MUSL_LIBC=ON \
 -DLIBCXX_ENABLE_SHARED=OFF \
 -DLIBCXX_ENABLE_STATIC=ON \
 -DLIBCXXABI_ENABLE_SHARED=OFF \
 -DLIBCXXABI_ENABLE_STATIC=ON
```

Compile e instale:

```
ninja -C build
DESTDIR=$STAGE1 ninja -C build install
```

Agora que temos as runtimes C++, atualize as antigas flags CXX e CXXFLAGS deixar o uso de libc++ explícito:

```
export CXX="$CC -stdlib=libc++"
export CXXFLAGS="$CFLAGS -stdlib=libc++"
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
  -DLLVM_ENABLE_LIBCXX=ON \
  -DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON \
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
