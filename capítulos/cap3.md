É aqui que iremos compilar as ferramentas necessárias para tornar o sistema independente do host usando nosso compilador inicial.

Antes de começarmos a compilar qualquer pacote, defina as variáveis com esses comandos:

```
export CC="$STAGE1/bin/clang --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld"
export CXX="$STAGE1/bin/clang++ --target=$SYSTARGET --sysroot=$STAGE1 -rtlib=compiler-rt -fuse-ld=lld"
export AR="$STAGE1/bin/llvm-ar"
export RANLIB="$STAGE1/bin/llvm-ranlib"
export LD="$STAGE1/bin/ld.lld"
export CFLAGS="-fPIC -Wno-unused-command-line-argument"
```

# • Cabeçalhos da API do Linux 6.19.5

Certifique-se de que não existem arquivos obsoletos embutidos no pacote: 

```
make mrproper CC=clang
```

Compile os cabeçalhos e os instale na raiz do sistema que estamos construindo:

```
make headers_install HOSTCC="$STAGE1/bin/clang --rtlib=compiler-rt -fuse-ld=lld" HOSTCFLAGS="--sysroot=$STAGE1" ARCH=x86_64 INSTALL_HDR_PATH=$STAGE2/usr
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
./configure --prefix=/usr --syslibdir=/lib --target=$SYSTARGET --disable-gcc-wrapper
```

Compile e instale:

```
make
make DESTDIR=$STAGE2 install
```

Agora que temos uma libc podemos compilar apontando a raíz para $STAGE2.

```
export CC="$STAGE1/bin/clang --target=$SYSTARGET --sysroot=$STAGE2 -rtlib=compiler-rt -fuse-ld=lld"
export CXX="$STAGE1/bin/clang++ --target=$SYSTARGET --sysroot=$STAGE2 -rtlib=compiler-rt -fuse-ld=lld"
export AR="$STAGE1/bin/llvm-ar"
export RANLIB="$STAGE1/bin/llvm-ranlib"
export LD="$STAGE1/bin/ld.lld"
export CFLAGS="-fPIC -Wno-unused-command-line-argument"
```

# • zlib-ng-compat (2.3.3.tar.gz)

Configure a compilação:

```
./configure --prefix=/usr --libdir=/lib --zlib-compat
```

Compile e instale:

```
make
make DESTDIR=$STAGE2 install
```

# • libatomic-chimera (v0.90.0.tar.gz)

Compile e instale:

```
make PREFIX=/usr LIBDIR=/usr/lib
make PREFIX=/usr LIBDIR=/usr/lib DESTDIR=$STAGE2 install
```

