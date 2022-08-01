# U-boot na VIM3 (SD Card)

<figure markdown>
  ![imagems](images/u-boot_logo.svg){ width="300" }
  <figcaption>U-boot logo</figcaption>
</figure>

> baseado no tutorial do próprio u-boot no [readthedocs](https://u-boot.readthedocs.io/en/latest/board/amlogic/khadas-vim3.html)

---

## Organizando a estrutura de pastas

Serão necessários vários arquivos e pastas durante esse post. Devido isso, iremos usar a seguinte estrutura de pastas

```
.
├── fip
├── toolchain
├── u-boot-mainline
└── u-boot-vim3
```

Elas serão criadas ao longo deste post.

## Baixando o u-boot Mainline

O u-boot possui um repositório no github. Vamos baixa-lo dentro da pasta ==u-boot-mainline==:

```
git clone https://github.com/u-boot/u-boot.git u-boot-mainline
```

Para fazer o [*cross compile*](https://en.wikipedia.org/wiki/Cross_compiler) é necessário baixar as [*toolchain*](https://en.wikipedia.org/wiki/Toolchain). As versões novas do u-boot precisam de versões novas de *toolchain*, que podem ser encontradas no site da [ARM](https://developer.arm.com/downloads/-/gnu-a). Nesse post estou usando a versão [gcc-arm-10.3-2021.07](https://drive.google.com/file/d/17MRLKZct7XoxGKUvNtmP1-R_l6z83PWw/view?usp=sharing) (meu link de backup).


### Compilando o u-boot mainline

Uma vez baixado a *toolchain* ([gcc-arm-10.3-2021.07](https://drive.google.com/file/d/17MRLKZct7XoxGKUvNtmP1-R_l6z83PWw/view?usp=sharing)), vamos criar a pasta ==toolchain== e mover a *toolchain* baixada para dentro dela.

Agora, precisamos descompactar a *toochain* e adicionar os binários ao *path* do sistema. Para isso

```
export PATH=CAMINHO_ABSOLUTO/toolchain/gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf/bin:$PATH
```

Observe que a pasta `/bin` está sendo adicionada ao *path*. Nela estão os executáveis que serão utilizados durante o processo de build.

Volte para a pasta do ==u-boot-mainline== e execute os seguintes comandos:

```{.sh}
make khadas-vim3_defconfig O=out/
make CROSS_COMPILE=aarch64-none-elf- O=out/
```

> O parâmetro `O` define a pasta onde os arquivos compilados serão salvos.

O argumento `CROSS_COMPILE=aarch64-none-elf-` informa qual *toolchain* será utilizada no momento da compilação, essa *toolchain* precisa estar no *path* do sistema (como fizemos anteriormente). Dentro os vários compiladores disponíveis, podemos ver que existem alguns bem conhecidos: `c++`, `cpp`, `g++` e muitos outros.


```
toolchain/gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf/bin
├── aarch64-none-elf-addr2line
├── aarch64-none-elf-ar
├── aarch64-none-elf-as
├── aarch64-none-elf-c++
├── aarch64-none-elf-c++filt
├── aarch64-none-elf-cpp
├── aarch64-none-elf-elfedit
├── aarch64-none-elf-g++
...
```

Se tudo ocorrer como o esperado, uma mensagem similar a essa:

```
aarch64-none-elf-ld.bfd: warning: -z norelro ignored
OBJCOPY lib/efi_loader/initrddump.efi
LD      u-boot
OBJCOPY u-boot.srec
OBJCOPY u-boot-nodtb.bin
RELOC   u-boot-nodtb.bin
CAT     u-boot-dtb.bin
COPY    u-boot.bin
SYM     u-boot.sym
LD      u-boot.elf
make[1]: Leaving directory 'CAMINHO_ABSOLUTO/u-boot-mainline/out'
```

O arquivo `u-boot.bin` deve ter sido gerado dentro da pasta `out`.

---

## Baixando u-boot VIM3

A [Khadas](https://www.khadas.com/) também possui um repositório com as versões do u-boot para as diversas placas que eles produzem. Vamos baixar o *brach* `khadas-vims-v2015.01` dentro da pasta ==u-boot-vim3==

```
git clone https://github.com/khadas/u-boot.git -b khadas-vims-v2015.01 u-boot-vim3
```
Para fazer o [*cross compile*](https://en.wikipedia.org/wiki/Cross_compiler) é necessário baixar as [*toolchain*](https://en.wikipedia.org/wiki/Toolchain).  Seguindo o tutorial [base](https://u-boot.readthedocs.io/en/latest/board/amlogic/khadas-vim3.html), precisaremos de 2 *toolchain* (`none-elf` e [`none-eabi`](https://en.wikipedia.org/wiki/Application_binary_interface)):

- [gcc-linaro-aarch64-none-elf-4.8-2013.11 - Linaro](https://releases.linaro.org/archive/13.11/components/toolchain/binaries/gcc-linaro-aarch64-none-elf-4.8-2013.11_linux.tar.xz)
    - [gcc-linaro-aarch64-none-elf-4.8-2013.11](https://drive.google.com/file/d/1cbF1GjMcCgsvowHB4tdbLCzDcqRGf4tM/view?usp=sharing) (backup).
- [gcc-linaro-arm-none-eabi-4.8-2013.11 - Linaro](https://releases.linaro.org/archive/13.11/components/toolchain/binaries/gcc-linaro-arm-none-eabi-4.8-2013.11_linux.tar.xz)
    - [gcc-linaro-arm-none-eabi-4.8-2013.11](https://drive.google.com/file/d/1gsffeq5i8KmYtZEcpg7jhPz7pLVVV2Wm/view?usp=sharing) (backup).


### Compilando o u-boot VIM3

> Antes de adicionar as novas *toolchain* ao *path*, limpe o terminal (abra um novo terminal). A versão utilizada no u-boot-mainline não é compatível com a versão da VIM3

Com as 2 *toolchain* baixadas, mova-as para a pastas ==toolchain== e descompacte cada uma em sua respectiva pasta. Por fim, adicionar os binários dos 2 *toolchain* ao *path* do sistema

```
export PATH=CAMINHO_ABSOLUTO/toolchain/gcc-linaro-aarch64-none-elf-4.8-2013.11_linux/bin:$PATH
export PATH=CAMINHO_ABSOLUTO/toolchain/gcc-linaro-arm-none-eabi-4.8-2013.11_linux/bin:$PATH 
```

Com os caminhos adicionados, podemos compilar o u-boot-vim3:

```{.sh}
make kvim3_defconfig O=out/
make CROSS_COMPILE=aarch64-none-elf- O=out/
```
> O parâmetro `O` define a pasta onde os arquivos compilados serão salvos.

<!-- utilizar o paramametro `O` acaba gerando um erro (`all warnings being treated as errors`) na build, por isso ele não é utilizado -->

Se tudo ocorrer bem, uma mensagem similar deve aparecer:

```
  LD      u-boot
  OBJCOPY u-boot.srec
  OBJCOPY u-boot.bin
  OBJCOPY u-boot.hex
Building board/khadas/kvim3/acs.bin

	CPP task_entry.s
	CPP user_task.lds
	CC task_entry.o
	CC user_task.o
	CPP misc.s
	CC misc.o
	CC uart.o
	CC suspend.o
	CC lib/string.o
	CC lib/delay.o
	LD CAMINHO_ABSOLUTO/u-boot-vim3/out/scp_task/bl301.out
	OBJDUMP CAMINHO_ABSOLUTO/u-boot-vim3/out/scp_task/bl301.dis
	OBJCOPY CAMINHO_ABSOLUTO/u-boot-vim3/out/scp_task/bl301.bin
```

---

## Juntando tudo

O primeiro passo é criar a pasta ==fip==.

Segundo, é preciso baixar esse [script](https://github.com/BayLibre/u-boot/releases/download/v2017.11-libretech-cc/blx_fix_g12a.sh) da [*BayLibre*](https://github.com/BayLibre/u-boot/) ou no meu [backup](https://drive.google.com/file/d/1bcYf6pl_cHXMGyBidKRMvZm38R-PAe_0/view?usp=sharing) e mover para dentro da pasta ==fip==

> Para mais informações sobre o que é, acesse:  [FIP](https://u-boot.readthedocs.io/en/latest/board/amlogic/pre-generated-fip.html) + [compilados](https://github.com/LibreELEC/amlogic-boot-fip/tree/master/khadas-vim3)

Terceiro, é preciso copiar vários arquivos para dentro da pasta ==fip==:

```
cp u-boot-vim3/build/scp_task/bl301.bin fip 
cp u-boot-vim3/build/board/khadas/kvim3/firmware/acs.bin fip

cp u-boot-vim3/fip/g12b/bl2.bin fip
cp u-boot-vim3/fip/g12b/bl30.bin fip
cp u-boot-vim3/fip/g12b/bl31.bin fip
cp u-boot-vim3/fip/g12b/ddr3_1d.fw fip
cp u-boot-vim3/fip/g12b/ddr4_1d.fw fip
cp u-boot-vim3/fip/g12b/ddr4_2d.fw fip
cp u-boot-vim3/fip/g12b/diag_lpddr4.fw fip
cp u-boot-vim3/fip/g12b/lpddr3_1d.fw fip
cp u-boot-vim3/fip/g12b/lpddr4_1d.fw fip
cp u-boot-vim3/fip/g12b/lpddr4_2d.fw fip
cp u-boot-vim3/fip/g12b/piei.fw fip
cp u-boot-vim3/fip/g12b/aml_ddr.fw fip

cp u-boot-mainline/out/u-boot.bin fip/bl33.bin

cp u-boot-vim3/fip/g12b/aml_encrypt_g12b fip 
```

### Concatenando os arquivos

Pra concatenar as informações, vamos utilizar o script da *baylibre* (dentro da pasta ==fip==)

```
bash blx_fix.sh bl30.bin zero_tmp bl30_zero.bin bl301.bin bl301_zero.bin bl30_new.bin bl30
bash blx_fix.sh bl2.bin  zero_tmp bl2_zero.bin  acs.bin   bl21_zero.bin  bl2_new.bin  bl2
```

### Encriptando os arquivos

Para mais informações, acesse: [link](https://github.com/angerman/meson64-tools)

```
./aml_encrypt_g12b --bl30sig --input bl30_new.bin            --output bl30_new.bin.g12a.enc --level v3
./aml_encrypt_g12b --bl3sig  --input bl30_new.bin.g12a.enc   --output bl30_new.bin.enc      --level v3 --type bl30
./aml_encrypt_g12b --bl3sig  --input bl31.bin                --output bl31.img.enc          --level v3 --type bl31
./aml_encrypt_g12b --bl3sig  --input bl33.bin --compress lz4 --output bl33.bin.enc          --level v3 --type bl33 --compress lz4
./aml_encrypt_g12b --bl2sig  --input bl2_new.bin             --output bl2.n.bin.sig

./aml_encrypt_g12b --bootmk --output u-boot-bin --bl2 bl2.n.bin.sig --bl30 bl30_new.bin.enc --bl31 bl31.img.enc --bl33 bl33.bin.enc --ddrfw1 ddr4_1d.fw --ddrfw2 ddr4_2d.fw --ddrfw3 ddr3_1d.fw --ddrfw4 piei.fw --ddrfw5 lpddr4_1d.fw --ddrfw6 lpddr4_2d.fw --ddrfw7 diag_lpddr4.fw --ddrfw8 aml_ddr.fw --ddrfw9 lpddr3_1d.fw --level v3
```

--- 

## Passando o u-boot para o SD Card

```
$ DEV=/dev/your_sd_device
$ dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=512 skip=1 seek=1
$ dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=1 count=444
```