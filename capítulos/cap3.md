É aqui que iremos compilar as ferramentas necessárias para tornar o sistema independente do host usando nosso compilador inicial.

Antes de começarmos a compilar qualquer pacote, defina as variáveis com esses comandos:

```
export CC="$STAGE1/bin/clang --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld"
export CXX="$STAGE1/bin/clang++ --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld"
export AR="$STAGE1/bin/llvm-ar"
export RANLIB="$STAGE1/bin/llvm-ranlib"
export LD="$STAGE1/bin/ld.lld"
export CFLAGS="-Wno-unused-command-line-argument"
```

Ou, se preferir pode deixar as variáveis salvas no arquivo de configuração do usuário, porém, depois teremos que modificar as variáveis quando trocar para a suíte de ferramentas do stage2.

```
cat >> ~/.bashrc << 'EOF'


export CC="$STAGE1/bin/clang --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld
export CXX="$STAGE1/bin/clang++ --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld
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

