# Criando um Servidor NFS

<figure markdown>
  ![imagems](assets/network-cable-g29244a4f1_640.jpg){ width="600" }
  <figcaption>
    Image by <a href="https://pixabay.com/users/blickpixel-52945/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=499792">Michael Schwarzenberger</a> from <a href="https://pixabay.com//?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=499792">Pixabay</a>
  </figcaption>
</figure>

## Introdução

Uma vez que o Kernel Linux é carregado, ele busca pelo *Root Filesystem* (RFS). Ele geralmente é flashado no mmc ou cartão SD, mas existem outras formas de disponibiliza-lo. Uma delas é por meio do *Network Filesytem* ([NFS](https://pt.wikipedia.org/wiki/Network_File_System)). Dessa forma, o RFS fica disponível via rede. Com isso, não é necessário re-flashar toda vez que alguma alteração for feita no RFS.

## Configurando o Servidor

São necessários os seguintes passos.

## Dependências

É preciso instalar o pacote `nfs-kernel-server`, ele é responsável por levantar no servidor NFS:

```
sudo apt install nfs-kernel-server
```

## Arquivos de Configuração

Devemos informar qual o caminho em que os arquivos serão compartilhados. Para isso, devemos informar no arquivo:

```
/etc/exports
```

E na última linha, devemos adicionar:

```
/home/joao/Documents/NFS 192.168.0.104(rw,sync,no_subtree_check)
```

> Perceba que é passado o caminho que será compartilhado, o IP que poderá acessar esse caminho e algumas outras opções.


## Permissões

Se for a caso, crie o caminho que será compartilhado, por exemplo:

```
mkdir -p /home/joao/Documents/NFS
```

Como um outro dispositivo irá acessar essa pasta (no nosso exemplo, a beaglebone), a pasta criada precisa ter permissões para isso. Primeiro, vamos configurar os usuários:

```
sudo chown nobody:nogroup /home/joao/Documents/NFS
```

Por fim, dando acesso total a pasta:

```
sudo chmod 777 /home/joao/Documents/NFS
```

!!! danger "Cuidado"
    Cuidado ao utilizar o comando acima. Usar o `777` é altamente desencorajado. Lembre-se que esse é um setup de testes.

### Reiniciando o Serviço

Por fim, basta reiniciar o serviço para que ele carregue as novas configurações:

```
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

## Conclusão

Seguindo esses passos é possível deixar um RFS acessível via rede, o que facilita os testes que vão ocorrendo. Um ponto importante é configurar os argumentos do Kernel Linux, nesse caso:

```
boot_args=root=/dev/nfs \
          ip=192.168.0.104 \
          nfsroot=192.168.0.203:/home/joao/Documents/NFS,nfsvers=4 rw rootwait
```

Dessa forma, o kernel saberá qual IP utilizar e em qual IP buscar o RFS.