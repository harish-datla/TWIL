This is my personal notes(handtyped and small improvements) after reading from https://elixirschool.com/en/lessons/basics/basics
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
    Apart from decimal representation, support for binary octal and hexadecimal comes built-in  
    Go to iex and type the following line by line 255, obo110, 0o664, 0x1F.
  - **Floats**
    Floating point numbers requires a decimal after at least one digit, they have 64-bit double precision and support e for exponent values:
    Go to iex and type 3.14, .014 and 1.0e-10.
  - **Booleans**
    Elixir supports `true` and `false` as booleans, everything is truthy except for `false` and `nil`
  - **Atoms**
    An atoms is a constant whose name is its value. If you ever worked in ruby, this should ring a [bell](https://www.codecademy.com/learn/learn-ruby/modules/learn-ruby-hashes-and-symbols-u/cheatsheet)
    booleans true and false are also atoms :true and :false respectively.
    ```elixir
    iex> is_atom(true)
    true
    iex> is_boolean(:true)
    true
    iex> :true === true
    true
    ```
    Name of modules in elixir are also atoms, MyApp.MyModule is a valid atom, even if no such module has been declared yet.
    ```elixir
    iex> is_atom(MyApp.MyModule)
    true
    ```
    Atoms are also used to reference modules from Erlang libraries, including built in ones.
    ```elixir
    iex> :crypto.storng_rand_bytes 4
    <<40, 217, 80, 65>>
    ```
  - **Strings**
    Strings in Elixir are [UTF-8](https://www.youtube.com/watch?v=nhN8larXM2w) encoded and are wrapped in double quotes.
    
- **Basic Operations**
  - **Arithmetic**
    Elixir supports basic operators like `+`,`-`,`*` and `/`
    Note: / will always return a float.
    ```elixir
    iex> 2 + 2
    4
    iex> 2 - 1
    1
    iex> 2*5
    10
    iex> 10/5
    2.0
    ```
    If you need integer division or the division remainder(mod), elixir comes with two hekpful functions to achieve this, namely `div` and `rem`.
    ```elixir
    iex>div(10,5)
    2
    iex>rem(10,3)
    1
    ```
  - **Boolean**
    Elixir supports `||`, `&&` and `!` operators and you can use these for any types.
    ```elixir
    iex> -20 || true
    -20
    iex> false || 42
    42
    iex> 42 && true
    true
    iex> 42 && nil
    nil
    iex> !42
    false
    iex> !false
    true
    ```
    Elixir also supports three additional operators whose first argument must be a boolean(`true` or `false`):
    ```elixir
    iex> true and 42
    42
    iex> false or true
    true
    iex> not false
    true
    iex> 42 and true
    ** (BadBooleanError) expected a boolean on left-side of "and", got: 42
    iex> not 42
    ** (ArgumentError) argument error
    ```
    NoteL Elixir's `and` and `or` actually map to `andalso` and `orelse` in Erlang.
  - **Comparison**
    Elixir comes with all the comparison operators we're used to `==`, `!=` , `===`, `!==`, `<=` , `>=` , `<` and `>`
    ```elixir
    iex> 1 > 2
    false
    iex> 1 != 2
    true
    iex> 2 == 2
    true
    iex> 2 <= 3
    true
    ```
    For strict omparison of integers and floats, use ===:
    ```elixir
    iex> 2 == 2.0
    true
    iex> 2 === 2.0
    false
    ```
    An important featur of elixir is that any two types can be compared, same will reflect in your sorting algorithms.
    `number < atom < reference < function < port < pid < tuple < map < list < bitstring`

  - **String Interpolation**
    If you've used Ruby, string interpolation in elixir will look familiar
    ```elixir
    iex> name = "Sean"
    "Sean"
    iex> "Hello #{name}"
    "Hello Sean"
    ```
  - **String Concatenation**
    String concatenation use the `<>` operator:
    ```elixir
    iex> name = "Sean"
    "Sean"
    iex> "Hello" <> name
    "Hello Sean"
    ```
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
```


