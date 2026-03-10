Este capítulo mostra como construir um compilador cruzado e as ferramentas associadas usando o ambiente do sistema host. Os programas compilados neste capítulo serão instalados sob o diretório $TOOLS para mantê-los separados dos arquivos instalados nos capítulos seguintes.

Entre no diretório que contém os pacotes:

```
cd "$ROOTFS"/sources/pkgs
```

# • Clang (Fase 1)

Esse vai ser o nosso compilador inicial, necessário para construir as dependências do compilador da fase 2.

Aplique as correções necessárias usando esse loop que aplica todas elas automaticamente:

```
for patch in "$ROOTFS"/sources/patches/llvm/*.patch; do
    echo "Aplicando $patch..."
    patch -Np1 --quiet < "$patch" || exit 1
done
```

Configure a compilação:

```
cmake -G Ninja -S llvm -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$TOOLS \
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

# • Cabeçalhos do Musl

Para posteriormente compilar compiler-rt, devemos instalar os cabeçalhos do musl para atender seus requirementos mínimos: 

Aplique as correções de segurança usando esse loop que aplica todas elas automaticamente:

```
for patch in "$ROOTFS"/sources/patches/musl/*.patch; do
    echo "Aplicando $patch..."
    patch -Np1 --quiet < "$patch" || exit 1
done
```

Configure a compilação:

```
CC=$TOOLS/usr/bin/clang ./configure --prefix=/usr --syslibdir=/lib --target=$SYSTARGET --disable-gcc-wrapper
```

Compile os cabeçalhos e os instale na raiz do sistema que estamos construindo:

```
make DESTDIR=$ROOTFS install-headers
```

Não remova a árvore de diretórios por enquanto pois ainda compilaremos o musl por completo, limpe somente nossa compilação:

```
make clean
```

# • compiler-rt

Compilaremos somente os builtins e os crts do compiler-rt nessa fase.

Para preservar a integridade da árvore de diretórios que estamos usando para compilar o clang e seus runtimes, faça uma cópia da pasta compiler-rt pois vamos modificar-la para esquivar da dependência circular entre compiler-rt e libcxxabi e libcxx:

```
cp -r compiler-rt compiler-rt.bak
```

Configure a compilação:

```
cmake -G Ninja -S compiler-rt -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=/usr \
 -DCMAKE_SYSROOT=$ROOTFS \
 -DCMAKE_C_COMPILER=$TOOLS/bin/clang \
 -DCMAKE_AR=$TOOLS/bin/llvm-ar \
 -DCMAKE_RANLIB=$TOOLS/bin/llvm-ranlib \
 -DCOMPILER_RT_BUILD_BUILTINS=ON \
 -DCOMPILER_RT_BUILD_CRT=ON \
 -DCOMPILER_RT_BUILD_SANITIZERS=OFF \
 -DCOMPILER_RT_BUILD_XRAY=OFF \
 -DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
 -DCOMPILER_RT_BUILD_PROFILE=OFF \
 -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON \
 -DCMAKE_C_COMPILER_TARGET=$SYSTARGET \
 -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY \
 -DCOMPILER_RT_EXCLUDE_ATOMIC_BUILTIN=OFF
```

Como não temos runtimes C++ ainda, omita gcc_personality_v0.c para não termos problemas:

```
gsed -i '/gcc_personality_v0.c/d' compiler-rt/lib/builtins/CMakeLists.txt
```

Compile e instale:

```
DESTDIR=$ROOTFS ninja -C build install-builtins
DESTDIR=$ROOTFS ninja -C build install-crt
```

Remova o diretório que modificamos e renomeie o backup anteriormente feito com o nome do diretório original:

```
rm -rf compiler-rt

mv compiler-rt.bak compiler-rt
```

Remova o diretório de compilação:

```
rm -rf build
```

Faça links simbólicos para corrigir problemas de caminhos incorretos ao compilar o musl:

```
mkdir -p $TOOLS/usr/lib/clang/21/lib/$SYSTARGET

ln -sv $ROOTFS/usr/lib/linux/libclang_rt.builtins-x86_64.a $TOOLS/usr/lib/clang/21/lib/$SYSTARGET/libclang_rt.builtins.a
ln -sv $ROOTFS/usr/lib/linux/clang_rt.crtbegin-x86_64.o $TOOLS/usr/lib/clang/21/lib/$SYSTARGET/clang_rt.crtbegin.o
ln -sv $ROOTFS/usr/lib/linux/clang_rt.crtend-x86_64.o $TOOLS/usr/lib/clang/21/lib/$SYSTARGET/clang_rt.crtend.o
```

# • Cabeçalhos do Linux

Certifique-se de que não existem arquivos obsoletos embutidos no pacote: 

```
make mrproper CC=clang
```

Compile os cabeçalhos e os instale na raiz do sistema que estamos construindo:

```
make headers_install HOSTCC=$TOOLS/usr/bin/clang ARCH=x86_64 INSTALL_HDR_PATH=$ROOTFS/usr
```

# • Musl

Configure a compilação:

```
CC="$TOOLS/bin/clang" AR="$TOOLS/bin/llvm-ar" RANLIB="$TOOLS/bin/llvm-ranlib" CFLAGS="--target=$SYSTARGET --sysroot=$ROOTFS" LDFLAGS="--target=$SYSTARGET --sysroot=$ROOTFS -rtlib=compiler-rt" ./configure --prefix=/usr --syslibdir=/lib --target=$SYSTARGET --disable-gcc-wrapper
```

Compile e instale:

```
make
make DESTDIR=$ROOTFS install
```

Agora que temos o musl podemos definir as flags necessárias para compilar os próximos pacotes:

```
export CC="$TOOLS/bin/clang --target=$SYSTARGET --sysroot=$ROOTFS -rtlib=compiler-rt -fuse-ld=lld"
export CXX="$TOOLS/bin/clang++ --target=$SYSTARGET --sysroot=$ROOTFS -rtlib=compiler-rt -fuse-ld=lld"

export AR="$TOOLS/bin/llvm-ar"
export RANLIB="$TOOLS/bin/llvm-ranlib"
export LD="$TOOLS/bin/ld.lld"
export NM="$TOOLS/bin/llvm-nm"

export CFLAGS="-fPIC"
export CXXFLAGS="$CFLAGS"
```

# • libatomic-chimera

Compile e instale:

```
make CC="$TOOLS/bin/clang --target=$SYSTARGET --sysroot=$ROOTFS -rtlib=compiler-rt -fuse-ld=lld -nostdlib -nodefaultlibs" PREFIX=/usr LIBDIR=/usr/lib
make CC="$TOOLS/bin/clang --target=$SYSTARGET --sysroot=$ROOTFS -rtlib=compiler-rt -fuse-ld=lld -nostdlib -nodefaultlibs" PREFIX=/usr LIBDIR=/usr/lib DESTDIR=$ROOTFS install
```

# • Runtimes do Clang

Instalaremos os runtimes mínimos (libunwind, libcxxabi e libcxx) para daqui a pouco compilar um novo Clang mais independente do host.

Configure a compilação:

```
cmake -G Ninja -S runtimes -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr \
  -DCMAKE_SYSROOT=$ROOTFS \
  -DCMAKE_C_COMPILER=$TOOLS/bin/clang \
  -DCMAKE_CXX_COMPILER=$TOOLS/bin/clang++ \
  -DCMAKE_C_COMPILER_TARGET=$SYSTARGET \
  -DCMAKE_CXX_COMPILER_TARGET=$SYSTARGET \
  -DCMAKE_C_FLAGS="$CFLAGS" \
  -DCMAKE_CXX_FLAGS="$CXXFLAGS" \
  -DLLVM_ENABLE_RUNTIMES="libunwind;libcxxabi;libcxx" \
  -DLIBUNWIND_USE_COMPILER_RT=ON \
  -DLIBUNWIND_ENABLE_SHARED=OFF \
  -DLIBUNWIND_ENABLE_STATIC=ON \
  -DLIBUNWIND_ENABLE_THREADS=OFF \
  -DLIBCXXABI_USE_COMPILER_RT=ON \
  -DLIBCXXABI_USE_LLVM_UNWINDER=ON \
  -DLIBCXXABI_HAS_CXA_THREAD_ATEXIT_IMPL=OFF \
  -DLIBCXXABI_ENABLE_SHARED=OFF \
  -DLIBCXXABI_ENABLE_STATIC=ON \
  -DLIBCXX_USE_COMPILER_RT=ON \
  -DLIBCXX_HAS_MUSL_LIBC=ON \
  -DLIBCXX_ENABLE_SHARED=OFF \
  -DLIBCXX_ENABLE_STATIC=ON \
  -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY
```

Compile e instale:

```
ninja -C build
DESTDIR=$ROOTFS ninja -C build install
```

Agora que temos as runtimes C++, atualize as antigas flags CXX e CXXFLAGS deixar o uso de libc++ explícito:

```
export CC="$TOOLS/bin/clang --target=$SYSTARGET --sysroot=$ROOTFS  --unwindlib=libunwind -rtlib=compiler-rt -fuse-ld=lld"
export CXX="$TOOLS/bin/clang++ --target=$SYSTARGET --sysroot=$ROOTFS -rtlib=compiler-rt -fuse-ld=lld -stdlib=libc++"
export CXXFLAGS="$CFLAGS -stdlib=libc++"
```

Capítulo Anterior:
[Capítulo 1 - Preparação](cap1.md)

Próximo Capítulo:
[Capítulo 3 - Reservado](cap3.md)
