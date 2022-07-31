# U-boot na VIM3

<figure markdown>
  ![imagems](images/u-boot_logo.svg){ width="600" }
  <figcaption>U-boot logo</figcaption>
</figure>


baseado no tutorial do pŕoprio u-boot em [readthedocs](https://u-boot.readthedocs.io/en/latest/board/amlogic/khadas-vim3.html)


---

## Main line

Baixar o u-boot mainline

```
git clone https://github.com/u-boot/u-boot.git u-boot-mainline
```

baixei o toolchain [versao ](https://drive.google.com/file/d/17MRLKZct7XoxGKUvNtmP1-R_l6z83PWw/view?usp=sharing), para versões mais atuais checar em [toolchain](https://developer.arm.com/downloads/-/gnu-a)

Precisa adicionar o path do arquivo descompactado ao $PATH do linux

execute o seguinte comando dentro da pasta descompactada

```{.sh linenums="1" title="Buildando o u-boot"}
make khadas-vim3_defconfig O=out/
make CROSS_COMPILE=aarch64-none-elf- O=out/
```

O parâmetro `O` define a pasta onde os arquivos gerados serão salvos.

---

# uboot modificado VIM3

Baixar 

```
git clone https://github.com/khadas/u-boot.git -b khadas-vims-v2015.01 u-boot-vim3
```

baixei a toolchain [linaro](https://releases.linaro.org/archive/13.11/components/toolchain/binaries/gcc-linaro-aarch64-none-elf-4.8-2013.11_linux.tar.xz) ou no meu backup [backup](https://drive.google.com/file/d/1cbF1GjMcCgsvowHB4tdbLCzDcqRGf4tM/view?usp=sharing)

outra toolchain [linaro](https://releases.linaro.org/archive/13.11/components/toolchain/binaries/gcc-linaro-arm-none-eabi-4.8-2013.11_linux.tar.xz) ou meu backup [backup](https://drive.google.com/file/d/1gsffeq5i8KmYtZEcpg7jhPz7pLVVV2Wm/view?usp=sharing)

Precisa adicionar o path dos arquivos descompactados ao $PATH do linux

```{.sh linenums="1" title="Buildando o u-boot VIM3"}
make kvim3_defconfig
make CROSS_COMPILE=aarch64-none-elf-
```

utilizar o paramametro `O` acaba gerando um erro (`all warnings being treated as errors`) na build, por isso ele não é utilizado

---

juntando tudo

Primeiro é preciso baixar esse [script](https://github.com/BayLibre/u-boot/releases/download/v2017.11-libretech-cc/blx_fix_g12a.sh) da [*BayLibre*](https://github.com/BayLibre/u-boot/) ou meu [backup](https://drive.google.com/file/d/1bcYf6pl_cHXMGyBidKRMvZm38R-PAe_0/view?usp=sharing)

mais informações sobre a [FIP](https://u-boot.readthedocs.io/en/latest/board/amlogic/pre-generated-fip.html)
+ [compilados](https://github.com/LibreELEC/amlogic-boot-fip/tree/master/khadas-vim3)


agora precisamos copiar os arquivos

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

As pastas ficaram organizadas assim:

```
.
├── fip
├── toolchains
├── u-boot-mainline
└── u-boot-vim3
```

---

concatenando os arquivos

```
bash blx_fix.sh bl30.bin zero_tmp bl30_zero.bin bl301.bin bl301_zero.bin bl30_new.bin bl30
bash blx_fix.sh bl2.bin zero_tmp bl2_zero.bin acs.bin bl21_zero.bin bl2_new.bin bl2
```

Encriptando -> mais informações [link](https://github.com/angerman/meson64-tools)

```
./aml_encrypt_g12b --bl30sig --input bl30_new.bin            --output bl30_new.bin.g12a.enc --level v3
./aml_encrypt_g12b --bl3sig  --input bl30_new.bin.g12a.enc   --output bl30_new.bin.enc      --level v3 --type bl30
./aml_encrypt_g12b --bl3sig  --input bl31.bin                --output bl31.img.enc          --level v3 --type bl31
./aml_encrypt_g12b --bl3sig  --input bl33.bin --compress lz4 --output bl33.bin.enc          --level v3 --type bl33 --compress lz4
./aml_encrypt_g12b --bl2sig  --input bl2_new.bin             --output bl2.n.bin.sig

./aml_encrypt_g12b --bootmk --output u-boot-bin --bl2 bl2.n.bin.sig --bl30 bl30_new.bin.enc --bl31 bl31.img.enc --bl33 bl33.bin.enc --ddrfw1 ddr4_1d.fw --ddrfw2 ddr4_2d.fw --ddrfw3 ddr3_1d.fw --ddrfw4 piei.fw --ddrfw5 lpddr4_1d.fw --ddrfw6 lpddr4_2d.fw --ddrfw7 diag_lpddr4.fw --ddrfw8 aml_ddr.fw --ddrfw9 lpddr3_1d.fw --level v3
```

--- 

# escrever no sd card

```
$ DEV=/dev/your_sd_device
$ dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=512 skip=1 seek=1
$ dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=1 count=444
```