Como usuário root e exporte uma variável apontando pro caminho que será usado para construir o sistema, nesse caso a variável será chamada de ROOTFS e o caminho será /mnt/working.

```
export ROOTFS=/mnt/working 
export TOOLS="$ROOTFS"/tools
```

Agora adicione um usuário para evita contaminar o sistema host e garantir builds reproduzíveis:

```
groupadd builder
useradd -s /bin/bash -g builder -m -k /dev/null builder
passwd builder
```

Crie o layout exigido de diretório emitindo os seguintes comandos:

```
mkdir -vp "$ROOTFS"/sources/{patches,files,pkgs}
mkdir -vp "$ROOTFS"/{boot,dev,proc,sys,run,tmp,home,mnt,etc,opt}
mkdir -vp "$ROOTFS"/etc/init.d
mkdir -vp "$ROOTFS/usr"/{bin,sbin,lib,share}
mkdir -vp "$ROOTFS/var"/{log,run,cache,tmp,lib}
mkdir -pv $TOOLS/usr/{bin,sbin,lib,include,share}
```

Faça links simbólicos:

```
ln -sv usr/bin "$ROOTFS/bin"
ln -sv usr/sbin "$ROOTFS/sbin"
ln -sv usr/lib "$ROOTFS/lib"
ln -sv lib "$ROOTFS/usr/lib64"

ln -sv usr/bin $TOOLS/bin
ln -sv usr/sbin $TOOLS/sbin
ln -sv usr/lib $TOOLS/lib
```

Baixe esse repositório e copie os patches e a lista de pacotes:

```
cd "$ROOTFS"/sources/files
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
wget --input-file="$ROOTFS"/sources/files/source_pkgs.list --continue --directory-prefix="$ROOTFS"/sources/pkgs
```

Agora conceda ao usuário builder as permissões adequadas:

```
chown -R builder:builder "$ROOTFS"
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

ROOTFS=/mnt/working

TOOLS="$ROOTFS"/tools

LC_ALL=POSIX
PATH=$TOOLS/bin:/usr/bin:/bin

export ROOTFS TOOLS LC_ALL PATH
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

Se desejar trocar o identificador, mude o 'pc' pelo seu identificador no comando a seguir e o adicione no próximo comando:

```
export SYSTARGET=x86_64-pc-linux-musl
```

```
cat >> ~/.bashrc << EOF
export SYSTARGET="$SYSTARGET"
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
cat >> ~/.bashrc << EOF
unset CFLAGS
unset CXXFLAGS
EOF
```

Finalmente, para garantir que o ambiente esteja totalmente preparado para a construção das ferramentas temporárias, force o shell bash a ler o perfil do novo usuário:

```
source ~/.bash_profile
```

Próximo Capítulo:
[Capítulo 2 - Reservado](cap2.md)