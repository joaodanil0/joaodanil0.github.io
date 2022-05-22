# Introdução

- [ ] Instalando o Rust no linux
- [ ] Escrevendo o *Hello World!*
- [ ] Usando `cargo` (gerenciador de pacotes e sistema de build)

## 1. Instalando o Rust no linux

Devemos baixar o `rustup`, que é responspável por baixar o `rust` e fazer a instalação

```
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
```

Algumas informações devem aparecer

>
Welcome to Rust!
>
This will download and install the official compiler for the Rust<br>
programming language, and its package manager, Cargo.
>
Rustup ==metadata and toolchains will be installed== into the Rustup<br>
home directory, located at:
>
  /home/jao/Documents/Programs/rust/rustup/
>
This can be modified with the ==RUSTUP_HOME== environment variable.
>
The ==Cargo home directory== located at:
>
  /home/jao/Documents/Programs/rust/cargo/
>
This can be modified with the ==CARGO_HOME== environment variable.
>
The cargo, rustc, rustup and other commands will be added to<br>
==Cargo's bin directory==, located at:
>
  /home/jao/.cargo/bin
>
This path will then be added to your PATH environment variable by<br>
modifying the profile files located at:
>
  /home/jao/.profile
  /home/jao/.zshenv
>
You can uninstall at any time with ==rustup self uninstall== and<br>
these changes will be reverted.
>
Current installation options:
>
   default host triple: x86_64-unknown-linux-gnu<br>
     default toolchain: stable (default)<br>
               profile: default<br>
  modify PATH variable: yes
>
1) Proceed with installation (default) <br>
2) Customize installation<br>
3) Cancel installation<br>

Basta digitar 1 e a instalação irá começar. 

Se tudo occorrer bem, a seguinte mensagem deve aparecer

>
Rust is installed now. Great!
>
To get started you may need to restart your current shell.
This would reload your PATH environment variable to include
Cargo's bin directory (/home/jao/Documents/Programs/rust/cargo//bin).
>
To configure your current shell, run:
source /home/jao/Documents/Programs/rust/cargo//env

### 1.1 Alguns comandos úteis

- rustup update
- rustup self uninstall
- rustc --version
- rustup doc

## 2. Escrevendo o Hello World!

Crie um arquivo chamado `main.rs` e adicione o conteúdo

```{.rs title=main.rs}
fn main() {
	println!("Hello World!");
}
```

Para compilar e executar, basta digitar no terminal

```
rustc main.rc
./main
```

O resultado será o texto ==Hello World!==

### 2.1 Alguns comandos úteis

- rustfmt (para formatar o código)

## 3. Usando cargo (gerenciador de pacotes e sistema de build)

Para criar um projeto utilizando o `cargo`, basta usar o comando

```
cargo new hello_cargo
```
 
 Um arquivo `main.rs` será criado dentro da pasta `hello_cargo/src/`. Para compilar o código, basta digitar
 
 ```
 cargo build
 ```
 
 dentro da pasta hello_cargo, a pasta `target/`será criada e o executável está em `hello_cargo/target/debug`, chamado de ==hello_cargo==

### 3.1 Comandos úteis
 
 - cargo run (para executar o programa)
 - cargo check (para fazer as checagens no código, sem compilar)
 - cargo build --release (para fazer uma build de release, por padrão a build é de debug).
