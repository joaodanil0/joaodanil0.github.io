# Decifrando o BOOTCMD

<figure markdown>
  ![qemu](images/magnifying-glass-g0cf482bb8_640.jpg){ width="600" } 
  <figcaption>
Image by <a href="https://pixabay.com/users/geralt-9301/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1607160">Gerd Altmann</a> from <a href="https://pixabay.com//?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1607160">Pixabay</a>
  </figcaption>
</figure>


## Introdução

Na minha saga para desvendar o mundo dos sistemas embarcados, nesse post eu pretendo ~~entender~~ explicar como o u-boot **mainline** carrega o kernel Android na ==VIM3==.

## O bootcmd

Depois que que o u-boot é carregado, ele executa comandos para encontrar o kernel. Para isso, é gerado um script para checar as possíveis formas de carrega-lo. 

O primeiro passo foi dar o comando ==printenv== no console do u-boot, resultando nas seguintes informações:

```{.sh linenums="1" hl_lines="18"}
arch=arm
avb_verify=avb init ${mmcdev}; avb verify $slot_suffix;
baudrate=115200
board=vim3
board_name=vim3
boot_a_script=load ${devtype} ${devnum}:${distro_bootpart} ${scriptaddr} ${prefix}${script}; source ${scriptaddr}
boot_efi_binary=load ${devtype} ${devnum}:${distro_bootpart} ${kernel_addr_r} efi/boot/bootaa64.efi; if fdt addr -q ${fdt_addr_r}; then bootefi ${kernel_addr_r} ${fdt_addr_r};else bootefi ${kernel_addr_r} ${fdtcontroladdr};fi
boot_efi_bootmgr=if fdt addr -q ${fdt_addr_r}; then bootefi bootmgr ${fdt_addr_r};else bootefi bootmgr;fi
boot_extlinux=sysboot ${devtype} ${devnum}:${distro_bootpart} any ${scriptaddr} ${prefix}${boot_syslinux_conf}
boot_net_usb_start=usb start
boot_pci_enum=pci enum
boot_prefixes=/ /boot/
boot_script_dhcp=boot.scr.uimg
boot_scripts=boot.scr.uimg boot.scr
boot_source=sd
boot_syslinux_conf=extlinux/extlinux.conf
boot_targets=fastboot recovery system panic 
bootcmd=run distro_bootcmd
bootcmd_fastboot=setenv run_fastboot 0;if test "${boot_source}" = "usb"; then echo Fastboot forced by usb rom boot;setenv run_fastboot 1;fi;if test "${run_fastboot}" -eq 0; then if gpt verify mmc ${mmcdev} ${partitions}; then; else echo Broken MMC partition scheme;setenv run_fastboot 1;fi; fi;if test "${run_fastboot}" -eq 0; then if bcb load 2 misc; then if bcb test command = bootonce-bootloader; then echo BCB: Bootloader boot...; bcb clear command; bcb store; setenv run_fastboot 1;elif bcb test command = boot-fastboot; then echo BCB: fastboot userspace boot...; setenv force_recovery 1;fi; else echo Warning: BCB is corrupted or does not exist; fi;fi;if test "${run_fastboot}" -eq 1; then echo Running Fastboot...;fastboot 0; fi
bootcmd_panic=fastboot 0; reset
bootcmd_recovery=pinmux dev pinctrl@14;pinmux dev pinctrl@40;setenv run_recovery 0;if run check_button; then echo Recovery button is pressed;setenv run_recovery 1;fi; if bcb load 2 misc; then if bcb test command = boot-recovery; then echo BCB: Recovery boot...; setenv run_recovery 1;fi;else echo Warning: BCB is corrupted or does not exist; fi;if test "${skip_recovery}" -eq 1; then echo Recovery skipped by environment;setenv run_recovery 0;fi;if test "${force_recovery}" -eq 1; then echo Recovery forced by environment;setenv run_recovery 1;fi;if test "${run_recovery}" -eq 1; then echo Running Recovery...;mmc dev ${mmcdev};setenv bootargs "${bootargs} androidboot.serialno=${serial#}"; if test "${force_avb}" -eq 1; then if run avb_verify; then echo AVB verification OK.;setenv bootargs "$bootargs $avb_bootargs";else echo AVB verification failed.;exit; fi;else setenv bootargs "$bootargs androidboot.verifiedbootstate=orange";echo Running without AVB...; fi;part start mmc ${mmcdev} recovery${slot_suffix} boot_start;part size mmc ${mmcdev} recovery${slot_suffix} boot_size;if mmc read ${loadaddr} ${boot_start} ${boot_size}; then echo Preparing FDT...; if test $board_name = sei510; then echo "  Reading DTB for sei510..."; setenv dtb_index 0;elif test $board_name = sei610; then echo "  Reading DTB for sei610..."; setenv dtb_index 1;elif test $board_name = vim3l; then echo "  Reading DTB for vim3l..."; setenv dtb_index 2;elif test $board_name = vim3; then echo "  Reading DTB for vim3..."; setenv dtb_index 3;else echo Error: Android boot is not supported for $board_name; exit; fi; abootimg get dtb --index=$dtb_index dtb_start dtb_size; cp.b $dtb_start $fdt_addr_r $dtb_size; fdt addr $fdt_addr_r  0x80000; if test $board_name = sei510; then echo "  Reading DTBO for sei510..."; setenv dtbo_index 0;elif test $board_name = sei610; then echo "  Reading DTBO for sei610..."; setenv dtbo_index 1;elif test $board_name = vim3l; then echo "  Reading DTBO for vim3l..."; setenv dtbo_index 2;elif test $board_name = vim3; then echo "  Reading DTBO for vim3..."; setenv dtbo_index 3;else echo Error: Android boot is not supported for $board_name; exit; fi; part start mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_start; part size mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_size; mmc read ${dtboaddr} ${p_dtbo_start} ${p_dtbo_size}; echo "  Applying DTBOs..."; adtimg addr $dtboaddr; adtimg get dt --index=$dtbo_index dtbo0_addr; fdt apply $dtbo0_addr;setenv bootargs "$bootargs androidboot.dtbo_idx=$dtbo_index ";echo Running Android Recovery...;bootm ${loadaddr} ${loadaddr} ${fdt_addr_r};fi;echo Failed to boot Android...;reset;fi
bootcmd_system=echo Loading Android boot partition...;mmc dev ${mmcdev};setenv bootargs ${bootargs} androidboot.serialno=${serial#}; if test "${force_avb}" -eq 1; then if run avb_verify; then echo AVB verification OK.;setenv bootargs "$bootargs $avb_bootargs";else echo AVB verification failed.;exit; fi;else setenv bootargs "$bootargs androidboot.verifiedbootstate=orange";echo Running without AVB...; fi;part start mmc ${mmcdev} boot${slot_suffix} boot_start;part size mmc ${mmcdev} boot${slot_suffix} boot_size;if mmc read ${loadaddr} ${boot_start} ${boot_size}; then echo Preparing FDT...; if test $board_name = sei510; then echo "  Reading DTB for sei510..."; setenv dtb_index 0;elif test $board_name = sei610; then echo "  Reading DTB for sei610..."; setenv dtb_index 1;elif test $board_name = vim3l; then echo "  Reading DTB for vim3l..."; setenv dtb_index 2;elif test $board_name = vim3; then echo "  Reading DTB for vim3..."; setenv dtb_index 3;else echo Error: Android boot is not supported for $board_name; exit; fi; abootimg get dtb --index=$dtb_index dtb_start dtb_size; cp.b $dtb_start $fdt_addr_r $dtb_size; fdt addr $fdt_addr_r  0x80000; if test $board_name = sei510; then echo "  Reading DTBO for sei510..."; setenv dtbo_index 0;elif test $board_name = sei610; then echo "  Reading DTBO for sei610..."; setenv dtbo_index 1;elif test $board_name = vim3l; then echo "  Reading DTBO for vim3l..."; setenv dtbo_index 2;elif test $board_name = vim3; then echo "  Reading DTBO for vim3..."; setenv dtbo_index 3;else echo Error: Android boot is not supported for $board_name; exit; fi; part start mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_start; part size mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_size; mmc read ${dtboaddr} ${p_dtbo_start} ${p_dtbo_size}; echo "  Applying DTBOs..."; adtimg addr $dtboaddr; adtimg get dt --index=$dtbo_index dtbo0_addr; fdt apply $dtbo0_addr;setenv bootargs "$bootargs androidboot.dtbo_idx=$dtbo_index ";setenv bootargs "${bootargs}  "  ; echo Running Android...;bootm ${loadaddr} ${loadaddr} ${fdt_addr_r};fi;echo Failed to boot Android...;
bootdelay=2
check_button=gpio input ${gpio_recovery};test $? -eq 0;
cpu=armv8
distro_bootcmd=setenv nvme_need_init; for target in ${boot_targets}; do run bootcmd_${target}; done
dtboaddr=0x08200000
efi_dtb_prefixes=/ /dtb/ /dtb/current/
ethaddr=c8:63:14:71:2d:35
fastboot_raw_partition_bootenv=0x0 0xfff mmcpart 2
fastboot_raw_partition_bootloader=0x1 0xfff mmcpart 1
fdt_addr_r=0x01000000
fdtcontroladdr=f0efa100
fdtfile=amlogic/meson-g12b-a311d-khadas-vim3.dtb
force_avb=0
gpio_recovery=88
kernel_addr_r=0x01080000
load_efi_dtb=load ${devtype} ${devnum}:${distro_bootpart} ${fdt_addr_r} ${prefix}${efi_fdtfile}
load_logo=if test "${boot_source}" != "usb" && gpt verify mmc ${mmcdev} ${partitions}; then; mmc dev ${mmcdev};part start mmc ${mmcdev} logo boot_start;part size mmc ${mmcdev} logo boot_size;if mmc read ${loadaddr} ${boot_start} ${boot_size}; then bmp display ${loadaddr} m m;fi;fi;
loadaddr=0x01080000
mmc_boot=if mmc dev ${devnum}; then devtype=mmc; run scan_dev_for_boot_part; fi
mmcdev=2
nvme_boot=run boot_pci_enum; run nvme_init; if nvme dev ${devnum}; then devtype=nvme; run scan_dev_for_boot_part; fi
nvme_init=if ${nvme_need_init}; then setenv nvme_need_init false; nvme scan; fi
partitions=uuid_disk=${uuid_gpt_disk};name=logo,start=512K,size=2M,uuid=43a3305d-150f-4cc9-bd3b-38fca8693846;name=misc,size=512K,uuid=${uuid_gpt_misc};name=dtbo,size=8M,uuid=${uuid_gpt_dtbo};name=vbmeta,size=512K,uuid=${uuid_gpt_vbmeta};name=boot,size=32M,bootable,uuid=${uuid_gpt_boot};name=recovery,size=32M,uuid=${uuid_gpt_recovery};name=cache,size=256M,uuid=${uuid_gpt_cache};name=super,size=1792M,uuid=${uuid_gpt_super};name=userdata,size=12786M,uuid=${uuid_gpt_userdata};name=rootfs,size=-,uuid=ddb8c3f6-d94d-4394-b633-3134139cc2e0;
pxefile_addr_r=0x01080000
ramdisk_addr_r=0x13000000
scan_dev_for_boot=echo Scanning ${devtype} ${devnum}:${distro_bootpart}...; for prefix in ${boot_prefixes}; do run scan_dev_for_extlinux; run scan_dev_for_scripts; done;run scan_dev_for_efi;
scan_dev_for_boot_part=part list ${devtype} ${devnum} -bootable devplist; env exists devplist || setenv devplist 1; for distro_bootpart in ${devplist}; do if fstype ${devtype} ${devnum}:${distro_bootpart} bootfstype; then run scan_dev_for_boot; fi; done; setenv devplist
scan_dev_for_efi=setenv efi_fdtfile ${fdtfile}; for prefix in ${efi_dtb_prefixes}; do if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}${efi_fdtfile}; then run load_efi_dtb; fi;done;run boot_efi_bootmgr;if test -e ${devtype} ${devnum}:${distro_bootpart} efi/boot/bootaa64.efi; then echo Found EFI removable media binary efi/boot/bootaa64.efi; run boot_efi_binary; echo EFI LOAD FAILED: continuing...; fi; setenv efi_fdtfile
scan_dev_for_extlinux=if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}${boot_syslinux_conf}; then echo Found ${prefix}${boot_syslinux_conf}; run boot_extlinux; echo SCRIPT FAILED: continuing...; fi
scan_dev_for_scripts=for script in ${boot_scripts}; do if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}${script}; then echo Found U-Boot script ${prefix}${script}; run boot_a_script; echo SCRIPT FAILED: continuing...; fi; done
scriptaddr=0x08000000
serial#=C86314712D35
soc=meson
stderr=vidconsole,serial
stdin=usbkbd,serial
stdout=vidconsole,serial
usb_boot=usb start; if usb dev ${devnum}; then devtype=usb; run scan_dev_for_boot_part; fi
vendor=amlogic

Environment size: 9785/65532 bytes
=> 
```

Estudaremos inicialmente essa saída, começando pelo ==bootcmd== (linha 18). Ele é o ponta pé inicial para o script.

Podemos observar que ele executa outra função:

```{.sh}
bootcmd=run distro_bootcmd
```

Ele executa a ==distro_bootcmd== (linha 26).

## O distro_bootcmd

Essa função é um pouco mais elaborada, quebrei em várias linhas. Vamos analizá-las

```{.sh}
distro_bootcmd=
setenv nvme_need_init; 
for target in ${boot_targets}; 
    do run bootcmd_${target}; 
done
```

A *flag* ==nvme_need_init== não está setada,  mas podemos encontrar alguns detalhes no arquivo:

> include/config_distro_bootcmd.h

No diretório do código fonte do u-boot, não vou focar nessa linha pois não é de interesse utilizar o NVME (ainda :secret:).

Seguindo no código, podemos obsevar que a variável ==boot_targets==  (atribuída na linha 17) é iterada com os seguintes valores:

```
boot_targets=fastboot recovery system panic 
```

Cada valor é concatenado com a *string* ==bootcmd_==, resultando em:

- bootcmd_fastboot
- bootcmd_recovery
- bootcmd_system
- bootcmd_panic

## As ramificações do bootcmd_

O ==distro_bootcmd== acaba executando outras 4 funções, vamos entender o que elas fazem.

### O bootcmd_fastboot

Atribuída na linha 19, ela possui a seguinte implementação:

```{.sh}
bootcmd_fastboot=
setenv run_fastboot 0;

if test "${boot_source}" = "usb"; then 
    echo Fastboot forced by usb rom boot;
    setenv run_fastboot 1;
fi;

if test "${run_fastboot}" -eq 0; then 
    if gpt verify mmc ${mmcdev} ${partitions}; then; 
    else 
        echo Broken MMC partition scheme;
        setenv run_fastboot 1;
    fi; 
fi;

if test "${run_fastboot}" -eq 0; then 
    if bcb load 2 misc; then 
        if bcb test command = bootonce-bootloader; then 
            echo BCB: Bootloader boot...; 
            bcb clear command; 
            bcb store; 
            setenv run_fastboot 1;
        elif bcb test command = boot-fastboot; then 
            echo BCB: fastboot userspace boot...; 
            setenv force_recovery 1;
        fi; 
    else 
        echo Warning: BCB is corrupted or does not exist; 
    fi;
fi;

if test "${run_fastboot}" -eq 1; then 
    echo Running Fastboot...;
    fastboot 0; 
fi
```

A *flag* ==run_fastboot== é setada para 0. A variável ==boot_source== é atribuída na linha 15:

```
boot_source=sd
```

Com isso, o primeiro `if` não é acionado. No proximo `if` já sabemos que `run_fastboot==0`, ao entrar no `if` é checado a variável `mmcdev` que é atribuída na linha 42 e a variável `partitions`, atribuída na linha 45:

```{.sh}
mmcdev=2
```

```{.sh}
partitions=
uuid_disk=${uuid_gpt_disk};
name=logo,start=512K,size=2M,uuid=43a3305d-150f-4cc9-bd3b-38fca8693846;
name=misc,size=512K,uuid=${uuid_gpt_misc};
name=dtbo,size=8M,uuid=${uuid_gpt_dtbo};
name=vbmeta,size=512K,uuid=${uuid_gpt_vbmeta};
name=boot,size=32M,bootable,uuid=${uuid_gpt_boot};
name=recovery,size=32M,uuid=${uuid_gpt_recovery};
name=cache,size=256M,uuid=${uuid_gpt_cache};
name=super,size=1792M,uuid=${uuid_gpt_super};
name=userdata,size=12786M,uuid=${uuid_gpt_userdata};
name=rootfs,size=-,uuid=ddb8c3f6-d94d-4394-b633-3134139cc2e0;
```

Logo, na linha `if gpt verify mmc ${mmcdev} ${partitions}; then;` é checado se as partições existem, nesse caso no eMMC da placa (`mmcdev=2`), caso contrário imprime a mensagem de erro `Broken MMC partition scheme;` e seta a *flag* ==run_fastboot== para 1.

Mais adiante, é checado novamente a *flag* ==run_fastboot==. Caso as partições no eMMC estejam corretas, é checado `if bcb load 2 misc; then`. Essa linha tenta carregar a partição `misc` do eMMC (mais informações nesse [link](https://u-boot.readthedocs.io/en/v2021.04/android/bcb.html)). Se tudo der certo, em seguida é checado o conteúdo da váriável `command` que foi guardada dentro da partição `misc`: 

- `if bcb test command = bootonce-bootloader; then ` 
- `elif bcb test command = boot-fastboot; then `

Caso algum problema ocorra ao carregar a partição `misc`, a seguinte mensagem é impressa na tela `Warning: BCB is corrupted or does not exist;`. 

O último `if` checa se a *flag* ==run_fastboot== é 1, caso positivo é impresso na tela a mensagem `Running Fastboot...` e entra no modo *Fastboot* com o comando `fastboot 0`.

### O bootcmd_recovery

Atribuído na linha 21 e possui a seguinte implementação:

```{.sh}
bootcmd_recovery=
pinmux dev pinctrl@14;
pinmux dev pinctrl@40;

setenv run_recovery 0;

if run check_button; then 
    echo Recovery button is pressed;
    setenv run_recovery 1;
fi; 

if bcb load 2 misc; then 
    if bcb test command = boot-recovery; then 
        echo BCB: Recovery boot...; 
        setenv run_recovery 1;
    fi;
else 
    echo Warning: BCB is corrupted or does not exist; 
fi;

if test "${skip_recovery}" -eq 1; then 
    echo Recovery skipped by environment;
    setenv run_recovery 0;
fi;

if test "${force_recovery}" -eq 1; then 
    echo Recovery forced by environment;
    setenv run_recovery 1;
fi;

if test "${run_recovery}" -eq 1; then 
    echo Running Recovery...;
    mmc dev ${mmcdev};
    setenv bootargs "${bootargs} androidboot.serialno=${serial#}"; 
    if test "${force_avb}" -eq 1; then 
        if run avb_verify; then 
            echo AVB verification OK.;
            setenv bootargs "$bootargs $avb_bootargs";
        else 
            echo AVB verification failed.;
            exit; 
        fi;
    else 
        setenv bootargs "$bootargs androidboot.verifiedbootstate=orange";
        echo Running without AVB...; 
    fi;

    part start mmc ${mmcdev} recovery${slot_suffix} boot_start;
    part size mmc ${mmcdev} recovery${slot_suffix} boot_size;

    if mmc read ${loadaddr} ${boot_start} ${boot_size}; then 

        echo Preparing FDT...; 
        if test $board_name = sei510; then 
            echo "  Reading DTB for sei510..."; 
            setenv dtb_index 0;
        elif test $board_name = sei610; then 
            echo "  Reading DTB for sei610..."; 
            setenv dtb_index 1;
        elif test $board_name = vim3l; then 
            echo "  Reading DTB for vim3l..."; 
            setenv dtb_index 2;
        elif test $board_name = vim3; then 
            echo "  Reading DTB for vim3..."; 
            setenv dtb_index 3;
        else 
            echo Error: Android boot is not supported for $board_name; 
            exit; 
        fi;

        abootimg get dtb --index=$dtb_index dtb_start dtb_size; 
        cp.b $dtb_start $fdt_addr_r $dtb_size; 
        fdt addr $fdt_addr_r  0x80000; 

        if test $board_name = sei510; then 
            echo "  Reading DTBO for sei510..."; 
            setenv dtbo_index 0;
        elif test $board_name = sei610; then 
            echo "  Reading DTBO for sei610..."; 
            setenv dtbo_index 1;
        elif test $board_name = vim3l; then 
            echo "  Reading DTBO for vim3l..."; 
            setenv dtbo_index 2;
        elif test $board_name = vim3; then 
            echo "  Reading DTBO for vim3..."; 
            setenv dtbo_index 3;
        else 
            echo Error: Android boot is not supported for $board_name; 
            exit; 
        fi; 
        
        part start mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_start; 
        part size mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_size; 
        mmc read ${dtboaddr} ${p_dtbo_start} ${p_dtbo_size}; 
        echo "  Applying DTBOs..."; 
        adtimg addr $dtboaddr; 
        adtimg get dt --index=$dtbo_index dtbo0_addr; 
        fdt apply $dtbo0_addr;
        setenv bootargs "$bootargs androidboot.dtbo_idx=$dtbo_index ";
        echo Running Android Recovery...;
        bootm ${loadaddr} ${loadaddr} ${fdt_addr_r};
    fi;
    echo Failed to boot Android...;
    reset;
fi
```

As duas primeiras linhas:

```
pinmux dev pinctrl@14;
pinmux dev pinctrl@40;
```

A príncipio não funcionam, tentei ver o status desses pinos e retornaram `pinctrl@14 not found` e `pinctrl@40 not found`. Talvez esteja faltando alguma implementação.

Seguindo no código, a *flag* ==run_recovery== é setada para 0 e então a função `check_button` é checada. Ela foi atribuída na linha 24:

```{.sh}
check_button=gpio input ${gpio_recovery};test $? -eq 0;
```
Na linha 36 a variável `gpio_recovery=88` é atribuída. Por padrão, esse *GPIO* é setado para 1 `gpio: pin 88 (gpio 88) value is 1`, logo o valor retornado é falso. Dessa forma, a comparação `if run check_button; then` não é acionada.

Mais adiante, a partição `misc` é carregada novamente e é checado se a variável `command` está setada para `boot-recovery`. Em seguinda, a váriável `skip_recovery` não está setada, logo não entra no próximo `if`.

Se a variável `force_recovery` foi setada na etapa do ==bootcmd_fastboot==, então a linha `if test "${force_recovery}" -eq 1; then ` é acionada é a variável a *flag* ==run_recovery== é setada para 1. Isso faz com que o próximo `if` seja acionado (`if test "${run_recovery}" -eq 1; then `) e começar o processo de entrar no modo *Recovery*.

Vimos na seção anterior que `mmcdev=2`. Com isso, o comando `mmc dev ${mmcdev};` ativa o eMMC. A próxima linha adiciona a variável `bootargs` a propriedade `androidboot.serialno=${serial#}`. A variável `serial#` é atribuída na linha 54:

```{.sh}
serial#=C86314712D35
```

Seguindo, é checado a váriável `force_avb=0`, atribuída na linha 35. Como o *Android Verified Boot* (AVB) não está setado, é adicionado mais uma propriedade ao `bootargs`:

```{.sh}
setenv bootargs "$bootargs androidboot.verifiedbootstate=orange";
```

Mais a frente, temos a linha:

```{.sh}
part start mmc ${mmcdev} recovery${slot_suffix} boot_start;
```

a variável `slot_suffix` não está setada. Dessa forma, essa linha procura o inicio da partição *recovery* dentro do eMMC e adiciona o valor na variável `boot_start`. A linha: 

```{.sh}
part size mmc ${mmcdev} recovery${slot_suffix} boot_size;
```

é semelhante, mas nesse caso armazena o tamanho da partição *recovery* na variável `boot_size`.

Na linha seguinte:

```{.sh}
if mmc read ${loadaddr} ${boot_start} ${boot_size}; then 
```

a variável `loadaddr=0x01080000` é atribuída na linha 40. A partição *recovery* é carregada e se tudo ocorrer corretamente, segue dentro do `if`. A variável `board_name=vim3` é atribuída na linha 5, consequentemente, setando a *flag* `setenv dtb_index 3;`.

A próxima linha:

```{.sh}
abootimg get dtb --index=$dtb_index dtb_start dtb_size; 
```

Armazena o endereço e o tamanho do *Device Tree Blob* (DTB) nas variáveis `dtb_start` e `dtb_size`, respectivamente.

Seguindo, temos:

```{.sh}
cp.b $dtb_start $fdt_addr_r $dtb_size;
```

a variável `fdt_addr_r=0x01000000` é atribuída na linha 32. A linha acima copia o conteúdo do DTB para o endereço da variável `fdt_addr_r`.


Após isso, a localização do ftd é setada para o endereço `0x80000`, pelo comando:

```{.sh}
fdt addr $fdt_addr_r  0x80000;
```

Agora é necessário carregar o *Device Tree Blob Overlay* (DTBO). Para isso, é checando novamente a varíavel `board_name`, consequentemente, a *flag* `setenv dtb_index 3` é setada. As seguintes linhas tem função similar as anteriores:

```{.sh}
part start mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_start; 
part size mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_size; 
mmc read ${dtboaddr} ${p_dtbo_start} ${p_dtbo_size}; 
```

a variável `dtboaddr=0x08200000` é atribuída na linha 27. 

As linhas:

```{.sh}
adtimg addr $dtboaddr; 
adtimg get dt --index=$dtbo_index dtbo0_addr; 
fdt apply $dtbo0_addr;
```

Seta a localização da imagem do DTBO para o endereço da variável `dtboaddr`, armazena o endereço do DTBO na variável `dtbo0_addr`. Por fim, aplica o DTBO no DTB.

Na linha seguinte, é adicionado a propriedade `setenv bootargs "$bootargs androidboot.dtbo_idx=$dtbo_index "` na variável `bootargs`.

Finalmente, na linha:

```{.sh}
bootm ${loadaddr} ${loadaddr} ${fdt_addr_r};
```

faz o boot da imagem de *recovery*.

### O bootcmd_system

Atribuído na linha 22, ele possui a seguinte implementação:

```{.sh}
bootcmd_system=

echo Loading Android boot partition...;

mmc dev ${mmcdev};
setenv bootargs ${bootargs} androidboot.serialno=${serial#}; 

if test "${force_avb}" -eq 1; then 
    if run avb_verify; then 
        echo AVB verification OK.;
        setenv bootargs "$bootargs $avb_bootargs";
    else 
        echo AVB verification failed.;
        exit; 
    fi;
else 
    setenv bootargs "$bootargs androidboot.verifiedbootstate=orange";
    echo Running without AVB...; 
fi;

part start mmc ${mmcdev} boot${slot_suffix} boot_start;
part size mmc ${mmcdev} boot${slot_suffix} boot_size;

if mmc read ${loadaddr} ${boot_start} ${boot_size}; then 
    echo Preparing FDT...; 
    if test $board_name = sei510; then
        echo "  Reading DTB for sei510..."; 
        setenv dtb_index 0;
    elif test $board_name = sei610; then 
        echo "  Reading DTB for sei610..."; 
        setenv dtb_index 1;
    elif test $board_name = vim3l; then 
        echo "  Reading DTB for vim3l..."; 
        setenv dtb_index 2;
    elif test $board_name = vim3; then 
        echo "  Reading DTB for vim3..."; 
        setenv dtb_index 3;
    else 
        echo Error: Android boot is not supported for $board_name; 
        exit; 
    fi; 
    
    abootimg get dtb --index=$dtb_index dtb_start dtb_size; 
    cp.b $dtb_start $fdt_addr_r $dtb_size; 
    fdt addr $fdt_addr_r  0x80000; 
    
    if test $board_name = sei510; then 
        echo "  Reading DTBO for sei510..."; 
        setenv dtbo_index 0;
    elif test $board_name = sei610; then
        echo "  Reading DTBO for sei610...";
        setenv dtbo_index 1;
    elif test $board_name = vim3l; then 
        echo "  Reading DTBO for vim3l..."; 
        setenv dtbo_index 2;
    elif test $board_name = vim3; then
        echo "  Reading DTBO for vim3..."; 
        setenv dtbo_index 3;
    else 
        echo Error: Android boot is not supported for $board_name; 
        exit; 
    fi; 
    
    part start mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_start; 
    part size mmc ${mmcdev} dtbo${slot_suffix} p_dtbo_size; 
    mmc read ${dtboaddr} ${p_dtbo_start} ${p_dtbo_size}; 
    echo "  Applying DTBOs...";
    adtimg addr $dtboaddr;
    adtimg get dt --index=$dtbo_index dtbo0_addr; 
    fdt apply $dtbo0_addr;
    setenv bootargs "$bootargs androidboot.dtbo_idx=$dtbo_index ";
    setenv bootargs "${bootargs}  "  ; 
    
    echo Running Android...;
    
    bootm ${loadaddr} ${loadaddr} ${fdt_addr_r};
fi;

echo Failed to boot Android...;
```

Todo o processo é o mesmo do modo *recovery* a diferença fica nas seguintes linhas:

```{.sh}
part start mmc ${mmcdev} boot${slot_suffix} boot_start;
part size mmc ${mmcdev} boot${slot_suffix} boot_size;
```

Observe que agora o script busca pela partição `boot`. O resto segue inalterado.

### O bootcmd_panic

Atribuído na linha 20, ele possui a seguinte implementação:

```{.sh}
bootcmd_panic=fastboot 0; reset
```

Como último recurso, caso não tenha sido possível fazer o boot do Android. O script entra no modo fastboot e ao sair ele reinicia a VIM3.

## Conclusão

Sem dúvida entender esse processo não é algo trivial, não existe uma padronização e tudo fica na mão do desenvolvedor na hora de criar o script responsável por carregar o kernel do Android. Com certeza, esse passo foi fundamental para um melhor entendimento de como o Android funciona.