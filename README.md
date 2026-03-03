# Log de Desenvolvimento

Começarei a fazer, o que chamo de base do sistema, existe termos técnicos para isso mas tanto faz. Usarei um loop device para a construção do até então "Pardal" e também estou usando o Chimera Linux para isso com certos pacotes instalados que depois os citarei, então vamos nessa.

Para começarmos fiz um script que já cria uma imagem e monta deixando no ponto de mexer, vou deixar o script nesse repositório só precisando mexer nas variáveis para o que você deseja. Com a imagem montada prosseguimos, vou me basear em uma fork mais atualizada do CMLFS (https://github.com/Xiaoyue2532/CMLFS-Legacy) e no Chimera Linux.

# **Fase 0 - Preparação**

# • Preparando o ambiente e criando o usuário builder

```
export PARDAL=/mnt/working 
```

Agora adicione um usuário para evita contaminar o sistema host e garantir builds reproduzíveis:

```
groupadd builder
useradd -s /bin/bash -g builder -m -k /dev/null builder
passwd builder
```

Crie o layout exigido de diretório emitindo os seguintes comandos:

```
mkdir -vp "$PARDAL"/sources/{patches/{musl,llvm},pkgs}
mkdir -vp "$PARDAL"/{stage1,stage2,boot,dev,proc,sys,run,tmp,home,mnt,etc,opt}
mkdir -vp "$PARDAL"/etc/init.d
mkdir -vp "$PARDAL/usr"/{bin,sbin,lib,share}
mkdir -vp "$PARDAL/var"/{log,run,cache,tmp,lib}
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
```

Baixe o arquivo source_pkgs.list desse repositório e execute o comando a seguir para baixar os pacotes. Obs.: Se o arquivo estiver em uma pasta diferente da pasta onde você está, coloque o caminho completo para o arquivo (Ex.: /home/user/Downloads/source_pkgs.list):

```
wget --input-file=source_pkgs.list --continue --directory-prefix=$PARDAL/sources/pkgs
```

Também baixe os patches:

```
wget --input-file=musl_patches.list --continue --directory-prefix=$PARDAL/sources/patches/musl
```

Agora conceda ao usuário builder as permissões adequadas:

```
chown -R builder:builder $PARDAL
```

# • Entrando no usuário builder

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

# • Aviso vindo do livro Linux From Scratch

Muitas distribuições comerciais acrescentam uma instância não documentada de /etc/bash.bashrc à inicialização do bash. Esse arquivo tem o potencial de modificar o ambiente do(a) usuário(a) builder de formas que podem afetar a construção de pacotes críticos. Para assegurar que o ambiente do(a) usuário(a) builder esteja limpo, verifique a presença de /etc/bash.bashrc e, se presente, mova-o para fora do caminho. Como o(a) usuário(a) root, execute: 

```
[ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE
```

# • Continuando a configurar o ambiente

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

Finalmente, para garantir que o ambiente esteja totalmente preparado para a construção das ferramentas temporárias, force o shell bash a ler o perfil do(a) novo(a) usuário(a):

```
source ~/.bash_profile
```

Agora iremos compilar o clang inicial para compilar os cabeçalhos do kernel posteriormente.

Entre no diretório do pacote llvm:

```
cd $PARDAL/sources/pkgs
tar -xf llvm-project-21.1.8.src.tar.xz

cd llvm-project-21.1.8.src

cmake -G Ninja \
  -S llvm \
  -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$STAGE1 \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_ENABLE_RUNTIMES="compiler-rt" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_DEFAULT_TARGET_TRIPLE="$SYSTARGET" \
  -DLLVM_RUNTIME_TARGETS="$SYSTARGET" \
  -DLLVM_INCLUDE_TESTS=OFF \
  -DLLVM_INCLUDE_EXAMPLES=OFF \
  -DLLVM_ENABLE_BINDINGS=OFF \
  -DLLVM_ENABLE_OCAMLDOC=OFF \
  -DLLVM_BUILD_TOOLS=ON \
  -DLLVM_BUILD_UTILS=ON \
  -DCOMPILER_RT_BUILD_BUILTINS=ON \
  -DCOMPILER_RT_BUILD_SANITIZERS=OFF \
  -DCOMPILER_RT_BUILD_XRAY=OFF \
  -DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
  -DCOMPILER_RT_BUILD_PROFILE=OFF \
  -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON

ninja -C build
ninja -C build install
```

# **Fase 1 - Ferramentas de Compilação Cruzada inicial**

É aqui que iremos compilar as ferramentas necessárias para tornar o sistema independente do host. Você DEVE está na pasta do pacote a ser compilado.

Antes de começarmos a compilar qualquer pacote, defina as variáveis:

```
export CC="clang --target=$SYSTARGET --sysroot=$SYSROOT"
export CXX="clang++ --target=$SYSTARGET --sysroot=$SYSROOT"
export AR="llvm-ar"
export RANLIB="llvm-ranlib"
export LD="ld.lld"
```

# • Cabeçalhos da API do Linux 6.19.5

Certifique-se de que não existem arquivos obsoletos embutidos no pacote: 

```
make mrproper CC=clang
```

Compile os cabeçalhos e os instale na raiz falsa do sistema que estamos construindo:

```
make headers_install HOSTCC=/usr/bin/clang ARCH=x86 INSTALL_HDR_PATH=$PARDAL/usr
```

# • Musl

Aplique as correções de segurança usando esse loop que aplica todas as correções:

```
for patch in $PARDAL/sources/patches/musl/*.patch; do
    echo "Aplicando $patch..."
    patch -Np1 -i < "$patch" || exit 1
done
```

```
etc
```
