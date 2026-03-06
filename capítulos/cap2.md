Agora entre com o usuário builder e configure o ambiente:

```
su - builder
```

Configure um bom ambiente de trabalho criando dois novos arquivos de inicialização para o shell bash:

```
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```

```
cat > ~/.bashrc << "EOF"
set +h
umask 022

PARDAL=/mnt/working

STAGE1=$PARDAL/stage1
STAGE2=$PARDAL/stage2

SYSTARGET=x86_64-pardal-linux-musl

LC_ALL=POSIX
PATH=$STAGE1/bin:/usr/bin:/bin

export PARDAL STAGE1 SYSTARGET LC_ALL PATH
EOF
```

Caso deseje deixar o uso de todos os núcleos explícito, execute o seguinte comando:

```
cat >> ~/.bashrc << "EOF"
export MAKEFLAGS=-j$(nproc)
EOF
```

Substitua "$(nproc)" pelo número de núcleos lógicos que você deseja usar se não quiser usar todos os núcleos lógicos.

CFLAGS e CXXFLAGS não devem ser configurados durante a construção de ferramentas cruzadas. Execute os seguinte comando para evitar isso.

```
unset CFLAGS
unset CXXFLAGS

cat >> ~/.bashrc << EOF
unset CFLAGS
unset CXXFLAGS
EOF
```

Finalmente, para garantir que o ambiente esteja totalmente preparado para a construção das ferramentas temporárias, force o shell bash a ler o perfil do novo usuário:

```
source ~/.bash_profile
```

# • Construção do compilador inicial

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

