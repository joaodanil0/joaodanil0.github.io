# U-Boot na Beaglebone

<figure markdown>
  ![qemu](../images/bbb_soc.jpg){ width="600" } 
  <figcaption>
  Image by <a href="https://pixabay.com/users/christoph1703-633357/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4753037">christoph1703</a> from <a href="https://pixabay.com//?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4753037">Pixabay</a>
  </figcaption>
</figure>



## Introdução

Dando prosseguimento na minha saga por bootloader para embarcados, vou postar meus passos para usar o U-Boot com SD-Card.

> Baseado no curso da *Udemy* -> [`Embedded Linux Step by Step Using Beaglebone Black`](https://www.udemy.com/share/101X4W3@N0l6s9l1hoIcz2r0Zt6A8C0udGIlHPqNhW2BPV-FlD-zHXPTp5xIIMh4rNfdLAFg/)

## Baixando

O repositório oficial do U-Boot é o da denx:

```{.sh}
git clone https://source.denx.de/u-boot/u-boot.git
```

eles também mantém um espelho no github

```{.sh}
git clone https://github.com/u-boot/u-boot.git
```

Como a maioria das placas, os criadores portaram o U-Boot para a Beaglebone
```{.sh}
git clone https://github.com/beagleboard/u-boot.git
```

## Buildando

Primeiro, foi preciso baixar o *toolchain*

```{.sh}
sudo apt install gcc-arm-linux-gnueabihf
```

Tanto o *U-Boot mainline* quanto o *U-Boot beaglebone* possuem o mesmo nome de arquivo para as configurações padrões:

```
am335x_evm_defconfig
```

No meu caso, eu utilizei o *U-boot mainline*.

Dessa forma, para criar o `.config` usei o comando:

```{.sh}
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- am335x_evm_defconfig
```
> OBS: para limpar o ambiente compilação, basta usar o comando `make distclean`

Por fim, para gerar as imagens, utilizei o comando:

```{.sh}
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16
```
> OBS: -j é o parâmetro que define a quantidade de núcleos de processador utilizado no processo de compilação.

Se tudo ocorrer bem, uma mensagem semelhante a essa deve aparecer: 

```{.sh hl_lines="1 11"}
  MKIMAGE MLO
  MKIMAGE MLO.byteswap
  DTC     arch/arm/dts/am335x-pdu001.dtb
  DTC     arch/arm/dts/am335x-chiliboard.dtb
  DTC     arch/arm/dts/am335x-sl50.dtb
  DTC     arch/arm/dts/am335x-base0033.dtb
  DTC     arch/arm/dts/am335x-guardian.dtb
  DTC     arch/arm/dts/am335x-wega-rdk.dtb
  DTC     arch/arm/dts/am335x-regor-rdk.dtb
  SHIPPED dts/dt.dtb
  MKIMAGE u-boot.img
  CAT     u-boot-dtb.bin
  COPY    u-boot.dtb
  MKIMAGE u-boot-dtb.img
  COPY    u-boot.bin
```

Mostrando que os binários `MLO` e `u-boot.img` foram criados.

## Estágios 

A forma como a ==Texas Instruments== desenvolveu o SOC ==AM335x== ([mais detalhes](https://youtu.be/DV5S_ZSdK0s?t=1357)), se faz necessário que o processo de *boot* da placa precise de 3 estágios

1. BROM - que é definido pela própria ==Texas Instruments== (não é possível modifica-lo)
2. SPL ou *Secondary Program Loader* - No U-boot chamado de `MLO` ou *Memory LOader*
3. O U-boot em si (`u-boot.img`)

Para maiores detalhes, checar o [*Technical Reference manual*](https://www.ti.com/lit/ug/spruh73q/spruh73q.pdf) (TRM).

> No TRM também é possível verificar que existe a possibilidade de fazer o boot sem um sistema de arquivos no SD-Card/eMMC, na seção `26.1.8.5.5 MMC/SD Read Sector Procedure in Raw Mode`

### BROM

O ==BROM== faz várias checagens no hardware, mas o que nos interessa nesse estágio é o que ele busca após finalizar sua execução. O TRM encontramos a seguinte informação

```{.txt}
The next task for the ROM Code is to find the booting file named “MLO” inside 
the Root Directory of the FAT12/16/32 file system. The file is not searched 
in any other location.
```

Ou seja, no nosso SD-Card/eMMC precisamos de uma partição do tipo ==FAT12==, ==FAT16== ou ==FAT32==. Dentro da partição, um arquivo chamado `MLO`.

### MLO

Resumidamente, o MLO é responsável por "habilitar" a memória ram (DRAM) e carregar a imagem do U-Boot para a mesma.

Um primeiro teste foi colocar apenas o `MLO` no SD-Card (sem o `u-boot.img`) e ver o que aconteceria. E essa foi a mensagem que retornou:

```{hl_lines="3"}
U-Boot SPL 2022.04-rc3-00052-g6d3c46ed0e (Sep 11 2022 - 11:17:49 -0400)         
Trying to boot from MMC1                                                        
spl_load_image_fat: error reading image u-boot.img, err - -2                    
SPL: failed to boot from all boot devices                                       
### ERROR ### Please RESET the board ### 
```

> OBS: Para visualizar o logs de boot, cheque esse [post](../USB%20Serial.md)

Uma mensagem dizendo não foi possível carregar o `u-boot.img`. Vale ressaltar que esses logs são do `MLO`, podemos observar referências ao *SPL* nos logs.

Isso já mostra os primeiros sinais de vida da Beaglebone, mostrando que o SD-Card foi formatado corretamente e a placa está conseguindo detectar o mesmo. Com essas confirmações, podemos seguir em frente.

### U-boot.img

Copiando o binário `u-boot.img` para o SD-Card é possível ver o u-boot carregado, como podemos ver abaixo:
```
U-Boot SPL 2022.04-rc3-00052-g6d3c46ed0e (Sep 11 2022 - 11:17:49 -0400)
Trying to boot from MMC1


U-Boot 2022.04-rc3-00052-g6d3c46ed0e (Sep 11 2022 - 11:17:49 -0400)

CPU  : AM335X-GP rev 2.1
Model: TI AM335x BeagleBone Black
DRAM:  512 MiB
Core:  150 devices, 14 uclasses, devicetree: separate
WDT:   Started wdt@44e35000 with servicing (60s timeout)
NAND:  0 MiB
MMC:   OMAP SD/MMC: 0, OMAP SD/MMC: 1
Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1... 
<ethaddr> not set. Validating first E-fuse MAC
Net:   eth2: ethernet@4a100000, eth3: usb_ether
Hit any key to stop autoboot:  0 
```

Dessa forma, consegui compilar e rodar o U-Boot na minha Beaglebone.

## Próximos passos

Para o próximo desafio, fica carregar o kernel Linux. Procurando rapidamente, percebi o u-boot deixou de dar suporte ao arquivo ==uEnv.txt==, o que vai dificultar as coisas um pouco mais. Algumas fontes:

- https://forum.beagleboard.org/t/u-boot-ignores-uenv-txt/30549/3

Possivelmente esse commit desabilitou o ==uEnv.txt== :clown_face:

```
https://github.com/u-boot/u-boot/commit/ff8f277e9121c6636e21bb7d7381c4dcac2a596b\
```

Mas também encontrei, que existe uma outra forma de carregar as imagens do `Kernel` e o `DTB`. Utilizando o *Distro Boot* ([ref](https://www.jstuber.net/2021/08/05/distro-boot-with-buildroot-on-a-beaglebone-black/))

Além de fazer da forma tradicional, que é criando o próprio script de boot ([ref](https://sergioprado.org/utilizando-o-u-boot-na-raspberry-pi/))

```
cat <<"EOF" > boot.cmd
mmc dev 0
fatload mmc 0:1 ${kernel_addr_r} zImage
setenv bootargs console=tty1 console=ttyAMA0,115200 earlyprintk root=/dev/mmcblk0p2 rootwait
bootz ${kernel_addr_r} - ${fdt_addr}
EOF
```
```
tools/mkimage -C none -A arm -T script -d boot.cmd boot.scr
```