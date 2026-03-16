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
                  | interface_decl
                  | enum_decl
                  | const_decl
                  | type_alias
```

---

## 2. Declarações de Variáveis

```ebnf
var_decl        ::= "static"? ("val" | "mut val" | "const") IDENT (":" type)? "=" expr ";"

type_alias      ::= "type" IDENT "=" type ";"
```

---

## 3. Tipos

```ebnf
type            ::= primitive_type
                  | nullable_type
                  | generic_type
                  | optional_type
                  | result_type
                  | IDENT

result_type     ::= "result" "<" type "," type ">"

range_type      ::= "range" "<" type ">"

primitive_type  ::= "int" | "float" | "bool" | "string" | "void" | "dynamic"

nullable_type   ::= type "?"

generic_type    ::= IDENT "<" type_list ">"   (* inclui ref<T>, array<T>, etc. *)
type_list       ::= type ("," type)*
```

---

## 4. Funções

```ebnf
fn_decl         ::= annotation* fn_modifier* "fn" IDENT
                    generic_params? "(" param_list? ")" (":" type)?
                    block

fn_modifier     ::= "async" | "static" | "override" | "abstract"
                  | access_modifier

param_list      ::= param ("," param)*
param           ::= IDENT ":" type ("=" expr)?

generic_params  ::= "<" generic_param ("," generic_param)* ">"
generic_param   ::= IDENT (":" type)?

lambda          ::= "fn" "(" param_list? ")" (":" type)? "=>" (expr | block)

access_modifier ::= "public" | "private" | "protected"
```

---

## 5. Enums

```ebnf
enum_decl       ::= annotation* access_modifier? "enum" IDENT "{" enum_member* "}"

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

## 6. Classes e Interfaces

```ebnf
class_decl      ::= annotation* access_modifier? "abstract"? "class" IDENT
                    generic_params?
                    ("extends" IDENT)?
                    ("implements" IDENT ("," IDENT)*)?
                    "{" class_member* "}"

class_member    ::= var_decl
                  | fn_decl
                  | constructor_decl

constructor_decl ::= "fn" "(" param_list? ")" block

interface_decl  ::= "interface" IDENT generic_params?
                    ("extends" IDENT ("," IDENT)*)?
                    "{" interface_member* "}"

interface_member ::= fn_signature ";"
fn_signature     ::= fn_modifier* "fn" IDENT generic_params? "(" param_list? ")" (":" type)?
```

---

## 6. Statements

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
                  | unsafe_stmt
                  | match_expr
                  | block

block           ::= "{" stmt* "}"

assign_stmt     ::= lvalue assign_op expr ";"
assign_op       ::= "=" | "+=" | "-=" | "*=" | "/=" | "%="
lvalue          ::= IDENT
                  | expr "." IDENT
                  | expr "[" expr "]"

expr_stmt       ::= expr ";"
return_stmt     ::= "return" expr? ";"
unsafe_stmt     ::= "unsafe" block
```

---

## 7. Controle de Fluxo

```ebnf
if_stmt         ::= "if" "(" expr ")" block
                    ("else" "if" "(" expr ")" block)*
                    ("else" block)?

while_stmt      ::= "while" "(" expr ")" block

loop_stmt       ::= "loop" block

for_stmt        ::= "for" "(" IDENT "in" (expr | range_expr) ")" block

```

---

## 8. Expressões

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
                  )*

primary_expr    ::= literal
                  | IDENT
                  | "this"
                  | "super"
                  | alloc_expr
                  | "(" expr ")"
                  | await_expr
                  | try_expr
                  | match_expr
                  | lambda

await_expr      ::= "await" expr
try_expr        ::= "try" expr              (* desembrulha result.OK ou faz early return do result.ERROR *)
alloc_expr      ::= "alloc" "<" type ">" "(" arg_list? ")"

arg_list        ::= expr ("," expr)*

null_coalesce   ::= postfix_expr "??" expr
```

---

## 9. Literais

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

## 10. Anotações e Macros

```ebnf
annotation      ::= "@" IDENT ("(" annotation_args? ")")?
annotation_args ::= annotation_arg ("," annotation_arg)*
annotation_arg  ::= (IDENT "=")? expr

macro_call      ::= "#" IDENT "(" macro_args? ")"
macro_args      ::= expr ("," expr)*
```

---

## 11. Identificadores e Espaço em Branco

```ebnf
IDENT           ::= [a-zA-Z_] [a-zA-Z0-9_]*
WHITESPACE      ::= [ \t\n\r]+   (* ignorado *)
COMMENT_LINE    ::= "//" [^\n]*
COMMENT_BLOCK   ::= "/*" .* "*/"
```

---

## 12. Regras de Desambiguação do Parser

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

## 13. Gerenciamento de Memória — Reference Counting

A memória é gerenciada automaticamente por **contagem de referências (RC)**. Ao alocar com `alloc<T>`, o runtime mantém um contador interno. Quando o contador chega a zero, a memória é liberada automaticamente.

### Comportamento

```
val a = alloc<Ponto>(1, 2)   // RC = 1
val b = a                     // RC = 2 (b e a apontam para o mesmo objeto)
// fim do escopo de b → RC = 1
// fim do escopo de a → RC = 0 → liberado
```

### `val` vs `mut val`

| Declaração | Mutável |
| ---------- | ------- |
| `val`      | não     |
| `mut val`  | sim     |

### Referências (`ref<T>`)

`ref<T>` é o tipo de qualquer valor alocado via `alloc`. Atribuição de `ref<T>` incrementa o contador.

```
val a: ref<Ponto> = alloc<Ponto>(1, 2)
val b: ref<Ponto> = a    // mesmo objeto, RC incrementado
```

---

## 14. Bloco Unsafe e Dealloc

O bloco `unsafe` desabilita as proteções do RC e permite operações manuais de memória. Fora de `unsafe`, `dealloc` não pode ser chamado.

```ebnf
unsafe_stmt     ::= "unsafe" block

dealloc_expr    ::= "dealloc" "(" expr ")"   (* expr deve ser ref<T> *)
```

`dealloc` ignora o RC e libera a memória imediatamente. O ponteiro torna-se inválido após a chamada — o compilador não oferece garantias dentro de `unsafe`.

```
unsafe {
    val p: ref<Ponto> = alloc<Ponto>(1, 2)
    dealloc(p)       // liberação manual imediata
    // p inválido a partir daqui
}
```

> ⚠️ Usar `dealloc` fora de um bloco `unsafe` é erro de compilação.

---

## 15. Optional

`optional<T>` encapsula um valor que pode estar presente ou ausente (`null`). É a forma segura de lidar com ausência de valor — sem acessos nulos acidentais.

```ebnf
optional_type   ::= "optional" "<" type ">"
```

### Criação

```
val a: optional<int> = 42      // presente
val b: optional<int> = null    // ausente
```

### Safe call `?.`

Acessa atributos ou métodos apenas se o valor estiver presente. Retorna `null` caso contrário:

```
val name: optional<string> = user?.profile?.name
```

### Null coalescing `??`

Fornece um valor padrão quando o optional é `null`:

```
val display = name ?? "Anônimo"
```

### Encadeamento

```
val len: optional<int> = user?.name?.length ?? 0
```

> ⚠️ Acessar um `optional<T>` com `.` diretamente sem `?.` é erro de compilação.

---

## 16. Range

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
val r: range<int> = 1..10
val down: range<int> = 10..1..-1
```

> ⚠️ Step `0` é erro de compilação. Step negativo com `início < fim` também é erro.

---

## 17. Pattern Matching

`match` é uma expressão que compara um valor contra múltiplos padrões. O compilador exige que os casos sejam **exaustivos** (ou que exista `_`).

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
val result = match i {
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
val msg: string = match status {
    Status.OK(_) => "sucesso",
    Status.ERROR(e) => "falha: $e",
    _ => "desconhecido"
}
```

---

## 18. Result

`result<T, E>` representa uma operação que pode ter sucesso (`OK`) ou falhar (`ERROR`). Substitui exceptions — erros são valores.

```ebnf
result_type     ::= "result" "<" type "," type ">"
```

`result` é um enum built-in com dois variantes:

```
// equivalente conceitual:
enum result<T, E> {
    OK(T),
    ERROR(E)
}
```

### Criação

```
fn divide(a: int, b: int): result<int, string> {
    if (b == 0) {
        return result.ERROR("divisão por zero")
    }
    return result.OK(a / b)
}
```

### Consumo com `match`

```
val res = divide(10, 0)
match res {
    result.OK(value) => print("$value"),
    result.ERROR(err) => print("erro: $err")
}
```

### Consumo com `??` (fallback)

```
val value = divide(10, 0) ?? -1    // -1 se ERROR
```

### Propagação com `try`

`try expr` desembrulha `result.OK(T)` e retorna `T`. Se for `result.ERROR(E)`, faz **early return** do erro na função atual. A função que usa `try` deve retornar `result<_, E>`.

```
fn process(): result<string, string> {
    val a = try divide(10, 2)       // a: int = 5
    val b = try divide(a, 0)        // early return result.ERROR("divisão por zero")
    return result.OK("$b")
}
```

> ⚠️ Usar `try` em função que não retorna `result<_, E>` é erro de compilação.

---

## ⚠️ Em Aberto / A Decidir

| Tópico            | Questão                                                       |
| ----------------- | ------------------------------------------------------------- | ----------------------------------------------------- |
| Allocator         | Haverá `dealloc<T>(ref<T>)` simétrico, ou memória gerenciada? | Ownership — memória liberada ao fim do escopo do dono |
| `struct`          | Haverá structs separados de classes?                          |
| `string`          | Interpolação com `${}` ou outro delimitador?                  |
| Deref `*`         | Haverá derreferência explícita ou é automática?               |
| `unsafe`          | Haverá blocos unsafe para operações de baixo nível?           |
| Generics bounds   | `T: Interface` ou outra sintaxe de constraints?               |
| Multiline strings | Usar `"""..."""` ou backtick?                                 |
