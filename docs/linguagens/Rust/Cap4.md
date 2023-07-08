# Ownership

Dando continuidade ao conhecimento do `Rust`, vamos ao capítulo 4 do livro:

> The Rust Programming Language

Neste capítulo é visto um dos conceitos que faz o `rust` ser ==memory safety==.


## O que é o ownership?

*Ownership* é um conjunto de regras que governam a forma como `rust` gerencia a memória. Algumas linguagens possuem *garbage collector*, que varrem a memória periodicamente buscando alocações que não estão sendo mais utilizadas. Outras linguagens, o desenvolvedor deve alocar e desalocar a memória utilizada explicitamente.

`Rust` utiliza outra abordagem, a memória é gerenciada por meio de um conjunto de regras checadas pelo compilador. Se alguma dessas regras é violada, o compilador retorna um erro.

## A Stack e a Heap

Em linguagens de sistema, como `rust`, se um valor está na *stack* ou na *heap* afeta o comportamento da linguagem. Partes do `ownership` tem relação com a *stack* e a *heap*.

A *stack* e a *heap* fazem parte da memória disponível no código em tempo de execução, mas elas são estruturadas de formas diferentes. A *stack* armazena os valores em ordem e remove os valores em ordem reversa (LIFO). Adicionar um valor é chamado de *pushing onto the stack* e remover um valor é chamado de *popping off the stack*. Todos os valores armazenados na *stack* possuem um valor fixo, conhecido. Valors com tamanho desconhecido em tempo de compilação ou que mudam de tamanho, devem ser armazenados na *heap*

Quando um certo valor é armazenado na *heap*, é requisitado uma certa quantidade de memória (alocação). Esse processo é chamado de *allocating on the heap*. Uma vez que o espaço de memória é alocado, ele possui um valor fixo e conhecido. O processo de alocação retorna um ponteiro com o endereço da região de memória alocada. Esse ponteiro pode ser armazenado na *stack*, se quisermos saber o conteúdo desse endereço de memória, basta seguir o ponteiro.

> Adicionar valors na *stack* não é considerado alocação.

Adicionar um valor na *stack* é mais rápido que alocar um espaço na *heap*, porque não é necessário procurar um espaço disponível.

Procurar um valor na *heap* é mais lento do que acessar um valor na *stack*, porque não é preciso seguir o ponteiro para ter acesso ao valor.

É necessário ficar atento aos valores alocados na *heap* para evitar duplicações e consequentemente o desperdício. Além de limpar os espaços de memória que não estão sendo mais utilizados. Esses problemas são checados pelo `ownership`.

> Esse vídeo mostra alguns exemplos de uso sobre *heap* [The Origins of Process Memory](https://youtu.be/c7xf5dvUb_Q)

## Regras do Ownership

São eles:

- Cada valor em `rust` possui um dono (*owner*)
- Só pode existir um dono (*owner*) de cada vez.
- Quando o dono (*owner*) sai do escopo, o valor é descartado.

## Escopo de variável

Um escopo é um espaço dentro de um programa:

```rs
fn main(){

    // s is not valid here, it’s not yet declared
    {                      
        let s = "hello";   // s is valid from this point forward

        println!("{s}");
    }
    // s is not valid here, it’s out of scope
}
```

## O tipo String

O tipo `string` pode exemplificar melhor o uso do `ownership`. Segue um exemplo simples de uso do tipo `string`: 

```rs
fn main(){

    let mut s = String::from("hello");

    s.push_str(", world!"); // push_str() appends a literal to a String

    println!("{}", s); // This will print `hello, world!`
}
```

## Memória e alocação

No caso de `string` literal:

```rs
let s = "hello";
```

O conteúdo é conhecido em tempo de compilação, então o texto é adicionado diretamente ao final do executável (devido estar na *stack*). Devido isso, esse tipo de uso de `string` é mais rápido e eficiente.

Com o tipo `string`, para suportar uma variável mutável e de tamanho variável, é necessário alocar uma quantidade de memória da *heap*, desconhecida em tempo de compilação. Isso significa:

- A memória é requisitada em tempo de execução.
- É necessário devolver a memória alocada, quando a variável não for mais necessária. 

A requisição de memória é feita utilizando `String::from`.

Devolver a memória que não está mais sendo utilizada, é um desafio maior. Linguagens que utilizam os *garbage collector*, ficam buscando por espaços de memória que não são mais utilizados pelo programa, para desalocar o recurso. Para linguagens que não utilizam o *garbage collector*, é de responsabilidade do desenvolvedor identificar quando um espaço alocado não é mais necessário. Historicamente, esse é um problema em computação.

Em `rust` a memória é automaticamente retornada quando ela sai do escopo:

```rs
fn main(){

    // s is not valid here, it’s not yet declared
    {                      
        let s = String::from("hello");   // s is valid from this point forward

        println!("{s}");
    }
    // s is not valid here, it’s out of scope
}
```

Quando uma variável sai do escopo, `rust` chama uma função especial chamada `drop`. `Rust` chama o `drop` automaticamente ao final das chaves (`}`) em cada escopo.

> A função `drop` é similar ao [*Resource Acquisition Is Initialization*](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization) (RAII) utilizado  em C++.


## Reatribuindo variáveis

As variáveis armazenadas na *stack* funcionam da mesma forma que outras linguagens de programação. `Rust` se diferencia nas variáveis armazenadas na *heap*, por exemplo:

```rs
    let s1 = String::from("hello"); 
    let s2 = s1; // s1 no longer valid, after this.

    println!("{}, world!", s2);
```

Quando `s2 = s1`, automaticamente o `rust` considera `s1` não mais válido. Uma vez que `s2` aponta para a mesma região de memória de `s1`, não faz muito sentido termos duas variáveis apontando para o mesmo endereço. Dessa forma, `rust` acaba invalidando o a variável mais antiga, no caso, `s1`. Essa abordagem é conhecida como *move*. Então pode-se dizer que `s1` foi movido para `s2`.

> A vantagem dessa abordagem é que ela impede de termos o problema de *double free error* 

Caso seja necessário fazer um cópia de uma variável que possui valores na *heap*, `rust` oferece o comando `clone`:

```rs
    let s1 = String::from("hello");
    let s2 = s1.clone();

    println!("s1 = {}, s2 = {}", s1, s2);
```

Agora, ambas as variáveis `s1`e `s2` estão disponíveis, mas em regiões de memória distintas.

## Ownership e Funções

Trabalhar com funções é similar as variáveis:

```rs linenums="1" hl_lines="4"
fn main() {
    let s = String::from("hello");  // s comes into scope

    takes_ownership(s);             // s's value moves into the function...
                                    // ... and so is no longer valid here

    let x = 5;                      // x comes into scope

    makes_copy(x);                  // x would move into the function,
                                    // but i32 is Copy, so it's okay to still
                                    // use x afterward

} // Here, x goes out of scope, then s. But because s's value was moved, nothing
  // special happens.

fn takes_ownership(some_string: String) { // some_string comes into scope
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
  // memory is freed.

fn makes_copy(some_integer: i32) { // some_integer comes into scope
    println!("{}", some_integer);
} // Here, some_integer goes out of scope. Nothing special happens.
```

Novamente, a grande diferença está na linha 4. Uma vez que a variável `s` é passada para a função, ela não fica mais disponível.

## Retornando Valores em Funções

Retornar valores em `rust` é chamado ==*transfer ownership*==

```rs linenums="1" hl_lines="7 25"
fn main() {
    let s1 = gives_ownership();         // gives_ownership moves its return
                                        // value into s1

    let s2 = String::from("hello");     // s2 comes into scope

    let s3 = takes_and_gives_back(s2);  // s2 is moved into
                                        // takes_and_gives_back, which also
                                        // moves its return value into s3
} // Here, s3 goes out of scope and is dropped. s2 was moved, so nothing
  // happens. s1 goes out of scope and is dropped.

fn gives_ownership() -> String {             // gives_ownership will move its
                                             // return value into the function
                                             // that calls it

    let some_string = String::from("yours"); // some_string comes into scope

    some_string                              // some_string is returned and
                                             // moves out to the calling
                                             // function
}

// This function takes a String and returns one
fn takes_and_gives_back(a_string: String) -> String { // a_string comes into
                                                      // scope

    a_string  // a_string is returned and moves out to the calling function
}
```

Nesse caso, a variável `s2` é passada para a função. Por sua vez, a função retorna a variável recebida. 

## Referências e Empréstimos

Uma referência é como um ponteiro (de `C`) em que o valor contido no ponteiro é outro dono (*owner*). Diferente de ponteiros, uma referência sempre aponta para um valor válido.

```rs
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

Essa passagem de referência é conhecida como empréstimo (`borrowing`), em `rust`. Caso alguma coisa seja alterada na variável emprestada, o compilador irá retornar erro.

> Podemos referenciar uma variável com `&` e desreferenciar com `*`.

Para que seja possível fazer alterações em referências, é necessário declarar a referência com `mut`:

```rs
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

Referências mutáveis possuem um grande restrição. Só podemos ter um referência mutável por vez, de uma mesma variável:

```rs
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s;

    println!("{}, {}", r1, r2);
}
```

Neste caso, o compilador irá retornar um erro, devido `r1` e `r2` estão pegando a referência da mesma variável `s`.

Essa abordagem previne alguns um problema chamado `data race`, similar a `race condition`. Esse problema ocorre quando:

- 2 ou mais ponteiros acessarem um mesmo dado ao mesmo tempo.
- Pelo menos um dos ponteiros está sendo usado para gravar nos dados.
- Não há nenhum mecanismo sendo usado para sincronizar o acesso aos dados.

Podemos ter referência mutável de uma mesma variável em tempos diferentes, ou seja, em escopos diferentes:

```rs
fn main(){
    let mut s = String::from("hello");

    {
        let r1 = &mut s;
    } // r1 goes out of scope here, so we can make a new reference with no problems.

    let r2 = &mut s;
}
```

Podemos fazer um mix de referências mutáveis e não mutáveis, desde que de forma correta:

```rs
fn main(){
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    println!("{} and {}", r1, r2);
    // variables r1 and r2 will not be used after this point

    let r3 = &mut s; // no problem
    println!("{}", r3);
}
```

## O tipo Slice

`Slices` passam a referência de uma parte contínua de elementos de uma coleção.

```rs
fn main(){
    let s = String::from("hello world");

    let hello = &s[0..5];
    let world = &s[6..11];
}
```

```rs
fn main(){
    let a = [1, 2, 3, 4, 5];

    let slice = &a[1..3];

    assert_eq!(slice, &[2, 3]);
}
```