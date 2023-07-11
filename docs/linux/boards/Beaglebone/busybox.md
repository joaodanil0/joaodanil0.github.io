# Buildando BusyBox


<figure markdown>
  ![qemu](images/busybox1.png){ width="600" } 
  <figcaption>
      Image by <a href="https://busybox.net/">Busybox</a> from <a href="https://busybox.net/">Busybox</a>
  </figcaption>
</figure>

## Introdução

Seguindo com os tutoriais [Habilitando o env.txt com tfpt](Bootloader/env.md) e [Buildando o Linux](Buildando.md), vamos compilar o busybox para termos um **Root File System** (RFS) mínimo. Para entendermos o que realmente é necessário para termos um distro Linux.

## Setup

- Distro: Linux Mint 21.1
- Kernel: Linux 6.3.7-060307-generic 
- Toolchain: [gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads/10-3-2021-07)
- Busybox : busybox-1.33.2

## Download 

Para baixar o busybox podemos utilizar o comando:

```
wget https://busybox.net/downloads/busybox-1.33.2.tar.bz2
```

Para extrair:

```
tar -xvjf busybox-1.33.2.tar.bz2
```

## Build

!!! warning "Aviso"
    Não esqueça de adicionar o caminho dos binários da *toolchain* nas variáveis de ambiente. No meu caso: <br>
    export PATH=/home/joao/Documents/4.Busybox/toolchains/gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf/bin:$PATH

O processo de build pode consistir em 3 passos básicos.

Selecionando o `defconfig`:

```
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- -j16 O=out/ defconfig
```

Habilitando recursos com o menuconfig:

```
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- -j16 O=out/ menuconfig
```

!!! tip "Dica"
    Ativar a compilação das bibliotecas de forma estática: <br>
    Settings -> Build static binary (no shared libs)

Finalmente a build em si:

```
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- -j16 O=out/ CONFIG_PREFIX=$PWD/../RFS install
```

O comando acima irá compilar o busybox e depois copiar  os binários e hierarquia de pastas para o diretório passado na variável `CONFIG_PREFIX`

## Conclusão

O busybox é o canivete suíço de sistemas embarcados, ele é muito utilizado em roteadores (com as devidas modificações). Além disso o grub também utiliza o busybox. 


