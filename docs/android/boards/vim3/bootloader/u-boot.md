# U-boot

<figure markdown>
  ![qemu](images/eclipse-g564ecd7dc_640.jpg){ width="600" } 
  <figcaption>
Image by <a href="https://pixabay.com/users/ipicgr-2249158/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1492818">ipicgr</a> from <a href="https://pixabay.com//?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1492818">Pixabay</a>
</figure>


## Introdução

Com o intuito armazenar a maior quantidade de informação possível aqui no blog, percebi que ainda nao tinha registrado a forma de flashar o u-boot na vim 3. Que a princípio não é uma tarefa trivial, mas vou descrever os passos.

## Baixando U-boot

O [u-boot mainline](https://github.com/u-boot/u-boot) já possui suporte a VIM3. Devido isso, estou usando ele ao invés do fornecido pela própria [Khadas](https://github.com/khadas/u-boot). Essa escolha possui alguns impactos, pois o u-boot fornecido pela Khadas possui algumas implementações de comandos que facilitam a usabilidade, como os [comandos KBI](https://github.com/khadas/u-boot/blob/86dfffbe5bf893544d05a2433382c0ced9991df7/common/cmd_kbi.c). Em contra partida eles usam uma versão antiga do u-boot e meu foco é trabalhar no mainline.

O primeiro passo é clonar o repositório:

```
git clone https://github.com/u-boot/u-boot.git
```

## Dependências

Como a arquitetura do meu PC é `x86_64` e a VIM3 usa a `aarch64`, é necessário utilizar um `toolchain` para fazer o ==cross-compile==. Atualmente estou utilizando o [gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf.tar.xz?rev=9d9808a2d2194b1283d6a74b40d46ada&hash=4E429A41C958483C9DB8ED84B051D010F86BA624), mas talvez seja melhor baixar a versão mais atual no [site da arm](https://developer.arm.com/downloads/-/gnu-a) e procurar por ==aarch64-none-elf==.

Para facilitar, eu adiciono o caminho dos binários da `toolchain`:

```
export PATH=/home/joao/Documents/boards/VIM3/1_bootloader/Android/gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf/bin:$PATH
```

## Compilando o U-boot

> Esses passos são executados dentro da pasta u-boot

O primeiro passo da compilação é carregar o `defconfig`. Para o Android o nome do arquivo é `khadas-vim3_android_defconfig` ele se encontra dentro da pasta `configs/`. Para carregar o esse arquivo utilizo o comando:

```
make CROSS_COMPILE=aarch64-none-elf- O=../out_android/ khadas-vim3_android_defconfig
```

Caso seja necessário fazer alguma alteração nas configurações, basta utilizar o seguinte comando: 

```
make CROSS_COMPILE=aarch64-none-elf- O=../out_android/ menuconfig
```

Por fim, para compilar basta usar o comando:

```
make CROSS_COMPILE=aarch64-none-elf- O=../out_android/
```

Os arquivos compilados estarão dentro da pasta out_android.

## Adicionando assinatura

Para que a VIM3 reconheça o u-boot compilado, é necessário adicionar uma assinatura no começo do binário gerado. Para isso, podemos utilizar um script:

```
git clone https://github.com/LibreELEC/amlogic-boot-fip.git
```

Para armazenar o u-boot assinado, criei uma pasta com o nome `assinado`.

Agora, para adicionar a assinatura, utilizo o seguinte comando dentro da pasta `amlogic-boot-fip`:

```
./build-fip.sh khadas-vim3 ../out_android/u-boot.bin ../assinado
```

Ele irá buscar o u-boot compilado, adicionar a assinatura e salvar na pasta com o nome assinado.

## Estrutura das pastas

Para facilitar o entendimento das pastas, essa é a hierarquia final:

```
.
├── amlogic-boot-fip
├── assinado
├── gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf
├── out_android
└── u-boot
```

## Flashando o U-boot

É necessário um programa extra para flashar o u-boot no mmc da VIM3, que pode ser baixado nesse [link](https://github.com/khadas/utils/blob/master/aml-flash-tool/tools/linux-x86/update). No meu caso, eu coloquei o programa dentro da pasta `assinado`.

Agora é necessário colocar a VIM3 em um modo especial, para isso ela precisa estar conectado ao computador pela porta USB-C e depois apertar rapidamente 3 vezes no botão do meio ([Function Button](https://docs.khadas.com/_media/products/sbc/vim3/hardware/vim3-interface.jpg?cache=)), entre o botão de `power` e o de `reset`. O led azul da placa irá piscar rapidamente durante um espaço de tempo, informando que ele entrou nesse modo especial.

Agora, para flashar o u-boot podemos seguir os passos desse [tutorial](https://source.android.com/docs/setup/create/devices#vim3-fastboot). Adaptando para o cenário desse post, dentro da pasta `assinado`, eu utilizo os seguintes comandos:

```
./update write u-boot.bin 0xfffa0000 0x10000
./update run 0xfffa0000
./update bl2_boot u-boot.bin
```

> Talvez seja necessário ser super usuário para executar esses comandos

Depois disso, a VIM3 entrará em modo Fastboot, e então podemos executar os seguintes comandos:

```
fastboot oem format
fastboot flash bootloader u-boot.bin
fastboot erase bootenv
```

> Para instalar o fastboot: sudo apt install fastboot.

Reset a placa e pronto, o u-boot estará funcionando. Para ver os logs do u-boot é necessário usar a porta UART. 