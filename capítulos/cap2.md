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

# • LLVM (fase 1)

Esse vai ser o nosso compilador inicial, necessário para construir as dependências do compilador da fase 2.

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

Após isso remova o diretório de compilação para liberar armazenamento:

```
rm -rf build
```

Para posteriormente compilarmos a nossa biblioteca C inicial (LLVM) precisamos de bibliotecas de tempo de execução (compiler-rt). O nosso compilador possui um que podemos compilar com os seguintes comandos:

Primeiro entre na pasta do compiler-rt:

```
cd $BUILDDIR/sources/pkgs/llvm-project-21.1.8.src/compiler-rt
```

Configure a compilação:

```
cmake -G Ninja -B build \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=$STAGE1 \
 -DCMAKE_C_COMPILER=$STAGE1/bin/clang \
 -DCMAKE_AR=$STAGE1/bin/llvm-ar \
 -DCMAKE_RANLIB=$STAGE1/bin/llvm-ranlib \
 -DCOMPILER_RT_BUILD_BUILTINS=ON \
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
```

E, finalmente, remova o diretório llvm:

```
mkdir -p $STAGE1/usr/lib/clang/21/lib/x86_64-alpes-linux-musl
ln -sv $STAGE1/usr/lib/linux/libclang_rt.builtins-x86_64.a $STAGE1/usr/lib/clang/21/lib/x86_64-alpes-linux-musl/libclang_rt.builtins.a
```

Esse compilador é construído apontado para o host, sendo necessário para construir as dependências necessárias para construir um compilador isolado do host.




