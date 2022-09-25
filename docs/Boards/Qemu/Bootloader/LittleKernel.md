# LittleKernel


## Introdução

Recentemente descobri que grandes empresas utilizam uma alternativa ao `U-Boot` para *bootloader*. Por isso, resolvi escrever esse post sobre o `LittleKernel`. Ele não tem a popularidade do `U-Boot`, mas é tão antigo quanto.

Devido sua impopularidade, vamos primeiro seguir o tutorial que utiliza o ==QEMU==. Para que tenhamos um ambiente minimamente funcional.

## Baixando o LittleKernel

O [`LittleKernel`](https://github.com/littlekernel/lk) está hospedado no github. Para baixa-lo, basta utilizar o comando:

```{.sh}
git clone https://github.com/littlekernel/lk.git
```

Checando o diretórios com o comando `tree -L 1 .`, temos a seguinte configuração:

```{.sh}  
.
├── app
├── arch
├── build-qemu-virt-arm32-test
├── dev
├── docs
├── engine.mk
├── external
├── kernel
├── lib
├── LICENSE
├── lk_inc.mk.example
├── make
├── makefile
├── platform
├── project
├── README.md
├── scripts
├── target
├── tools
└── top
```

## Instalando as dependências

Para esse post, vou utilizar o `qemu-system-arm`. Para isso, é preciso instala-lo:

```{.sh}
sudo apt-install qemu-system-arm
```

Na própria página do `Littekernel` tem uma sugestão para *toolchain*:

```{.sh}
sudo apt-get install gcc-arm-none-eabi
```

E essas são as dependências iniciais.

## Buildando

Os desenvolvedores do `LittleKernel` fornecem um script para rodar o *bootloader* no ==QEMU==. Basta entrar na pasta do repositório e digitar:

```{.sh}
scripts/do-qemuarm
```

Esse comando irá compilar o `LittleKernel` e carrega-lo no ==QEMU==. Como resultado o terminal (ou console) do *bootloader* será carregado e as seguintes mensagens devem parecer:

```{.sh linenums="1" hl_lines="1 42"}
welcome to lk/MP

boot args 0x0 0x0 0x0 0x0
INIT: cpu 0, calling hook 0x8011c5ed (version) at level 0x3ffff, flags 0x1
version:
	arch:     arm
	platform: qemu-virt-arm
	target:   qemu-virt-arm
	project:  qemu-virt-arm32-test
	buildid:  L9PCS_LOCAL
INIT: cpu 0, calling hook 0x8011e18d (vm_preheap) at level 0x3ffff, flags 0x1
initializing heap
calling constructors
INIT: cpu 0, calling hook 0x8011e1d5 (vm) at level 0x4ffff, flags 0x1
initializing mp
initializing threads
initializing timers
initializing ports
creating bootstrap completion thread
top of bootstrap2()
INIT: cpu 0, calling hook 0x801196e9 (minip) at level 0x70000, flags 0x1
INIT: cpu 0, calling hook 0x80119eb1 (pktbuf) at level 0x70000, flags 0x1
pktbuf: creating 256 pktbuf entries of size 1536 (total 393216)
INIT: cpu 0, calling hook 0x8011cf85 (virtio) at level 0x70000, flags 0x1
releasing 0 secondary cpus
initializing platform
PCIE: initializing pcie with ecam at 0x3f000000 found in FDT
PCI: pci ecam functions installed
PCI: last pci bus is 15
PCI dump:
  bus 0
   dev 0000:00:00.0 vid:pid 1b36:0008 base:sub:intr 6:0:0 
PCI dump post assign:
  bus 0
   dev 0000:00:00.0 vid:pid 1b36:0008 base:sub:intr 6:0:0 
initializing target
INIT: cpu 0, calling hook 0x8011ced5 (e1000) at level 0x90001, flags 0x1
initializing apps
starting app inetsrv
starting internet servers
starting app shell
entering main console loop
] 
```

Pode digitar o comando `help` e começar a desbravá-lo.

Para sair, é preciso usar as seguintes teclas: ==CTRL + A== e depois ==c==. O terminal sairá do `LittleKernel` e entrará no ==QEMU==:

```{.sh}
 QEMU 4.2.1 monitor - type 'help' for more information
(qemu) 
```

Por fim, bast digitar ==quit==.

### Decifrando o do-qemuarm

Esse script possui muitas linhas, mas as 3 últimas linhas resumem o processo:

```{.sh linenums="132"}
$DIR/make-parallel $MAKE_VARS $PROJECT &&
echo $SUDO $QEMU $ARGS $@ &&
$SUDO $QEMU $ARGS $@
```

Na ==linha 132==:

- `$MAKE_VARS` é vazio
- `$PROJECT` é `"qemu-virt-arm32-test"`
    - Esse nome é referente a um arquivo dentro do diretório `projects/` na pasta raiz.
- Esses são os parâmetros para compilar o `LittleKernel`.

Na ==linha 133==:

- Mostra o comando completo para emulação do `LittleKernel` pelo ==QEMU==:

```{.sh}
qemu-system-arm -cpu cortex-a15 -m 512 -smp 1 -machine virt,highmem=off -kernel build-qemu-virt-arm32-test/lk.elf -net none -nographic
```

- Baseado no comando gerado, pode-se observar que é criado a pasta `build-qemu-virt-arm32-test` e dentro dela o arquivo `lk.elf`. Esse arquivo é o binário gerado pela compilação do `LittleKernel`.
Na ==linha 134==:

- Executa a emulação.

## Conclusão

Com esses passos é possível começar a explorar o `LittleKernel` e tirar as primeiras impressões. Particularmente, achei a estrutura de pastas mais amigável que o do `U-Boot`. Isso não quer dizer que seja melhor ou pior, mas parece mais fácil de organizar.