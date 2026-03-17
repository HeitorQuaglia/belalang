# Gramática Formal — Linguagem [Bela]

> Notação: EBNF estendida. `?` = opcional, `*` = zero ou mais, `+` = um ou mais, `|` = alternativa, `(...)` = agrupamento.

---

## 1. Estrutura do Programa

```ebnf
program         ::= module_decl? import_decl* top_level_decl*

module_decl     ::= "module" IDENT ";"

import_decl     ::= "from" module_path "use" import_list ";"
module_path     ::= IDENT ("." IDENT)*
import_list     ::= import_item ("," import_item)*
import_item     ::= IDENT ("as" IDENT)?

top_level_decl  ::= fn_decl
                  | class_decl
                  | struct_decl
                  | interface_decl
                  | enum_decl
                  | var_decl
                  | type_alias
```

### Visibilidade

Todas as declarações top-level são **públicas por padrão** — não há keyword de exportação. A convenção para indicar que um símbolo é de uso interno ao módulo é o **prefixo `_`**:

```
fn processarPedido(order) { ... }    // público — acessível por outros módulos
fn _validarCampos(fields) { ... }    // privado por convenção — uso interno

class Servico { ... }                // público
class _Cache { ... }                 // privado por convenção
```

> O prefixo `_` é uma convenção semântica, não enforçada pelo compilador. Ferramentas (LSP, linter) podem emitir warnings se símbolos `_` forem importados por outros módulos.

---

## 2. Declarações de Variáveis

Bela é **dinamicamente tipada por padrão**. Variáveis não precisam de keyword — a primeira atribuição cria a variável no escopo atual. Atribuições seguintes no mesmo escopo reatribuem.

```ebnf
var_decl        ::= ("const" | "final")? IDENT (":" type_annotation)? "=" expr ";"
type_annotation ::= "fixed" "<" type ">" | type    (* type sem fixed é apenas hint/documentação *)

type_alias      ::= "type" IDENT "=" type ";"
```

| Forma | Mutável | Tipo |
| ----- | ------- | ---- |
| `a = 10` | sim | dinâmico |
| `a: int = 10` | sim | hint (documentação, warning) |
| `a: fixed<int> = 10` | sim | enforçado (erro em runtime) |
| `const a = 10` | não (runtime) | dinâmico |
| `final a = 10` | não (compile-time) | — |

> ⚠️ `: type` sem `fixed` é apenas uma anotação informativa — não impede reatribuição com tipo diferente. Apenas `fixed<T>` enforça o tipo.

---

## 3. Tipos

```ebnf
type            ::= primitive_type
                  | generic_type
                  | result_type
                  | fn_type
                  | IDENT

fn_type         ::= "async"? "fn" "(" type_list? ")" (":" type)?

result_type     ::= "result" "<" type "," type ">"

range_type      ::= "range" "<" type ">"

primitive_type  ::= "int" | "float" | "bool" | "string" | "void" | "dynamic"

generic_type    ::= IDENT "<" type_list ">"   (* inclui array<T>, Future<T>, fixed<T>, etc. *)
type_list       ::= type ("," type)*
```

> Bela é dinamicamente tipada — `dynamic` é o tipo padrão implícito. Generics são opcionais: `array` equivale a `array<dynamic>`.

### Tipos Built-in Especiais

| Tipo | Descrição |
|------|-----------|
| `array<T>` | Sequência mutável de elementos do tipo T |
| `result<T, E>` | Resultado de operação que pode falhar (ver seção 19) |
| `Future<T>` | Valor produzido por `async fn` — resolvido pelo event loop (ver seção 23) |
| `range<T>` | Sequência de valores entre dois extremos (ver seção 17) |

---

## 4. Funções

```ebnf
fn_decl         ::= annotation* fn_modifier* "fn" IDENT
                    generic_params? "(" param_list? ")" (":" type)?
                    block

fn_modifier     ::= "async" | "static" | "override" | "abstract"

param_list      ::= param ("," param)*
param           ::= IDENT (":" type)? ("=" expr)?
                  (* tipo opcional — warning quando inferência não é possível; obrigatório com fixed<T> *)

generic_params  ::= "<" generic_param ("," generic_param)* ">"
generic_param   ::= IDENT (":" type)?

lambda          ::= "fn" "(" param_list? ")" (":" type)? "=>" (expr | block)

access_modifier ::= "public" | "private" | "protected"
                  (* access_modifier é válido apenas no contexto de membros de classe *)
```

### Semântica de Captura em Lambdas

Lambdas capturam variáveis do escopo externo **por referência** (o RC do objeto sobe). A variável capturada mantém o objeto vivo enquanto a lambda existir.

```
multiplicador = fn(fator) {
    return fn(n) => n * fator    // captura 'fator' por referência
}

dobro = multiplicador(2)
dobro(5)    // 10 — 'fator' ainda acessível via RC
```

Para **modificar** uma variável do escopo pai (não apenas lê-la), use `outer`:

```
total = 0
items.forEach(fn(x) {
    outer total = total + x    // modifica 'total' no escopo pai
})
// total = soma de items
```

> `outer` em variável inexistente no escopo pai é erro de compilação. Closures que formam ciclos de referência (A captura B, B captura A) podem causar vazamento de memória — não há cicle collector, apenas RC.

### Bounds em Parâmetros Genéricos

A sintaxe `T: Interface` restringe o parâmetro de tipo `T` a tipos que implementam a interface especificada. O bound é **enforcável em runtime** — se o tipo passado não implementar a interface, é erro de runtime. O compilador tenta emitir **warning em compile-time** quando possível.

Apenas **um bound por parâmetro** de tipo é permitido. Não há sintaxe para múltiplos bounds (e.g., `T: A + B` não existe).

```
fn max<T: Comparable>(a: T, b: T): T {
    if (a > b) { return a }
    return b
}

max(3, 7)             // OK: int implementa Comparable
max("abc", "def")     // OK: string implementa Comparable
max(Point{x:1,y:2}, Point{x:3,y:4})  // erro runtime: Point não implementa Comparable
```

---

## 5. Enums

```ebnf
enum_decl       ::= annotation* "enum" IDENT "{" enum_member* "}"

enum_member     ::= IDENT ("(" primitive_type ")")? ","?
```

Exemplos:

```
enum Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}

enum Status {
    OK(int),
    ERROR(string),
    PENDING
}
```

---

## 6. Structs

Structs são **value types imutáveis**, alocados na **stack** (flat na memória). Servem exclusivamente para agrupar dados — sem métodos, sem herança, sem `implements`.

```ebnf
struct_decl     ::= annotation* "struct" IDENT "{" struct_field* "}"

struct_field    ::= IDENT ":" type ";"
                  (* tipo obrigatório — necessário para layout flat na stack *)
```

| | Struct | Class |
|---|---|---|
| Memória | Stack (flat) | Heap (RC) |
| Semântica | Imutável, cópia | Referência |
| Métodos | Não | Sim |
| Herança | Não | Sim |
| Inicialização | `Name { field: value }` | `Name(args)` |

O compilador gera automaticamente o método `.with(field: value)` para criar cópias com campos alterados.

### Exemplos

```
struct Point {
    x: float
    y: float
}

p1 = Point { x: 1.0, y: 2.0 }
p2 = Point { ...p1, x: 10.0 }     // spread: cópia com override
p3 = p1.with(y: 5.0)               // conveniência gerada pelo compilador

struct Handler {
    onSuccess: fn(data: string): void
    onError: fn(err: string): void
}

h = Handler {
    onSuccess: fn(data) => println(data),
    onError: fn(err) => eprintln(err)
}
```

---

## 7. Classes e Interfaces

```ebnf
class_decl      ::= annotation* "abstract"? "class" IDENT
                    generic_params?
                    ("extends" IDENT)?
                    ("implements" IDENT ("," IDENT)*)?
                    "{" class_member* "}"

class_member    ::= class_var_decl
                  | fn_decl
                  | constructor_decl
                  | factory_decl

class_var_decl  ::= "static"? ("val" | "const") IDENT (":" type_annotation)? ("=" expr)? ";"
                  (* classes mantêm val/const para declaração de propriedades *)

constructor_decl ::= access_modifier? "constructor" "(" param_list? ")" block

factory_decl     ::= "factory" IDENT "(" param_list? ")" block
                   (* sempre public, sempre static — sugar para static fn que retorna o tipo da classe *)

interface_decl  ::= "interface" IDENT generic_params?
                    ("extends" IDENT ("," IDENT)*)?
                    "{" interface_member* "}"

interface_member ::= fn_signature ";"
fn_signature     ::= fn_modifier* "fn" IDENT generic_params? "(" param_list? ")" (":" type)?
```

> `factory` é syntax sugar para um método `static fn` que retorna o tipo da classe. Factories são sempre públicas. Dentro do body do factory, o constructor da classe é acessível independente da visibilidade.

### Exemplo: Constructor privado + Factories

```
class Connection {
    val host: string
    val port: int

    private constructor(host, port) {
        this.host = host
        this.port = port
    }

    factory connect(host, port) {
        if (port < 0) {
            panic("porta inválida")
        }
        return Connection(host, port)
    }

    factory localhost(port) {
        return Connection("127.0.0.1", port)
    }
}

// uso:
conn = Connection.connect("example.com", 8080)
local = Connection.localhost(3000)
// Connection("x", 80)  → erro: constructor é private
```

---

## 8. Statements

```ebnf
stmt            ::= var_decl
                  | assign_stmt
                  | expr_stmt
                  | if_stmt
                  | while_stmt
                  | for_stmt
                  | loop_stmt
                  | return_stmt
                  | break_stmt
                  | continue_stmt
                  | match_expr
                  | block

block           ::= "{" stmt* "}"

assign_stmt     ::= "outer"? lvalue assign_op expr ";"
assign_op       ::= "=" | "+=" | "-=" | "*=" | "/=" | "%="
lvalue          ::= IDENT
                  | expr "." IDENT
                  | expr "[" expr "]"

expr_stmt       ::= expr ";"
return_stmt     ::= "return" expr? ";"
```

### Escopo e Shadowing

Toda atribuição cria uma variável no escopo atual. Se uma variável com o mesmo nome existe num escopo pai, a nova **faz shadow** (não modifica a original). Para modificar uma variável de escopo pai, use `outer`:

```
total = 0
for (i in 1..10) {
    outer total = total + i   // modifica total do escopo pai
}
// total é 55
```

> ⚠️ `outer` em variável que não existe no escopo pai é erro de compilação.

---

## 9. Controle de Fluxo

```ebnf
if_stmt         ::= "if" "(" expr ")" block
                    ("else" "if" "(" expr ")" block)*
                    ("else" block)?

while_stmt      ::= "while" "(" expr ")" block

loop_stmt       ::= "loop" block

for_stmt        ::= "for" "(" IDENT "in" (expr | range_expr) ")" block

```

O `for...in` funciona com qualquer tipo que implemente a interface `Iterable<T>` (ver seção 20). Tipos built-in (array, range, Map, Set, string) são iteráveis implicitamente.

---

## 10. Expressões

```ebnf
expr            ::= assignment_expr

assignment_expr ::= ternary_expr

ternary_expr    ::= logic_or_expr ("?" expr ":" ternary_expr)?

logic_or_expr   ::= logic_and_expr ("||" logic_and_expr)*

logic_and_expr  ::= equality_expr ("&&" equality_expr)*

equality_expr   ::= relational_expr (("==" | "!=") relational_expr)*

relational_expr ::= bitwise_expr (("<=" | ">=" | "<" | ">") bitwise_expr)*

bitwise_expr    ::= shift_expr (("&" | "|") shift_expr)*   (* & apenas bitwise, não endereço *)

shift_expr      ::= additive_expr (("<<" | ">>" | "<<<" | ">>>") additive_expr)*

additive_expr   ::= mult_expr (("+" | "-") mult_expr)*

range_expr      ::= additive_expr ".." additive_expr (".." additive_expr)?
                  (* início..fim..step — ambos inclusive; step negativo = decrescente *)

mult_expr       ::= unary_expr (("*" | "/" | "%") unary_expr)*

unary_expr      ::= ("!" | "-") unary_expr
                  | power_expr

power_expr      ::= postfix_expr ("**" unary_expr)?

postfix_expr    ::= primary_expr (
                      "." IDENT
                    | "." IDENT "(" arg_list? ")"
                    | "[" expr "]"
                    | "?." IDENT
                    | "?." IDENT "(" arg_list? ")"
                    | "is" type                        (* type checking — retorna bool *)
                    | "as" type                        (* casting — sugar para .to(type) *)
                  )*

primary_expr    ::= literal
                  | IDENT
                  | "this"
                  | "super"
                  | struct_init
                  | "(" expr ")"
                  | await_expr
                  | try_expr
                  | match_expr
                  | lambda

await_expr      ::= "await" expr
try_expr        ::= "try" expr              (* desembrulha result.OK ou faz early return do result.ERROR *)

arg_list        ::= expr ("," expr)*

null_coalesce   ::= postfix_expr "??" expr

struct_init     ::= IDENT "{" struct_init_fields "}"
struct_init_fields ::= struct_init_field ("," struct_init_field)* ","?
struct_init_field  ::= "..." expr                    (* spread *)
                     | IDENT ":" expr                (* campo nomeado *)
```

---

## 11. Literais

```ebnf
literal         ::= INT_LIT
                  | FLOAT_LIT
                  | BOOL_LIT
                  | CHAR_LIT
                  | STRING_LIT
                  | "null"

BOOL_LIT        ::= "true" | "false"
INT_LIT         ::= DEC_LIT | HEX_LIT | OCT_LIT | BIN_LIT
DEC_LIT         ::= [0-9]+
HEX_LIT         ::= "0x" [0-9A-Fa-f]+
OCT_LIT         ::= "0" [0-7]+
BIN_LIT         ::= "0b" [01]+
FLOAT_LIT       ::= [0-9]* "." [0-9]+
STRING_LIT      ::= string_single | string_multi

string_single   ::= '"' string_part* '"'
string_multi    ::= '"""' string_part* '"""'

string_part     ::= raw_chars
                  | "$" IDENT
                  | "${" expr "}"
raw_chars       ::= ([^"$\\] | "\\" .)+
```

---

## 12. Anotações e Macros

```ebnf
annotation      ::= "@" IDENT ("(" annotation_args? ")")?
annotation_args ::= annotation_arg ("," annotation_arg)*
annotation_arg  ::= (IDENT "=")? expr

macro_call      ::= "#" IDENT "(" macro_args? ")"
macro_args      ::= expr ("," expr)*
```

---

## 13. Identificadores e Espaço em Branco

```ebnf
IDENT           ::= [a-zA-Z_] [a-zA-Z0-9_]*
WHITESPACE      ::= [ \t\n\r]+   (* ignorado *)
COMMENT_LINE    ::= "//" [^\n]*
COMMENT_BLOCK   ::= "/*" .* "*/"
```

---

## 14. Regras de Desambiguação do Parser

### Generics vs. Operadores de Comparação

A sequência `IDENT "<"` é ambígua entre um tipo genérico e uma comparação. A resolução é feita por **lookahead**:

**Regra:** Após `IDENT "<"`, o parser tenta interpretar como `generic_type`. Se o conteúdo entre `<` e `>` for uma sequência válida de tipos separados por `,`, trata como generic. Caso contrário, volta e interpreta `<` como operador de comparação.

```
foo<int>        → generic_type      (int é um tipo válido)
foo<bar, Baz>   → generic_type      (bar e Baz são tipos válidos)
a < b           → comparação        (b não é seguido de >)
a < b > c       → comparação        (contexto de expressão, não tipo)
```

**Critério de desambiguação:** O lookahead interpreta como generic se:

1. O token após `<` for um `IDENT` ou `primitive_type`
2. Seguido de `,` (outro tipo) ou `>` (fechamento)
3. Nenhum operador aritmético ou lógico aparecer entre `<` e `>`

---

## 15. Gerenciamento de Memória — Reference Counting

A memória é gerenciada automaticamente por **contagem de referências (RC)**. Alocação e liberação são implícitas — não há API explícita de memória.

```
a = Ponto(1, 2)              // alocado, RC = 1
b = a                         // RC = 2 (b e a apontam para o mesmo objeto)
// fim do escopo de b → RC = 1
// fim do escopo de a → RC = 0 → liberado automaticamente
```

---

## 16. Null, Safe Navigation e Null Coalescing

Qualquer variável pode conter `null` — Bela trata null como um valor livre, sem wrappers.

### Safe call `?.`

Acessa atributos ou métodos apenas se o receptor não for `null`. Retorna `null` caso contrário:

```
name = user?.profile?.name          // null se user ou profile for null
```

### Null coalescing `??`

Fornece um valor padrão quando a expressão é `null`:

```
display = name ?? "Anônimo"
value = divide(10, 0) ?? -1         // também funciona com result.ERROR
```

> ⚠️ Acessar um valor `null` com `.` diretamente (sem `?.`) é erro de runtime.

---

## 17. Range

`range<T>` representa uma sequência de valores entre dois extremos, ambos **inclusivos**.

```ebnf
range_expr  ::= additive_expr ".." additive_expr (".." additive_expr)?
```

### Formas

```
1..10          // 1, 2, 3, ..., 10        (step padrão = +1)
1..10..2       // 1, 3, 5, 7, 9           (step explícito)
10..1..-1      // 10, 9, 8, ..., 1        (decrescente)
10..1..-2      // 10, 8, 6, 4, 2          (decrescente com step)
```

### Uso em `for`

```
for (i in 1..10) { ... }
for (i in 10..1..-1) { ... }
```

### Como variável

```
r = 1..10
down = 10..1..-1
```

> ⚠️ Step `0` é erro de compilação. Step negativo com `início < fim` também é erro.

---

## 18. Pattern Matching

`match` é uma expressão que compara um valor contra múltiplos padrões. Recomenda-se incluir `_` (wildcard) para garantir cobertura — em tipagem dinâmica, exaustividade é verificada em **best-effort**.

```ebnf
match_expr          ::= "match" expr "{" match_arm* "}"

match_arm           ::= pattern ("if" expr)? "=>" (expr | block) ","?

pattern             ::= literal_pattern
                      | range_pattern
                      | constructor_pattern
                      | enum_pattern
                      | wildcard_pattern
                      | binding_pattern

literal_pattern     ::= INT_LIT | FLOAT_LIT | STRING_LIT | BOOL_LIT | "null"
range_pattern       ::= additive_expr ".." additive_expr
wildcard_pattern    ::= "_"
binding_pattern     ::= IDENT
constructor_pattern ::= IDENT "(" pattern_list? ")"
enum_pattern        ::= IDENT "." IDENT ("(" pattern_list? ")")?
pattern_list        ::= pattern ("," pattern)*
```

> **Regra de binding:** `IDENT` em lowercase é tratado como binding (captura o valor). Literais, `_` e constructors/enums qualificados são padrões fixos.

### Exemplos

#### Matching de valores e ranges

```
r = match i {
    1..5 => "pequeno",
    10 => "dez",
    _ => "inesperado"
}
```

#### Destructuring de classe

```
match point {
    Point(10, 10) => print("origem"),
    Point(20, _) => print("x=20"),
    Point(x, y) if x > 0 => print("positivo: $x, $y"),
    Point() => print("qualquer ponto")
}
```

#### Destructuring de enum (qualificado)

```
match status {
    Status.OK(code) => print("ok: $code"),
    Status.ERROR(msg) => print("erro: $msg"),
    Status.PENDING => print("aguardando")
}
```

#### Como expressão

```
msg = match status {
    Status.OK(_) => "sucesso",
    Status.ERROR(e) => "falha: $e",
    _ => "desconhecido"
}
```

---

## 19. Result

`result` representa uma operação que pode ter sucesso (`OK`) ou falhar (`ERROR`). Substitui exceptions — erros são valores. Parâmetros de tipo são opcionais.

```ebnf
result_type     ::= "result" ("<" type "," type ">")?
```

`result` é um enum built-in com dois variantes:

```
// equivalente conceitual:
enum result {
    OK(dynamic),
    ERROR(dynamic)
}
```

### Criação

```
fn divide(a, b) {
    if (b == 0) {
        return result.ERROR("divisão por zero")
    }
    return result.OK(a / b)
}
```

### Consumo com `match`

```
res = divide(10, 0)
match res {
    result.OK(value) => print("$value"),
    result.ERROR(err) => print("erro: $err")
}
```

### Consumo com `??` (fallback)

```
value = divide(10, 0) ?? -1    // -1 se ERROR
```

### Propagação com `try`

`try expr` desembrulha `result.OK` e retorna o valor interno. Se for `result.ERROR`, faz **early return** do erro na função atual.

```
fn process() {
    a = try divide(10, 2)       // a = 5
    b = try divide(a, 0)        // early return result.ERROR("divisão por zero")
    return result.OK("$b")
}
```

> ⚠️ O compilador emite **warning** quando `try` é usado em expressão que provavelmente não retorna `result`. Erro real é verificado em runtime.

---

## 20. Interfaces Built-in

Interfaces built-in são definidas pelo compilador e implementadas implicitamente pelos tipos que satisfazem os requisitos.

```ebnf
builtin_interface ::= "Comparable" | "Iterable" "<" type ">" | "Iterator" "<" type ">"
```

### Comparable

Habilita os operadores de ordenação `<`, `>`, `<=`, `>=`. Também permite uso como argumento em funções que exigem `T: Comparable`.

```
interface Comparable {
    fn compareTo(other: dynamic): int    // < 0, 0, ou > 0
}
```

Tipos que implementam implicitamente: `int`, `float`, `string`.

```
fn max<T: Comparable>(a: T, b: T): T {
    if (a.compareTo(b) > 0) { return a }
    return b
}

max(3, 7)           // 7
max("abc", "def")   // "def"
```

### Iterable\<T\> e Iterator\<T\>

Habilita o uso de um tipo em `for...in`. A classe implementa `Iterable<T>` retornando um `Iterator<T>`.

```
interface Iterator<T> {
    fn next(): T
    fn hasNext(): bool
}

interface Iterable<T> {
    fn iterator(): Iterator<T>
}
```

Tipos que implementam implicitamente: `array<T>`, `range<T>`, `Map<K,V>` (itera sobre pares), `Set<T>`, `string` (itera sobre caracteres).

```
class Contagem implements Iterable<int> {
    val inicio: int
    val fim: int

    constructor(inicio, fim) {
        this.inicio = inicio
        this.fim = fim
    }

    fn iterator(): Iterator<int> {
        return ContagemIterator(this.inicio, this.fim)
    }
}

class ContagemIterator implements Iterator<int> {
    val atual: int
    val fim: int

    constructor(atual, fim) {
        this.atual = atual
        this.fim = fim
    }

    fn hasNext(): bool { return this.atual <= this.fim }
    fn next(): int {
        val = this.atual
        this.atual = this.atual + 1
        return val
    }
}

for (n in Contagem(1, 5)) {
    println(n)    // 1, 2, 3, 4, 5
}
```

### Igualdade e `.equals()`

O operador `==` entre **objetos** (instâncias de classe) chama o método `.equals()` implicitamente. A implementação padrão compara **endereços de ponteiro** (identidade). Para igualdade por valor, faça override:

```
class Ponto {
    val x: int
    val y: int

    constructor(x, y) { this.x = x; this.y = y }

    fn equals(other: dynamic): bool {
        if (!(other is Ponto)) { return false }
        return this.x == other.x && this.y == other.y
    }
}

p1 = Ponto(1, 2)
p2 = Ponto(1, 2)
p1 == p2    // true — usa equals() com override
```

> Primitivos (`int`, `float`, `bool`, `string`) sempre comparam por valor.

### Serialização e `.toString()`

Todos os objetos possuem um método `.toString()` implícito utilizado em:
- Interpolação de strings: `"valor: $obj"` ou `"${obj}"`
- `println(obj)` e variantes

A implementação padrão serializa o objeto para **JSON**:

```
class Produto {
    val nome: string
    val preco: float
    constructor(nome, preco) { this.nome = nome; this.preco = preco }
}

p = Produto("Caneta", 2.50)
println(p)    // {"nome":"Caneta","preco":2.5}
```

Para customizar, faça override de `toString()`:

```
class Produto {
    // ...
    fn toString(): string {
        return "$nome (R$ $preco)"
    }
}

println(Produto("Caneta", 2.50))    // "Caneta (R$ 2.5)"
```

---

## 21. Operadores `is` e `as`

### `is` — Type Checking com Smart Cast

`expr is Type` retorna `bool`. Dentro de um bloco `if` que usa `is`, a variável é automaticamente tratada como o tipo verificado (**smart cast**):

```ebnf
postfix_expr ::= primary_expr ( ... | "is" type | "as" type )*
```

```
fn processar(valor: dynamic) {
    if (valor is string) {
        println(valor.toUpper())    // valor é string aqui — sem cast manual
    } else if (valor is int) {
        println(valor * 2)
    }
}
```

> Smart cast funciona quando o compilador pode garantir que o valor não foi reatribuído entre o `is` e o uso. Se a variável for `dynamic` e reatribuída, o smart cast não se aplica.

### `as` — Casting

`expr as Type` é **syntax sugar** para `expr.to(Type)`. Lança erro de runtime se a conversão não for possível:

```ebnf
postfix_expr ::= primary_expr ( ... | "as" type )*
```

```
val = "42" as int      // equivale a "42".to(int) → 42
num = 3.14 as int      // 3 (trunca)
bad = "abc" as int     // erro de runtime: não conversível
```

Tipos que suportam conversão via `as`:

| De | Para | Comportamento |
|----|------|---------------|
| `string` | `int` | Parse — erro se inválido |
| `string` | `float` | Parse — erro se inválido |
| `int` | `float` | Promoção |
| `float` | `int` | Truncamento |
| `int` | `string` | `toString()` |
| `float` | `string` | `toString()` |
| `bool` | `string` | `"true"` ou `"false"` |

---

## 22. Prelude

O **prelude** é o conjunto mínimo de símbolos disponíveis sem nenhum `from ... use ...`. É intencionalmente restrito para que o host tenha controle total sobre o que o script pode fazer.

### Disponível sem import

| Símbolo | Tipo | Descrição |
|---------|------|-----------|
| `panic` | `fn(msg: string): void` | Encerra a VM imediatamente (ver seção abaixo) |
| `typeof` | `fn(val: dynamic): string` | Retorna o tipo como string: `"int"`, `"string"`, `"array"`, etc. |

### Tipos e keywords (parte da gramática)

Os seguintes identificadores são reservados e fazem parte da gramática, não do prelude:

`int`, `float`, `bool`, `string`, `void`, `dynamic`, `null`, `true`, `false`, `result`, `array`, `Future`, `range`

### `panic`

`panic` é uma função built-in que **encerra a VM** (não o processo host). É irrecuperável dentro de Bela — não há `try`/`catch` para panic. O host captura o panic via status `BELA_PANIC` na C API.

```
fn conectar(host: string, port: int) {
    if (port < 0 || port > 65535) {
        panic("porta inválida: $port")    // encerra a VM
    }
    // ...
}
```

**Quando usar `panic` vs `result.ERROR`:**

| Situação | Use |
|----------|-----|
| Erro esperado, recuperável (ex: arquivo não encontrado) | `result.ERROR` |
| Bug do programador (estado impossível, invariante violada) | `panic` |
| Argumento inválido que indica uso incorreto da API | `panic` |

### I/O não está no prelude

`print`, `println`, `eprint`, `eprintln` devem ser importados explicitamente de `io`:

```
from io use println, eprintln

fn main() {
    println("hello")
}
```

> Isso permite que o host controle completamente o I/O do script — substituindo ou bloqueando o acesso ao console.

---

## 23. Concorrência — Modelo Assíncrono

Bela usa um modelo de **event loop single-thread**, similar ao JavaScript. Não há threads reais — toda execução concorrente é cooperativa.

### `async fn` e `Future<T>`

Uma função marcada com `async` retorna implicitamente um `Future<T>`, onde `T` é o tipo de retorno declarado:

```
async fn buscar(url: string): string {
    resp = await client.get(url)    // suspende até resposta
    return resp.body                // Future<string> implícito
}
```

`await` suspende a função atual e cede o controle ao event loop. Só pode ser usado dentro de `async fn`.

### Regras

- `await` só é válido dentro de `async fn`
- `async fn` pode ser chamada de contexto síncrono, mas o resultado é um `Future<T>` não resolvido
- Para obter o valor de um `Future<T>` em contexto síncrono, use `concurrency.await_sync` (bloqueante — evitar em embedding)
- O event loop do host pode ser integrado ao loop de Bela via C API

### Integração com o host

O host controla o event loop:

```c
// Avançar o event loop manualmente (ex: a cada frame de jogo)
bela_vm_tick(vm);

// Ou delegar ao loop interno de Bela (modo standalone)
bela_vm_run_event_loop(vm);
```

### Coordenação de Futures (módulo `concurrency`)

Ver documentação do módulo `concurrency` para: `all`, `race`, `any`, `timeout`, `Channel<T>`.