# Criando um Servidor TFTP

<figure markdown>
  ![imagems](assets/cloud-computing-g7d7af6395_1280.png){ width="300" }
  <figcaption>
  Image by <a href="https://pixabay.com/users/openclipart-vectors-30363/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=2023902">OpenClipart-Vectors</a> from <a href="https://pixabay.com//?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=2023902">Pixabay</a>
  </figcaption>
</figure>


## Introdução

Com a ideia de facilitar o processo de testes do *bootloader*, Kernel Linux e do *Root FS* é possível uma rede de computadores para carregar o Kernel e o *Root FS*, ao invés de flashar no cartão SD. O primeiro passo é saber como configurar o servidor. Para isso, foi utilizado o protocolo [TFTP](https://pt.wikipedia.org/wiki/Trivial_File_Transfer_Protocol). Por fim, configurar o *bootloader* para buscar a imagem do Kernel Linux e o DTB no servidor, que foi mostrado no post [Habilitando o env.txt com tfpt](../boards/Beaglebone/Bootloader/env.md)

## Configurando o Servidor

É necessário instalar algumas dependências e configurar alguns arquivos.

### Dependências

Podemos instalar os pacotes `xinetd`, `tftp` e `tftpd` com o comando:

```
sudo apt-get install xinetd tftp tftpd
```

### Arquivo de Configuração

O serviço do TFTP busca o arquivo `tftp`, para isso devemos cria-lo:

```
sudo touch /etc/xinetd.d/tftp 
```

Dentro do arquivo precisamos adicionar algumas informações:

```
service tftp
{
protocol = udp
port = 69
socket_type = dgram
wait = yes
user = nobody
server = /usr/sbin/in.tftpd
server_args = /home/joao/Documents/TFTP -s
disable = no
}
```

> O parâmetro `server_args` é o local onde ficará os arquivos que serão compartilhados.

Se for o caso, agora crie a pasta configurada no parâmetro `server_args`:

```
mkdir -p /home/joao/Documents/TFTP
```

### Permissões

Como um outro dispositivo irá acessar essa pasta (no nosso exemplo, a beaglebone), a pasta criada precisa ter permissões para isso. Primeiro, vamos configurar os usuários:

```
sudo chown nobody:nogroup /home/joao/Documents/TFTP
```

Por fim, dando acesso total a pasta:

```
sudo chmod 777 /home/joao/Documents/TFTP
```

!!! danger "Cuidado"
    Cuidado ao utilizar o comando acima. Usar o `777` é altamente desencorajado. Lembre-se que esse é um setup de testes.

### Reiniciando o Serviço

Por fim, basta reiniciar o serviço para que ele carregue as novas configurações:

```
sudo /etc/init.d/xinetd restart
```

## Conclusão

O TFTP pode ser utilizado para compartilhar arquivos de forma geral, mas o propósito desse post é fazer com que o u-boot acesse o servidor criado e possa baixar a imagem do Kernel Linux.






