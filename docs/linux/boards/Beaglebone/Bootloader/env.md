# Habilitando o env.txt com tfpt

<figure markdown>
  ![qemu](images/bbb_soc.jpg){ width="600" } 
  <figcaption>
 Image by <a href="https://pixabay.com/users/christoph1703-633357/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=4753037">christoph1703</a> from <a href="https://pixabay.com//?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=4753037">Pixabay</a>
  </figcaption>
</figure>

## Introdução

Nas versões mais atuais do u-boot não é possível usar nativamente o arquivo `env.txt`, que facilitava a criação do script de inicialização do bootloader. Essa abordagem era bem útil pois não era necessário recompilar o u-boot para mudar o script de execução. Com o intuito apenas de facilitar os testes na placa, acabei reabilitando esse arquivo.

!!! danger "Cuidado"
    
    Esse post é apenas para fins de teste. Essa abordagem teve motivos para ser descontinuada. <br>
    Use por sua conta e risco!

## Setup

Para tentar aumentar as chances de reprodução dos passos, segue o setup utilizado:

- Distro: Linux Mint 21.1
- Kernel: Linux 6.3.7-060307-generic
- Toolchain: [gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads/10-3-2021-07)
- U-boot: [u-boot-mainline-2023.04](https://github.com/u-boot/u-boot/tree/v2023.04)

## Modificação

Existem várias formas de habilitar o `env.txt`, uma delas é alterando a variável `CONFIG_BOOTCOMMAND` no defconfig `am335x_evm_defconfig`. Por padrão essa variável vem com os seguintes comandos:

```
CONFIG_BOOTCOMMAND="run findfdt; run init_console; run finduuid; run distro_bootcmd"
```

Ela executa as funções `findfdt`, `init_console`, `finduuid` e `distro_bootcmd` respectivamente.

!!! info
    Para saber onde essas funções foram criadas é necessário dar uma olhada mais a fundo no arquivo: <br>
    `include/configs/am335x_evm.h` e seus *includes*.

Para reativar o `env.txt`, podemos fazer a seguinte alteração:

```
CONFIG_BOOTCOMMAND="setenv serverip 192.168.0.203; setenv ipaddr 192.168.0.104; if tftpboot 0x80000000 env.txt; then env import -t 0x80000000 ${filesize}; run agora_vai; fi;"
```

Para facilitar o entendimento do código, segue o mesmo indentado:

``` linenums="1"
setenv serverip 192.168.0.203; 
setenv ipaddr 192.168.0.104; 
if tftpboot 0x80000000 env.txt; then 
    env import -t 0x80000000 ${filesize}; 
    run agora_vai; 
fi;
```

- Na linha 1: É setado a variável de ambiente `serverip` que será o ip do servidor tftp.
- Na linha 2: É setados a variável de ambiente `ipaddr` que será o ip da placa.
- Na linha 3: O arquivo `env.txt` é buscado no servidor e armazenado na memória ram da placa no endereço 0x80000000.
- Na linha 4: Se tudo ocorrer corretamente, o u-boot carrega o conteúdo do arquivo `env.txt` nas variáveis de ambiente.
- Na linha 5: Executa a função `agora_vai` que foi escrita dentro do arquivo `env.txt`.

## O arquivo env.txt

Até o momento o u-boot está preparado para buscar no servidor tftp um arquivo com o nome `env.txt`. Agora, vamos checar o conteúdo desse arquivo:

``` linenums="1"
boot_args=setenv bootargs "console=ttyO0,115200n8 root=/dev/nfs \
          ip=192.168.0.104 \
          nfsroot=192.168.0.203:/home/joao/Documents/NFS,nfsvers=4 rw rootwait \
          loglevel=8"

load_kernel=tftpboot 0x82000000 zImage
load_dtb=tftpboot 0x88000000 am335x-boneblack.dtb
boot_command=bootz 0x82000000 - 0x88000000

agora_vai=run boot_args load_kernel load_dtb boot_command
```

A função `boot_args` adiciona algumas variáveis de ambiente que serão passadas ao kernel, relacionadas a *Network File System* (NFS). A função `load_kernel` busca no servidor tftp pelo arquivo ==zImage=== (imagem do kernel) e carrega no endereço de memória 0x82000000. Em seguida a função `load_dtb` carrega no endereço de memória 0x88000000 o arquivo ==am335x-boneblack.dtb== (*device tree blob*). Em `boot_command` o u-boot executa o que está nas regiões de memória 0x82000000 (zImage) e 0x88000000 (dtb). Por fim, a função `agora_vai` executa as funções descritas acima.

Perceba que esse arquivo não é executado imediatamente, ele apenas é carregado em memória. O script só será executado de fato, quando a função `agora_vai` for invocada (descrita na seção anterior na linha 5).

## Conclusão

Se tudo ocorrer corretamente o kernel Linux será carregado e executado, mas se o NFS ainda não estiver configurado irá ocorrer um *kernel panic* devido não ser possível encontrar o `init`. 





