# Basics

- **Dynamically-typed**  
  Elixir checks variable types only at runtime.
- **Functional-Paradigm**
  Elixir is majorly a functional programming language, it also has other paradigms like Metaprogramming, Process-Oriented Programming(Actor Model of Concurrency).
- **Usecases**
  
  [Read that could have been a BEAM](https://vereis.com/posts/you_built_an_erlang)
- **Installin Elixir**
  Follow this [link](https://elixir-lang.org/install.html) based on your choice of OS.
- **Try Interactive Mode**
  Like python you have an interactive shell, type iex in the shell and you can experiment for quick prototype scripts.
- **Basic Data Types**
  - **Integers**
    Support for binary octal and hexadecimal comes built-in
    Go to iex and type the following line by line 255, obo110, 0o664, 0x1F.
- **Pattern matching** (`=`)  
  You can bind any value to a variable name using `=` opertor, and re-bind variables of any type freely.
- A variable may hold any type of value.

---

### Modules

- Modules are the basis of code organization in Elixir.
- A module is visible to all other modules.
- Modules allow us to organize functions into a namespace
- Defined with `defmodule`.

---

### Named Functions

- Must be defined inside a module.
- Use `def` for public functions, `defp` for private functions.
- The value of the last expression is implicitly returned.
- Short one-line syntax is supported.

```elixir
def increment(n) do
  n + 1
end

defp private_increment(n) do
  n + 1
end

def short_increment(n), do: n + 1
