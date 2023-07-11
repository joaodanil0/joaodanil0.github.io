# Buildando o Linux

<figure markdown>
  ![qemu](images/linux.png){ width="600" } 
  <figcaption>
    Image by <a href="https://pixabay.com/users/openclipart-vectors-30363/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=159784">OpenClipart-Vectors</a> from <a href="https://pixabay.com//?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=159784">Pixabay</a>
  </figcaption>
</figure>

## Introdução

Com o intuito de dar uma sequência ao post [Habilitando o env.txt com tfpt](Bootloader/env.md), vamos seguindo com a build do kernel Linux para a beaglebone. Atualmente tem suas peculiaridades devido sua arquitetura de 32 bits (64 bits se tornou padrão em SBCs). Muitos outros artigos podem ser encontrados sobre isso, mas sua grande maioria são antigos e de lá pra cá algumas coisas mudaram.

## Setup

Para tentar tornar os passos reproduzíveis, segue o setup que eu utilizei para a build:

- Distro: Linux Mint 21.1
- Kernel (**host**): Linux 6.3.7-060307-generic 
- Toolchain: [gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads/10-3-2021-07)
- Kernel (**target**): Linux-mainline-6.1

## Download

Basta utilizar o comando:

```
git clone https://github.com/torvalds/linux.git -b v6.1
```

## Build

!!! warning "Aviso"
    Não esqueça de adicionar o caminho dos binários da *toolchain* nas variáveis de ambiente. No meu caso: <br>
    export PATH=/home/joao/Documents/2.Linux/toolchains/gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf/bin:$PATH

O processo de build pode consistir em 3 passos básicos.

Selecionando o `defconfig`:

```
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- -j16 O=out/ omap2plus_defconfig
```

Habilitando drivers com o `menuconfig`:

```
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- -j16 O=out/ menuconfig
```

Finalmente a build em si:

```
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- -j16 O=out/
```



## Os Binários

Uma vez que o processo de build finalizar, eles estarão disponíveis dentro dos caminhos:

```
out/arch/arm/boot
out/arch/arm/boot/dts
```
Os binários de interesse são `out/arch/arm/boot/zImage` e `out/arch/arm/boot/dts/am335x-boneblack.dtb`. Basta copiá-los para dentro da pasta onde fica o arquivo `env.txt` no servidor de tftp ou na partição `boot` do cartão SD.

## Conclusão

Estamos buildando uma das versão mais recentes do Kernel Linux atualmente. Dando uma sobre vida maior a beaglebone, o que torna ela ainda muito útil para o aprendizado de sistemas embarcados (lembrando que ela é *open hardware*)