Entre no diretório que contém os pacotes:

```
cd $PARDAL/sources/pkgs
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
for patch in $PARDAL/sources/patches/musl/*.patch; do
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

# • Clang

Extraia o pacote e entre no diretório do pacote llvm:

```
cd $PARDAL/sources/pkgs
tar -xf llvm-project-21.1.8.src.tar.xz
cd llvm-project-21.1.8.src
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

Após isso remova o diretório de compilação para liberar armazenamento:

```
rm -rf build
```

Para posteriormente compilarmos a nossa biblioteca C inicial (LLVM) precisamos de bibliotecas de tempo de execução (compiler-rt). O nosso compilador possui um que podemos compilar com os seguintes comandos:

Primeiro entre na pasta do compiler-rt:

```
cd $PARDAL/sources/pkgs/llvm-project-21.1.8.src/compiler-rt
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
cd $PARDAL/sources/pkgs
rm -rf llvm-project-21.1.8.src
```

Esse compilador é construído apontado para o host, sendo necessário para construir as dependências necessárias para construir um compilador isolado do host.

