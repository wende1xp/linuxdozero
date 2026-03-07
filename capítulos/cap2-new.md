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

# Precisa de revisão

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
  -DLLVM_INCLUDE_TESTS=OFF \
  -DLLVM_INCLUDE_EXAMPLES=OFF \
  -DLLVM_ENABLE_BINDINGS=OFF \
  -DLLVM_ENABLE_OCAMLDOC=OFF \
  -DLLVM_BUILD_TOOLS=ON \
  -DLLVM_BUILD_UTILS=ON \
  -DLLVM_DEFAULT_TARGET_TRIPLE=$SYSTARGET
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
cmake -G Ninja -B build \
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
CC=/usr/bin/clang AR=llvm-ar RANLIB=llvm-ranlib ./configure --prefix=/usr --syslibdir=/lib --target=$SYSTARGET --disable-gcc-wrapper
```

Compile e instale:

```
make
make DESTDIR=$STAGE1 install
```

# • Clang (Fase 2)

Esse vai ser o nosso compilador final para o $STAGE1, esse será o compilador usado para compilar os próximos programas em $STAGE2.

# Precisa de revisão
# Modelo copiado da fase 1 apenas de placeholder

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
  -DLLVM_INCLUDE_TESTS=OFF \
  -DLLVM_INCLUDE_EXAMPLES=OFF \
  -DLLVM_ENABLE_BINDINGS=OFF \
  -DLLVM_ENABLE_OCAMLDOC=OFF \
  -DLLVM_BUILD_TOOLS=ON \
  -DLLVM_BUILD_UTILS=ON \
  -DLLVM_DEFAULT_TARGET_TRIPLE=$SYSTARGET
```

Capítulo Anterior:
[Capítulo 1 - Preparação](cap1.md)

Próximo Capítulo:
[Capítulo 3 - Ferramentas de Compilação (Fase 2)](cap3.md)
