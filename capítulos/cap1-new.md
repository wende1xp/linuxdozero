Como usuário root adicione um usuário para evita contaminar o sistema host e garantir builds reproduzíveis:

```
groupadd builder
useradd -s /bin/bash -g builder -m -k /dev/null builder
passwd builder
```

Renomeie esse o arquvio bash.bashrc para evitar alguma chance de alguma instância não documentada afetar as compilações ##Mudar depois
```
[ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE
```

Agora entre com o novo usuário:

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

BUILDDIR=/mnt/working

STAGE1=$BUILDDIR/stage1
STAGE2=$BUILDDIR/stage2

LC_ALL=POSIX
PATH=$STAGE1/bin:/usr/bin:/bin

export BUILDDIR STAGE1 LC_ALL PATH
EOF
```

Definiremos uma variável de ambiente contendo o identificador da plataforma alvo (target triple). Esse identificador segue um formato padronizado que descreve arquitetura, fornecedor, sistema operacional e biblioteca C utilizada. Observe o exemplo de formato a seguir:

  `x86_64-pc-linux-musl`

*x86_64* → arquitetura da CPU

*pc* → identificador do fornecedor ou da distribuição

*linux* → sistema operacional alvo

*musl* → implementação da biblioteca padrão C utilizada

Você pode trocar o identificador do fornecedor pelo seu próprio, como no exemplo a seguir:

  `x86_64-pardal-linux-musl`

Se desejar trocar o identificador, mude o 'pc' pelo seu identificador no comando a seguir:

```
cat >> ~/.bashrc << "EOF"
export SYSTARGET=x86_64-pc-linux-musl
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

Crie o layout exigido de diretório emitindo os seguintes comandos:

```
mkdir -vp "$BUILDDIR"/sources/{patches,files,pkgs}
mkdir -vp "$BUILDDIR"/{stage1,stage2,boot,dev,proc,sys,run,tmp,home,mnt,etc,opt}
mkdir -vp "$BUILDDIR"/etc/init.d
mkdir -vp "$BUILDDIR/usr"/{bin,sbin,lib,share}
mkdir -vp "$BUILDDIR/var"/{log,run,cache,tmp,lib}
mkdir -pv $STAGE1/usr/{bin,sbin,lib,include,share}
mkdir -pv $STAGE2/usr/{bin,sbin,lib,include,share}
```

Ajuste permissões:

```
chmod 1777 "$BUILDDIR/tmp" "$BUILDDIR/var/tmp"
```

Faça links simbólicos:

```
ln -sv usr/bin "$BUILDDIR/bin"
ln -sv usr/sbin "$BUILDDIR/sbin"
ln -sv usr/lib "$BUILDDIR/lib"
ln -sv lib "$BUILDDIR/usr/lib64"

ln -sv usr/bin $STAGE1/bin
ln -sv usr/sbin $STAGE1/sbin
ln -sv usr/lib $STAGE1/lib

ln -sv usr/bin $STAGE2/bin
ln -sv usr/sbin $STAGE2/sbin
ln -sv usr/lib $STAGE2/lib
```

Baixe esse repositório e copie os patches e a lista de pacotes:

```
cd $BUILDDIR/sources/files
git clone https://github.com/wende1xp/linuxdozero.git
cp -rv linuxdozero/patches/* ../patches
cp linuxdozero/sources/source_pkgs.list .
```

Remova o repositório pois não será mais necessário:

```
rm -rf linuxdozero
```

Use o arquivo source_pkgs.list para baixar todos os pacotes necessários:

```
wget --input-file=$BUILDDIR/sources/files/source_pkgs.list --continue --directory-prefix=$BUILDDIR/sources/pkgs
```


Próximo Capítulo:
[Capítulo 2 - Ferramentas de Compilação (Fase 1)](cap2.md)




