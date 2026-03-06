# Log de Desenvolvimento

Começarei a fazer, o que chamo de base do sistema, existe termos técnicos para isso mas tanto faz. Usarei um loop device para a construção do até então "Pardal" e também estou usando o Chimera Linux para isso com certos pacotes instalados que depois os citarei, então vamos nessa.

Para começarmos fiz um script que já cria uma imagem e monta deixando no ponto de mexer, vou deixar o script nesse repositório só precisando mexer nas variáveis para o que você deseja. Com a imagem montada prosseguimos, vou me basear em uma fork mais atualizada do CMLFS (https://github.com/Xiaoyue2532/CMLFS-Legacy) e no Chimera Linux.

# **Fase 0 - Preparação**

Entre no usuário root e exporte uma variável apontando pro caminho que será usado para construir o sistema, nesse caso a variável será chamade de PARDAL e o caminho será /mnt/working.

```
export PARDAL=/mnt/working 
export STAGE1=/mnt/working/stage1
export STAGE2=/mnt/working/stage2
```

Agora adicione um usuário para evita contaminar o sistema host e garantir builds reproduzíveis:

```
groupadd builder
useradd -s /bin/bash -g builder -m -k /dev/null builder
passwd builder
```

Crie o layout exigido de diretório emitindo os seguintes comandos:

```
mkdir -vp "$PARDAL"/sources/{patches,files,pkgs}
mkdir -vp "$PARDAL"/{stage1,stage2,boot,dev,proc,sys,run,tmp,home,mnt,etc,opt}
mkdir -vp "$PARDAL"/etc/init.d
mkdir -vp "$PARDAL/usr"/{bin,sbin,lib,share}
mkdir -vp "$PARDAL/var"/{log,run,cache,tmp,lib}
mkdir -pv $STAGE1/usr/{bin,sbin,lib,include,share}
mkdir -pv $STAGE2/usr/{bin,sbin,lib,include,share}
```

Ajuste permissões:

```
chmod 0755 "$PARDAL"
chmod 1777 "$PARDAL/tmp" "$PARDAL/var/tmp"
```

Faça links simbólicos:

```
ln -sv usr/bin "$PARDAL/bin"
ln -sv usr/sbin "$PARDAL/sbin"
ln -sv usr/lib "$PARDAL/lib"
ln -sv lib "$PARDAL/usr/lib64"

ln -sv usr/bin $STAGE1/bin
ln -sv usr/sbin $STAGE1/sbin
ln -sv usr/lib $STAGE1/lib

ln -sv usr/bin $STAGE2/bin
ln -sv usr/sbin $STAGE2/sbin
ln -sv usr/lib $STAGE2/lib
```

Baixe esse repositório e copie os patches e a lista de pacotes:

```
cd $PARDAL/sources/files
git clone https://gitlab.com/pardal-linux/bootstrap.git
cp -rv bootstrap/patches/* ../patches
cp bootstrap/sources/source_pkgs.list .
```

Remova o repositório pois não será mais necessário:

```
rm -rf bootstrap
```

Use o arquivo source_pkgs.list para baixar todos os pacotes necessários:

```
wget --input-file=$PARDAL/sources/files/source_pkgs.list --continue --directory-prefix=$PARDAL/sources/pkgs
```

Agora conceda ao usuário builder as permissões adequadas:

```
chown -R builder:builder $PARDAL
```

# • Aviso vindo do livro Linux From Scratch

Muitas distribuições comerciais acrescentam uma instância não documentada de /etc/bash.bashrc à inicialização do bash. Esse arquivo tem o potencial de modificar o ambiente do(a) usuário(a) builder de formas que podem afetar a construção de pacotes críticos. Para assegurar que o ambiente do(a) usuário(a) builder esteja limpo, verifique a presença de /etc/bash.bashrc e, se presente, mova-o para fora do caminho. Como o(a) usuário(a) root, execute: 

```
[ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE
```

# **Fase 1 - Configurando o ambiente do usuário e construindo o compilador inicial**

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

# • Construção do compilador inicial

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

# **Fase 2 - Ferramentas de Compilação Cruzada inicial**

É aqui que iremos compilar as ferramentas necessárias para tornar o sistema independente do host usando nosso compilador inicial. Fique ciente que você DEVE está na pasta do pacote a ser compilado.

Antes de começarmos a compilar qualquer pacote, defina as variáveis com esses comandos:

```
export CC="$STAGE1/bin/clang --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -resource-dir=$STAGE1/lib/clang/21.1.8"
export CXX="$STAGE1/bin/clang++ --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -resource-dir=$STAGE1/lib/clang/21.1.8"
export AR="$STAGE1/bin/llvm-ar"
export RANLIB="$STAGE1/bin/llvm-ranlib"
export LD="$STAGE1/bin/ld.lld"
export CFLAGS="-Wno-unused-command-line-argument"
```

Ou, se preferir pode deixar as variáveis salvas no arquivo de confiração de usuário, mas modificaremos as variáveis quando chegarmos ao stage2.

```
cat >> ~/.bashrc << 'EOF'


export CC="$STAGE1/bin/clang --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -resource-dir=$STAGE1/lib/clang/21.1.8"
export CXX="$STAGE1/bin/clang++ --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -resource-dir=$STAGE1/lib/clang/21.1.8"
export AR="$STAGE1/bin/llvm-ar"
export RANLIB="$STAGE1/bin/llvm-ranlib"
export LD="$STAGE1/bin/ld.lld"
export CFLAGS="-Wno-unused-command-line-argument"


EOF
```

Se preferiu salvar as variáveis no arquivo de confiração, você precisa carregar a nova configuração para que as novas variáveis sejam carregadas. Use esse comando para carregar as novas variáveis:

```
source ~/.bash_profile
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
./configure --prefix=/usr --syslibdir=/lib --target=$SYSTARGET
```

Compile e instale:

```
make LIBCC="$STAGE1/lib/linux/libclang_rt.builtins-x86_64.a"
make DESTDIR=$STAGE1 install
```

Faça um link simbólico para evitar problemas na compilação:

```
mkdir -p $STAGE1/lib/clang/21/lib/x86_64-pardal-linux-musl
ln -sv $STAGE1/lib/linux/libclang_rt.builtins-x86_64.a $STAGE1/lib/clang/21/lib/x86_64-pardal-linux-musl/libclang_rt.builtins.a
```

Não apague o diretório do musl pois vamos usar ele na próxima fase

# **Fase 3 - Compilando as ferramentas do stage2**

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

Remova resquícios da antiga compilação:

```
make clean
```

Configure a compilação:

```
./configure --prefix=/usr --syslibdir=/lib --target=$SYSTARGET --disable-gcc-wrapper
```

Compile e instale:

```
make LIBCC="$STAGE1/lib/linux/libclang_rt.builtins-x86_64.a"
make DESTDIR=$STAGE2 install
```

# • zlib-ng-compat

Configure a compilação:

```
./configure --prefix=/usr --syslibdir=/lib --target=$SYSTARGET --disable-gcc-wrapper
```

Compile e instale:

```
make LIBCC="$STAGE1/lib/linux/libclang_rt.builtins-x86_64.a"
make DESTDIR=$STAGE2 install
```
