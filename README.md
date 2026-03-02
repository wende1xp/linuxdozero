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

Agora crie os diretórios que armazenarão as toolchains temporárias isoladas do sistema host e uma pasta que será usada como raíz falsa, representando o sitema a ser construído.

```
mkdir -pv $PARDAL/{tools_{stage1,stage2},sources/{patches/{musl,llvm},pkgs},sysroot}
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
STAGE1=$PARDAL/tools_stage1
SYSROOT=$PARDAL/sysroot
SYSTARGET=x86_64-pardal-linux-musl
LC_ALL=POSIX
PATH=$STAGE1/bin:/usr/bin:/bin
export PARDAL STAGE1 SYSROOT SYSTARGET LC_ALL PATH
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
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_ENABLE_RUNTIMES="" \
  -DLLVM_INCLUDE_TESTS=OFF \
  -DLLVM_INCLUDE_EXAMPLES=OFF \
  -DLLVM_ENABLE_BINDINGS=OFF \
  -DLLVM_ENABLE_OCAMLDOC=OFF \
  -DLLVM_BUILD_TOOLS=ON \
  -DLLVM_BUILD_UTILS=ON \
  -DLLVM_DEFAULT_TARGET_TRIPLE=$SYSTARGET

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

Teste as variáveis com o seguinte comando:

```
echo 'int main(){}' > test.c
$CC test.c -v
```

A saída deve ser parecida ou igual a essa:
```
clang version 21.1.8
Target: x86_64-pardal-linux-musl
Thread model: posix
InstalledDir: /mnt/working/tools_stage1/bin
 "/mnt/working/tools_stage1/bin/clang-21" -cc1 -triple x86_64-pardal-linux-musl -emit-obj -dumpdir a- -disable-free -clear-ast-before-backend -disable-llvm-verifier -discard-value-names -main-file-name test.c -mrelocation-model pic -pic-level 2 -pic-is-pie -mframe-pointer=all -ffp-contract=on -fno-rounding-math -mconstructor-aliases -funwind-tables=2 -target-cpu x86-64 -tune-cpu generic -debugger-tuning=gdb -fdebug-compilation-dir=/mnt/working/sources/pkgs -v -fcoverage-compilation-dir=/mnt/working/sources/pkgs -resource-dir /mnt/working/tools_stage1/lib/clang/21 -isysroot /mnt/working/sysroot -internal-isystem /mnt/working/sysroot/usr/local/include -internal-externc-isystem /mnt/working/sysroot/include -internal-externc-isystem /mnt/working/sysroot/usr/include -internal-isystem /mnt/working/tools_stage1/lib/clang/21/include -ferror-limit 19 -fmessage-length=78 -fgnuc-version=4.2.1 -fskip-odr-check-in-gmf -fcolor-diagnostics -faddrsig -D__GCC_HAVE_DWARF2_CFI_ASM=1 -o /tmp/test-4c5d0f.o -x c test.c
clang -cc1 version 21.1.8 based upon LLVM 21.1.8 default target x86_64-pardal-linux-musl
ignoring nonexistent directory "/mnt/working/sysroot/usr/local/include"
ignoring nonexistent directory "/mnt/working/sysroot/include"
ignoring nonexistent directory "/mnt/working/sysroot/usr/include"
#include "..." search starts here:
#include <...> search starts here:
 /mnt/working/tools_stage1/lib/clang/21/include
End of search list.
 "/usr/bin/ld" --sysroot=/mnt/working/sysroot --hash-style=gnu --eh-frame-hdr -m elf_x86_64 -pie -dynamic-linker /lib/ld-musl-x86_64.so.1 -o a.out Scrt1.o crti.o crtbeginS.o /tmp/test-4c5d0f.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed crtendS.o crtn.o
ld: error: cannot open Scrt1.o: No such file or directory
ld: error: cannot open crti.o: No such file or directory
ld: error: cannot open crtbeginS.o: No such file or directory
ld: error: unable to find library -lgcc
ld: error: unable to find library -lgcc_s
ld: error: unable to find library -lc
ld: error: unable to find library -lgcc
ld: error: unable to find library -lgcc_s
ld: error: cannot open crtendS.o: No such file or directory
ld: error: cannot open crtn.o: No such file or directory
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

Essa saída é esperada porque não temos a biblioteca em C (musl). Remova o arquivo usado no teste:

```
rm test.c
```

# • Cabeçalhos da API do Linux 6.19.5

Certifique-se de que não existem arquivos obsoletos embutidos no pacote: 

```
make mrproper CC=clang
```

Compile os cabeçalhos usando o compilador do host e os instale na raiz falsa do sistema que estamos construindo:

```
make headers_install HOSTCC=/usr/bin/clang ARCH=x86 INSTALL_HDR_PATH=$SYSROOT/usr
```

# • Cabeçalhos do Musl

```
etc
```

```
etc
```

# • Musl

```
etc
```

```
etc
```
