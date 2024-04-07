# Adivinhação

## Introdução

Dando continuidade ao conhecimento do `Rust`, vamos ao capítulo 2 do livro:

> The Rust Programming Language

Ele propõem introduzir alguns conceitos comuns da linguagem, criando o ==jogo da adivinhação==

## Configurando o Projeto

Criei um novo projeto utilizando o `Cargo`:

```{.sh}
cargo new adivinhacao
```

## Começando o Jogo

Vou dividir em alguns passos para facilitar o passo a passo da criação do jogo.

> Para compilar o programa utilizei o `cargo build` e para executar `cargo run`.

### Input
O primeiro passo é saber como fazer um *input* do usuário. Para isso, segue o código abaixo:

```{.rs linenums="1" title="src/main.rs"}
// Baseado no exemplo do próprio livo, com algumas modificações

use std::io; //Lib para tratar de I/O

fn main() {

    // Observe o termo "mut", pois essa variável 
    // será alterada com a entrada do usuário
    let mut guess = String::new(); 

    io::stdin()
      .read_line(&mut guess) // Lê a entrada e atribui a variável "guess"
      .expect("Failed to read line"); // Caso algo saia errado, mostra a mensagem
    
    println!("You guessed: {guess}"); //Mostra a entrada do usuário
}

```

### Números Aleatórios

O `Rust` não possui a geração de números aleatórios em sua biblioteca padrão. Com isso, usei o ==rand== disponível no [crate](https://crates.io/crates/rand). Para isso, é necessário adicionar a biblioteca no arquivo ==Cargo.toml==:

```{.toml hl_lines="2"}
[dependencies]
rand = "0.8.3"
```

Agora utilize o comando `cargo build` para instalar a dependência.

Voltando ao código, vamos adicionar biblioteca e a função que gera os números:

```{.rs  hl_lines="4 8-10" linenums="1" title="src/main.rs"}
// Baseado no exemplo do próprio livro, com algumas modificações

use std::io; //Lib para tratar de I/O
use rand::Rng; //Lib para gerar números randômicos

fn main() {

    // Gera um número inteiro de 0 a 100
    let secret_number = rand::thread_rng().
                            gen_range(1..=100);

    println!("The secret number is: {secret_number}");

    // Observe o termo "mut", pois essa variável 
    // será alterada com a entrada do usuário
    let mut guess = String::new(); 

    io::stdin()
      .read_line(&mut guess) // Lê a entrada e atribui a variável "guess"
      .expect("Failed to read line"); // Caso algo saia errado, mostra a mensagem
    
    println!("You guessed: {guess}"); //Mostra a entrada do usuário
}
```

### Comparando

Uma vez que a entrada do usuário e o número randômico estão disponíveis, só resta fazer a comparação entre os dois e checar se é menor, maior ou igual.

```{.rs linenums="1" title="src/main.rs" hl_lines="5 23-25 29-34"}
// Baseado no exemplo do próprio livro, com algumas modificações

use std::io; //Lib para tratar de I/O
use rand::Rng; //Lib para gerar números randômicos
use std::cmp::Ordering; //Lib para fazer comparação

fn main() {

    // Gera um número inteiro de 0 a 100
    let secret_number = rand::thread_rng().
                            gen_range(1..=100);

    println!("The secret number is: {secret_number}");

    // Observe o termo "mut", pois essa variável 
    // será alterada com a entrada do usuário
    let mut guess = String::new(); 

    io::stdin()
      .read_line(&mut guess) // Lê a entrada e atribui a variável "guess"
      .expect("Failed to read line"); // Caso algo saia errado, mostra a mensagem

    // Converte a entrada para inteiro
    let guess: u32 = guess.trim().parse().
                        expect("\n\n>>>> Please type a integer number! <<<<\n\n");
    
    println!("You guessed: {guess}"); //Mostra a entrada do usuário

    // Compara e informa se o número é menor, maior ou igual
    match guess.cmp(&secret_number) {
        Ordering::Less => println!("Too small!"),
        Ordering::Greater => println!("Too big!"),
        Ordering::Equal => println!("You win!"),
    }
}

```

### Tentando Novamente

Até agora o jogo só permite apenas uma tentativa (*try hard* :skull:). Agora vou adicionar o *loop* para fazer várias tentativas.

```{.rs linenums="1" title="src/main.rs" hl_lines="15 38"}
// Baseado no exemplo do próprio livro, com algumas modificações

use std::io; //Lib para tratar de I/O
use rand::Rng; //Lib para gerar números randômicos
use std::cmp::Ordering; //Lib para fazer comparação

fn main() {

    // Gera um número inteiro de 0 a 100
    let secret_number = rand::thread_rng().
                            gen_range(1..=100);

    println!("The secret number is: {secret_number}");

    loop {
        
        // Observe o termo "mut", pois essa variável 
        // será alterada com a entrada do usuário
        let mut guess = String::new(); 


        io::stdin()
        .read_line(&mut guess) // Lê a entrada e atribui a variável "guess"
        .expect("Failed to read line"); // Caso algo saia errado, mostra a mensagem

        // Converte a entrada para inteiro
        let guess: u32 = guess.trim().parse().
                            expect("\n\n>>>> Please type a integer number! <<<<\n\n");
        
        println!("You guessed: {guess}"); //Mostra a entrada do usuário

        // Compara e informa se o número é menor, maior ou igual
        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => println!("You win!"),
        }
    }
}
```

### Ganhando

Mesmo digitando o número correto, o jogo ainda permite fazer uma nova tentativa. Vou adicionar um *break* quando o número digitado for o correto.

```{.rs linenums="1" title="src/main.rs" hl_lines="36-39"}
// Baseado no exemplo do próprio livro, com algumas modificações

use std::io; //Lib para tratar de I/O
use rand::Rng; //Lib para gerar números randômicos
use std::cmp::Ordering; //Lib para fazer comparação

fn main() {

    // Gera um número inteiro de 0 a 100
    let secret_number = rand::thread_rng().
                            gen_range(1..=100);

    println!("The secret number is: {secret_number}");

    loop {

        // Observe o termo "mut", pois essa variável 
        // será alterada com a entrada do usuário
        let mut guess = String::new(); 


        io::stdin()
        .read_line(&mut guess) // Lê a entrada e atribui a variável "guess"
        .expect("Failed to read line"); // Caso algo saia errado, mostra a mensagem

        // Converte a entrada para inteiro
        let guess: u32 = guess.trim().parse().
                            expect("\n\n>>>> Please type a integer number! <<<<\n\n");
        
        println!("You guessed: {guess}"); //Mostra a entrada do usuário

        // Compara e informa se o número é menor, maior ou igual
        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}

```

### Tratando Erros

Caso a entrada do usuário não seja exatamente um inteiro, ocorre um *crash* no jogo. Vou adicionar um tratamento para lidar com esse problema.

```{.rs linenums="1" title="src/main.rs" hl_lines="27-33"}
// Baseado no exemplo do próprio livro, com algumas modificações

use std::io; //Lib para tratar de I/O
use rand::Rng; //Lib para gerar números randômicos
use std::cmp::Ordering; //Lib para fazer comparação

fn main() {

    // Gera um número inteiro de 0 a 100
    let secret_number = rand::thread_rng().
                            gen_range(1..=100);

    println!("The secret number is: {secret_number}");

    loop {

        // Observe o termo "mut", pois essa variável 
        // será alterada com a entrada do usuário
        let mut guess = String::new(); 


        io::stdin()
        .read_line(&mut guess) // Lê a entrada e atribui a variável "guess"
        .expect("Failed to read line"); // Caso algo saia errado, mostra a mensagem

        // Converte a entrada para inteiro
        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => {
                println!("Entrada Inválido");
                continue;
            }
        };
        
        println!("You guessed: {guess}"); //Mostra a entrada do usuário

        // Compara e informa se o número é menor, maior ou igual
        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}

```

## Conclusão

Criar o jogo introduziu o conceito de `match`, `loop`, `mut` e `Crate`. Que são básicos na linguagem.