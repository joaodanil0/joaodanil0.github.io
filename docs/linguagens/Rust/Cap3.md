# Conceitos

Dando continuidade ao conhecimento do `Rust`, vamos ao capítulo 3 do livro:

> The Rust Programming Language

Neste capítulo é visto os conceitos comuns de toda as linguagens, como: 

- [ ] Variáveis e mutabilidade
- [ ] Tipos de dado
- [ ] Funções
- [ ] Comentários
- [ ] Controle de fluxo 

## Variáveis e Mutabilidade

### Imutabilidade

Por padrão, as variáveis são ==imutáveis==. Isso acaba gerando mais segurança na hora de escrever o código, mas ainda existe a opção de criar variáveis ==mútaveis==.

Para exemplificar, criei o projeto `variables`, com o comando:

```
cargo new variables
```

e criei o seguinte programa: 

```rs linenums="1"
fn main() {
    let x = 5;
    println!("The value of x is: {x}");
    x = 6;
    println!("The value of x is: {x}");
}
```

ao tentar compilar o programa, o seguinte erro irá aparecer:

> 
error[E0384]: cannot assign twice to immutable variable `x` <br>
 --> src/main.rs:4:5<br>
  |<br>
2 |     let x = 5;<br>
  |         -<br>
  |         |<br>
  |         first assignment to `x`<br>
  |         help: consider making this binding mutable: `mut x`<br>
3 |     println!("The value of x is: {x}");<br>
4 |     x = 6;<br>
  |     ^^^^^ cannot assign twice to immutable variable<br>
<br>
For more information about this error, try `rustc --explain E0384`.<br>
error: could not compile `variables` due to previous error

O erro é `cannot assign twice to immutable variable`, isso quer dizer que uma variável teve seu valor alterado durante a execução do programa. Como por padrão `rust` possui variáveis ==imutáveis==, esse erro acabou ocorrendo (linha 6).

Apesar de `rust` encorajar o uso de variáveis ==imutáveis==, é possível criar variáveis ==mutáveis==, basta fazer a seguinte alteração no programa:

```rs linenums="1" hl_lines="2"
fn main() {
    let mut x = 5;
    println!("The value of x is: {x}");
    x = 6;
    println!("The value of x is: {x}");
}
```

Agora, o programa irá compilar e rodar corretamente.

### Constantes

Como variáveis ==imutáveis==, as ==constantes== também não podem ser alteradas. Existem algumas diferenças entre elas:

- Não é permitido usar `mut` em ==constantes==
- ==Constantes== são declaradas com `const`
- O tipo da ==constante== deve ser declarado.
- ==Constantes== podem ser declaradas em qualquer escopo

Um exemplo de declaração de ==constante==:

```rs linenums="1"
fn main() {
    const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
}
```

### Shadowing

É possível declarar variáveis com o mesmo nome, em `rust` isso quer dizer que **A primeira variável é *shadowing* pela segunda**. Segue um exemplo:

```rs linenums="1"
fn main() {

    let x = 5;

    let x = x + 1;

    {
        let x = x * 2;
        println!("The value of x in the inner scope is: {x}");
    }

    println!("The value of x is: {x}");
}
```

Na linha 5, a variável `x` é ==*shadowing*== pelo da linha 6. Dessa forma, o novo valor de `x` é 6. Na linha 8, ocorre o mesmo processo, resultando em `x` com valor 12. A diferença é que na linha 8, `x` está em outro escopo. Por isso, o valor de `x` na linha 12 não é alterado.

O resultado da execução do programa é:

>
The value of x in the inner scope is: 12 <br>
The value of x is: 6

A diferença entre ==*shadowing*== e variáveis com `mut`, é que um erro de compilação vai surgir se o valor da variável for modificada. Com ==*shadowing*== a variável continua ==imutável==, ou seja, o valor da variável não é simplesmente alterado, ela é totalmente recriada (como uma nova variável) em outra região de memória.

Esse exemplo ajuda a entender:

```rs linenums="1"
fn main() {
    let spaces = "   ";
    let spaces = spaces.len();
    println!("Spaces is {spaces}");
}
```

Inicialmente a variável `spaces` é uma `&str`, mas logo em seguida é um `usize`. Essa é uma vantagem do ==*shadowing*==.

Utilizando o `mut`, essa abordagem não é possível:

```rs linenums="1"
fn main() {
    let mut spaces = "   ";
    spaces = spaces.len();
    println!("Spaces is {spaces}");
}
```

Ao tentar compilar o programa, o seguinte erro aparece:

>
error[E0308]: mismatched types<br>
 --> src/main.rs:3:14<br>
  |<br>
2 |     let mut spaces = "   ";<br>
  |                      ----- expected due to this value<br>
3 |     spaces = spaces.len();<br>
  |              ^^^^^^^^^^^^ expected `&str`, found `usize`<br>
<br>
For more information about this error, try `rustc --explain E0308`.<br>
error: could not compile `variables` due to previous error

## Tipos de dado

`Rust` é uma ==linguagem estaticamente tipada==, isso quer dizer que os tipos de todas as variáveis devem ser conhecidas em tempo de compilação. Neste caso, o compilador irá inferir o tipo do dado. Em casos onde o tipo pode ser ambiguo, é nesserário adicionar o tipo manualmente, por exemplo:

```rs
let guess: u32 = "42".parse().expect("Not a number!");
```

Se o tipo `u32` não for declarado, a compilação retorna um erro.

### Escalar

O escalar representa um único valor. `Rust` possui 4 tipos de principais de escalares:

1. interger
- floating-point
- Booleans
- Character


#### Inteiros

Existem também alguns inteiros *built-in*:

|  Tamanho | com sinal | sem sinal |
|:--------:|:---------:|:---------:|
|  8-bits  |     i8    |     u8    |
|  16-bits |    i16    |    u16    |
|  32-bits |    i32 (padrão)   |    u32    |
|  64-bits |    i64    |    u64    |
| 128-bits |    i128   |    u128   |
|   arch   |   isize   |   usize   |

Além disso, é possível escrever inteiros na forma literal de outras formas:

| **Number literals** | **Example** |
|:-------------------:|:-----------:|
|       Decimal       |    98_222   |
|         Hex         |     0xFF    |
|        Octal        |     0o77    |
|        Binary       | 0b1111_0000 |
|    Byte (u8 only)   |     b'A'    |

#### Ponto-Flutuante

`Rust` possui 2 tipos primitivos para ponto-flutuante:

- f32 - single-precision
- f64 - double-precision (padrão)

```rust
fn main() {
    let x = 2.0; // f64

    let y: f32 = 3.0; // f32
}
```

> Pontos flutuantes em rust são representados de acordo com o IEEE-754.

#### Operação com números

- Adição
- Subtração
- Multiplicação
- Divisão
- Resto

```rust
fn main() {
    // Adição
    let sum = 5 + 10;

    // Subtração
    let difference = 95.5 - 4.3;

    // Multiplicação
    let product = 4 * 30;

    // Divisão
    let quotient = -1.0 / 2.0;
    let truncated = -1 / 2; // Results in 0

    // Resto
    let remainder = 43 % 5;
}
```

#### Boolean

`Rust` possui 2 valores possíveis:

- true
- false

#### Caracteres

Este tipo possui 4 bytes e suporta:

- ASCII
- Accented letters
- Chinese
- Japanese
- Korean characters
- emoji
- Unicode

Alguns exemplos:

```rust
fn main() {
    let c = 'z';
    let z: char = 'ℤ'; // with explicit type annotation
    let heart_eyed_cat = '😻';
}
```

> char literais utilizam aspas simples, ao contrário de strings literais que usam aspas duplas. 

Unicode: 

- `U+0000` até `U+D7FF` 
- `U+E000` até `U+10FFFF`

### Composto

`Rust` possui 2 tipos basicos de tipos compostos:

- tuples
- arrays

#### Tuplas

É capaz de juntar diversos tipos em um único tipo composto. Tuplas possuem tamanho fixo, ou seja, uma vez declarado, não pode ser alterado.

```rust
fn main() {
    let tup: (i32, f64, bool, char) = (500, 6.4, true, '😻');
    let (x, y, z, a) = tup; // destructuring

    let tup_0 = tup.0;
    let tup_1 = tup.1;
    let tup_2 = tup.2;
    let tup_3 = tup.3;

    let unit : ();
}
```

> Uma tupla vazia é chamada de `unit`.

#### Arrays

Diferente das tuplas, todos os elementos de um `arrays` são do mesmo tipo. Também possuem tamanho fixo.

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
    let b: [i32; 5] = [1, 2, 3, 4, 5]; // Declaring explicit type in initialization
    let c = [3; 5]; // Result: [3, 3, 3, 3, 3];

    let first_a = a[0];
}
```
> Arrays são uteis quando se quer utilizar a *stack* ao invés da *heap*


## Funções

`Rust`, por convensão, utiliza *snake case* como estilo para funções e nome de variáveis. As funções podem ser declaradas antes ou depois da `main`.

```rust
fn other_function(value: i32, unit_label: char) {
    println!("The measurement is: {value}{unit_label}");
}

fn main() {
    println!("Hello, world!");

    another_function();
    other_function(5, 'h');
}

fn another_function() {
    println!("Another function.");
}
```

Existe uma distinção entre: Declarações e Expressões:

- Statements: ==Não retornam== valores após sua execução.

```rust
fn main() {
    statement_function();
}

fn statement_function() {
  println!("Statement");
}
```

- Expressions: ==Retornam== algum valor após sua execução.

```rust
fn main() {
    let y = {
        let x = 3;
        x + 1
    };

    println!("The value of y is: {y}");
}
```

###  Retorno

Funções que retornam algum valor utilizam a seta (`->`) em sua assinatura para determinar o tipo do retorno:

```rust
fn main() {
    let x = plus_one(5);

    println!("The value of x is: {x}");
}

fn plus_one(x: i32) -> i32 {
    x + 1
}
```

> Observe que para retorno, não é utilizado o ponto e virgula no final. Caso ele seja colocado o compilador irá retornar um erro.

## Comentários

Comentários em `rust` utilizam apenas `//`.

```rust
// This is a
// Multiple Line commnet

fn main() {
    // Single line comment
    let lucky_number = 7; // I’m feeling lucky today
}
```

## Controle de fluxo

### If

Forma mais comum, utilizadas em outras linguagens:

```rs
fn main() {
    let number = 6;

    if number % 4 == 0 {
        println!("number is divisible by 4");
    } else if number % 3 == 0 {
        println!("number is divisible by 3");
    } else if number % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```

Uma forma mais enxuta de `if`, é chamada em `rust`de ==let statement==:

```rs
fn main() {
    let condition = true;
    let number = if condition { 5 } else { 6 };

    println!("The value of number is: {number}");
}
```

### Loops com Repetição

#### loop

```rs title="Infite Loop"
fn main() {
    loop {
        println!("again!");
    }
}
```

```rs title="Loop with return"
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    println!("The result is {result}");
}
```

Quando possui `loop`dentro de outro `loop` e queremos usar o comandos `break` ou `continue`, em `rust` é possível criar etiquetas para cada `loop`:

```rs title="Loop Label" hl_lines="3 13" linenums="1"
fn main() {
    let mut count = 0;
    'counting_up: loop {
        println!("count = {count}");
        let mut remaining = 10;

        loop {
            println!("remaining = {remaining}");
            if remaining == 9 {
                break;
            }
            if count == 2 {
                break 'counting_up;
            }
            remaining -= 1;
        }

        count += 1;
    }
    println!("End count = {count}");
}
```

#### while

```rs
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{number}!");

        number -= 1;
    }

    println!("LIFTOFF!!!");
}
```

#### for

```rs
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a {
        println!("the value is: {element}");
    }
}
```

```rs
fn main() {
    for number in (1..4).rev() {
        println!("{number}!");
    }
    println!("LIFTOFF!!!");
}
```

