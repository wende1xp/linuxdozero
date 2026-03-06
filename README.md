# **Cap. 1 - Preparação**

Como usuário root e exporte uma variável apontando pro caminho que será usado para construir o sistema, nesse caso a variável será chamade de PARDAL e o caminho será /mnt/working.

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

Renomeie esse o arquvio bash.bashrc para evitar alguma chance de alguma instância não documentada afetar as compilações ##Mudar depois
```
[ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE
```
