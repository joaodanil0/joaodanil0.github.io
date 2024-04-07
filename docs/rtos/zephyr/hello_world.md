# Zephyr Hello World

<figure markdown>
  ![imagems](images/Hello.png){ width="600" }
  <figcaption>
  Image by <a href="https://pixabay.com/users/asoyid-17566724/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=5458998">Asoy ID</a> from <a href="https://pixabay.com//?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=5458998">Pixabay</a>
  </figcaption>
</figure>

## Introdução

Ultimamente venho usando bastante o *Zephyr* junto com a placa `nRF53nrf52840 dk` da *nordic*. Frequentemente, quando quero criar um projeto do zero (para testar alguma coisa específica), acabo me deparando com a dúvida de quais são os arquivos realmente necessários para criar o projeto mais limpo possível. Por isso, resolvi explicar isso nesse post.

## Hello World

Nada melhor do que o bom e velho *Hello World* para mostras esses detalhes.

### CMake

Esse é o arquivo que vai linkar os módulos do *zephyr* com o seu código. 

```py title="CMakeLists.txt"
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.20.0) #Versão mínima suportada do cmake

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE}) #Busca os módulos do Zephyr
project(hello_world) #Define o nome do projeto

target_sources(app PRIVATE src/main.c) #Define quais são os arquivos do seu código
```

### Código

Aqui você deve colocar a implementação da sua aplicação.

> Observe que o nome do arquivo deve ser o mesmo colocaco no CMakeList.txt.

```c title="main.c"
/*
 * Copyright (c) 2012-2014 Wind River Systems, Inc.
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <stdio.h>

int main(void)
{
	printf("Hello World! %s\n", CONFIG_BOARD_TARGET);

	return 0;
}
```

### Configurações

O *Zephyr* utiliza o arquivo `prj.conf` para habilitar configurações do projeto. Para esse exemplo, não há necessidade de adicionar nada no arquivo.

```conf title="prj.conf"

```

### Árvore dos Arquivos

Para facilitar a organização dos arquivos segue a árvore hierarquica:

```.sh                                                       
hello_world
├── CMakeLists.txt
├── prj.conf
└── src
    └── main.c

1 directory, 3 files
```

## Conclusão

Uma explicação rápida dos arquivos necessários foi mostrada. Existem arquivos que são recomendandos, como o `README.rst` e o `sample.yaml`, mas eles não são necessários para buildar o projeto. Dessa forma, somenete os arquivos: `CMakeLists.txt`, `main.c` e `prj.conf` são necessários para compilar o projeto com sucesso.


