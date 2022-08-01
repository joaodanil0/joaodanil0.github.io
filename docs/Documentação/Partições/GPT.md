# Tabela de Partição GUID


> baseado nos [link1](https://www.codetd.com/pt/article/13094374), [link2](https://metebalci.com/blog/a-quick-tour-of-guid-partition-table-gpt/) e [link3](https://en.wikipedia.org/wiki/GUID_Partition_Table)

---

## Considerações iniciais

Vamos utilizar um cartão micro SD de 32GB. Para identificarmos o cartão no sistema, utilizamos o comando `lsblk`, o resultado será similar a essa

```{.sh}
╰─ lsblk  
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0 894,3G  0 disk 
└─sda1        8:1    0 894,3G  0 part /run/timeshift/backup
sdc           8:32   1  29,1G  0 disk 
└─sdc1        8:33   1  29,1G  0 part 
nvme0n1     259:0    0 465,8G  0 disk 
├─nvme0n1p1 259:1    0   476M  0 part /boot/efi
├─nvme0n1p2 259:2    0  30,5G  0 part [SWAP]
└─nvme0n1p3 259:3    0 434,8G  0 part /
```

Para ter certeza qual é o cartão SD, execute o `lsblk` com o cartão não conectado e depois conecte o cartão e execute o comando novamente (simples :smile:). No caso acima, o cartão SD é o `/dev/sdc`.

Para facilitar a visualização das alterações que iremos fazer, vamos zerar os primeiros 10240 bytes do cartão. Para isso, utilizaremos o comando [`dd`](https://man7.org/linux/man-pages/man1/dd.1.html)

```{.sh}
sudo dd if=/dev/zero of=/dev/sdc bs=1024 count=10
```

Podemos visualizar conteúdo do cartão, com o comando [`hexdump`](https://www.suse.com/c/making-sense-hexdump/)

```{.sh}
sudo hexdump -C /dev/sdc -n 10240
```

O resultado deve ser similar a esse

```{.sh}
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00002800
```

> o número 00002800 é em hexadecimal e seu equivalente em decimal é 10240 (ou 10 kB).

## Formatando o cartão com GPT

Com os primeiros bytes do cartão SD zerados, vamos adicionar a partição GPT ao cartão com o comando [`fdisk`](https://linuxize.com/post/fdisk-command-in-linux/)

```
sudo fdisk /dev/sdc
```

Um menu irá aparecer:

- Digite `g`
- Depois digite `n`
    - Partition number (1-128, default 1): `Enter`
    - First sector (2048-61067230, default 2048): `Enter`
    - Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-61067230, default 61067230): `Enter`
- Por fim, digite `w`

Agora o cartão está com o particionamento GPT. Para checar, vamos visualizar novamente o conteúdo do cartão SD

```{.sh}
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000001c0  02 00 ee ff ff ff 01 00  00 00 ff cf a3 03 00 00  |................|
000001d0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
00000200  45 46 49 20 50 41 52 54  00 00 01 00 5c 00 00 00  |EFI PART....\...|
00000210  fc d8 3a 53 00 00 00 00  01 00 00 00 00 00 00 00  |..:S............|
00000220  ff cf a3 03 00 00 00 00  00 08 00 00 00 00 00 00  |................|
00000230  de cf a3 03 00 00 00 00  90 30 7b d4 f3 72 46 59  |.........0{..rFY|
00000240  89 2c 7f 77 9e b0 00 e6  02 00 00 00 00 00 00 00  |.,.w............|
00000250  80 00 00 00 80 00 00 00  f1 ae 2f f5 00 00 00 00  |........../.....|
00000260  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000400  af 3d c6 0f 83 84 72 47  8e 79 3d 69 d8 47 7d e4  |.=....rG.y=i.G}.|
00000410  9c 58 84 76 3e 65 45 92  a4 ec 96 ed 84 df 22 99  |.X.v>eE.......".|
00000420  00 08 00 00 00 00 00 00  de cf a3 03 00 00 00 00  |................|
00000430  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00002800
```

Esses dados mostram que o GPT está sendo usando no cartão SD.

## O Backup de segurança

Como uma forma de segurança contra perda de dados, o GPT faz um backup da partição no final disco. Para checar basta utilizar o comando:

```
sudo tail -c 10M /dev/sdc | hexdump -C
```


Visualizando os últimos 10 MB do cartão SD, encontramos o seguinte conteúdo


```{.sh}
747900000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
7479fbe00  af 3d c6 0f 83 84 72 47  8e 79 3d 69 d8 47 7d e4  |.=....rG.y=i.G}.|
7479fbe10  9c 58 84 76 3e 65 45 92  a4 ec 96 ed 84 df 22 99  |.X.v>eE.......".|
7479fbe20  00 08 00 00 00 00 00 00  de cf a3 03 00 00 00 00  |................|
7479fbe30  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
7479ffe00  45 46 49 20 50 41 52 54  00 00 01 00 5c 00 00 00  |EFI PART....\...|
7479ffe10  66 9b 01 67 00 00 00 00  ff cf a3 03 00 00 00 00  |f..g............|
7479ffe20  01 00 00 00 00 00 00 00  00 08 00 00 00 00 00 00  |................|
7479ffe30  de cf a3 03 00 00 00 00  90 30 7b d4 f3 72 46 59  |.........0{..rFY|
7479ffe40  89 2c 7f 77 9e b0 00 e6  df cf a3 03 00 00 00 00  |.,.w............|
7479ffe50  80 00 00 00 80 00 00 00  f1 ae 2f f5 00 00 00 00  |........../.....|
7479ffe60  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
747a00000
```
Podemos perceber um espelho dos dados do inicio do disco.

## O *Logical Block Addressing*

O *Logical Block Addressing* ou LBA, são blocos de informação utilizado pelo GPT. Esses blocos possuem 512 bytes.

### LBA0

O `LBA0` existe para compatibilidade com o *Master Boot Record* (MBR), checando os primeiros 512 bytes do cartão SD

```
sudo hexdump -C /dev/sdc -s 0 -n 512
```

Temos esse resultado:

```
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000001c0  02 00 ee ff ff ff 01 00  00 00 ff cf a3 03 00 00  |................|
000001d0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
00000200
```

Nada relevante pode ser visto aqui (a não ser que você esteja usando MBR :-1:)

### LBA1

Esse LBA possui o cabeçalho da tabela de partição. Visualizando os próximos 512 bytes do cartão SD

```
sudo hexdump -C /dev/sdc -s 512 -n 512
```

Temos os seguintes dados:

```
00000200  45 46 49 20 50 41 52 54  00 00 01 00 5c 00 00 00  |EFI PART....\...|
00000210  d2 61 e7 36 00 00 00 00  01 00 00 00 00 00 00 00  |.a.6............|
00000220  ff cf a3 03 00 00 00 00  00 08 00 00 00 00 00 00  |................|
00000230  de cf a3 03 00 00 00 00  46 8a 0b 22 c3 8c 45 75  |........F.."..Eu|
00000240  8c ef f5 bd 9e 32 4c 5b  02 00 00 00 00 00 00 00  |.....2L[........|
00000250  80 00 00 00 80 00 00 00  af 06 c5 b4 00 00 00 00  |................|
00000260  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000400
```

Agora vamos entender esse vetor de bytes:

> Esses dados são *Little Endian*. Para interpretarmos o valor em decimal, devemos converter para *Big Endian*

- A assinatura são os primeiros 8 bytes

```
00000200  45 46 49 20 50 41 52 54                           |EFI PART|
00000208
```

- A revisão são os 4 próximos bytes

```
00000208  00 00 01 00                                       |....|
0000020c
```

- O tamanho do cabeçalho são os 4 próximos bytes

```
0000020c  5c 00 00 00                                       |\...|
00000210
```

- O CRC32 da partição são os 4 próximos bytes

```
00000210  d2 61 e7 36                                       |.a.6|
00000214
```

- Os próximos 4 bytes são reservados e devem ser 0's

```
00000214  00 00 00 00                                       |....|
00000218
```

- Endereço do LBA atual são os 8 próximos bytes

```
00000218  01 00 00 00 00 00 00 00                           |........|
00000220
```

- Localização do LBA espelho (backup) são os 8 próximos bytes

```
00000220  ff cf a3 03 00 00 00 00                           |........|
00000228
``` 

> Big Endian: 00 00 00 00 03 A3 CF FF | Decimal 61067263. Isso indica que o LBA espelho está no setor 61067263. Para acessa-lo podemos utilizar o comando: sudo dd if=/dev/sdc bs=512 count=1 skip=61067263

- Primeiro setor disponível para partição são os próximos 8 bytes

```
00000228  00 08 00 00 00 00 00 00                           |........|
00000230
```

> 00 00 08 00 00 00 00 00 -> 2048

- Último setor disponível para partição são os próximos 8 bytes

```
00000230  de cf a3 03 00 00 00 00                           |........|
00000238
```
> Big endian: 0000000003A3CFDE | Decimal: 61067230. O último setor disponível é o 61067230.

- O GUID são os próximos 16 bytes

```
00000238  46 8a 0b 22 c3 8c 45 75  8c ef f5 bd 9e 32 4c 5b  |F.."..Eu.....2L[|
00000248
```

> O GUID possui segmentações: 468a0b22-c38c-4575-8cef-f5bd9e324c5b. Os primeiros 8 bytes são convertidos para big endian e o restante não é convertido, resultando em: 220B8A46-8CC3-7545-8CEF-F5BD9E324C5B

- O setor que inicia as tabelas de partição são os próximos 8 bytes

```
00000248  02 00 00 00 00 00 00 00                           |........|
00000250
```

- Quantidades possíveis de partições são os próximos 4 bytes

```
00000250  80 00 00 00                                       |....|
00000254
```

- Tamanho de um entrada da tabela de partição são os próximos 4 bytes

```
00000254  80 00 00 00                                       |....|
00000258
```

- CRC32 de todas as entradas de partição do disco são os próximos 4 bytes

```
00000258  af 06 c5 b4                                       |....|
0000025c
``` 

- Restante do espaço são 420 bytes

```
0000025c  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000400
```

### LB2 - LBN

```
00000400  af 3d c6 0f 83 84 72 47  8e 79 3d 69 d8 47 7d e4  |.=....rG.y=i.G}.|
00000410  a0 d5 b4 b0 cf d0 4d fe  b7 09 a8 3f 44 67 bd b7  |......M....?Dg..|
00000420  00 08 00 00 00 00 00 00  de cf a3 03 00 00 00 00  |................|
00000430  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000480
```

- Tipo de partição GUID são os primeiros 16 bytes

```
00000400  af 3d c6 0f 83 84 72 47  8e 79 3d 69 d8 47 7d e4  |.=....rG.y=i.G}.|
00000410
```

- GUID único são os próximos 16 bytes

```
00000410  a0 d5 b4 b0 cf d0 4d fe  b7 09 a8 3f 44 67 bd b7  |......M....?Dg..|
00000420
```

- Primeiro setor disponível para partição são os próximos 8 bytes

```
00000420  00 08 00 00 00 00 00 00                           |........|
00000428
```

- Último setor disponível para partição são os próximos 8 bytes

```
00000428  de cf a3 03 00 00 00 00                           |........|
00000430
```

- Os atributos são os próximos 8 bytes

```
00000430  00 00 00 00 00 00 00 00                           |........|
00000438
```

- O nome da partição são os próximos 72 bytes

```
00000438  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000480
```