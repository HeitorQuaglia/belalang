# Compiler — Lexer

O Lexer é a primeira fase do frontend de compilação. Ele transforma o source bruto (bytes UTF-8) em um array plano de tokens (`TokenArray`) consumido pelo [Parser](parser.md). Opera em **modo lazy** (on-demand via `next()`, usado pelo Parser diretamente) ou em **modo batch** para popular o `TokenArray` cacheado na [Compilation Database](compilation-db.md).

---

## Representação dos Tokens

Tokens não carregam o texto — apenas um `tag` e um `start` (byte offset no source). O texto de um token é recuperado via slice do source quando necessário:

```zig
const Token = struct {
    tag:   TokenTag,  // enum: .ident, .kw_fn, .plus, ...
    start: u32,       // byte offset no source
};
```

O comprimento de cada token é derivado do offset do token seguinte: `source[token.start..next_token.start]`. Para garantir que o último token real também tenha um sucessor, o Lexer sempre acrescenta um token sentinela `.eof` na posição `source.len` ao final do `TokenArray`. Isso mantém o `TokenArray` compacto e cache-friendly — nenhum token carrega uma cópia da string.

---

## TokenTag

> **TBD:** A lista completa e definitiva de `TokenTag` está reservada para o spec de implementação. A tabela abaixo documenta o conjunto representativo utilizado como referência de design.

### Keywords

| Tag | Lexema |
|-----|--------|
| `kw_fn` | `fn` |
| `kw_class` | `class` |
| `kw_struct` | `struct` |
| `kw_enum` | `enum` |
| `kw_interface` | `interface` |
| `kw_if` | `if` |
| `kw_else` | `else` |
| `kw_for` | `for` |
| `kw_while` | `while` |
| `kw_loop` | `loop` |
| `kw_return` | `return` |
| `kw_match` | `match` |
| `kw_const` | `const` |
| `kw_final` | `final` |
| `kw_outer` | `outer` |
| `kw_async` | `async` |
| `kw_await` | `await` |
| `kw_try` | `try` |
| `kw_null` | `null` |
| `kw_true` | `true` |
| `kw_false` | `false` |
| `kw_from` | `from` |
| `kw_use` | `use` |
| `kw_module` | `module` |
| `kw_type` | `type` |
| `kw_override` | `override` |
| `kw_abstract` | `abstract` |
| `kw_val` | `val` |
| `kw_static` | `static` |
| `kw_this` | `this` |
| `kw_super` | `super` |
| `kw_break` | `break` |
| `kw_continue` | `continue` |
| `kw_in` | `in` |
| `kw_as` | `as` |
| `kw_is` | `is` |
| `kw_result` | `result` |
| `kw_factory` | `factory` |
| `kw_constructor` | `constructor` |
| `kw_extends` | `extends` |
| `kw_implements` | `implements` |

### Literais

| Tag | Descrição |
|-----|-----------|
| `int_lit` | Inteiro decimal, hexadecimal (`0x`), octal (`0`), binário (`0b`) |
| `float_lit` | Ponto flutuante (`[0-9]* "." [0-9]+`) |
| `char_lit` | Caractere delimitado por aspas simples |
| `string_literal_part` | Segmento de texto literal dentro de uma string interpolada |
| `interp_ident` | Interpolação simples `$ident` |
| `interp_expr_begin` | Abertura de interpolação de expressão `${` |
| `interp_expr_end` | Fechamento de interpolação de expressão `}` |

### Identificador

| Tag | Descrição |
|-----|-----------|
| `ident` | `[a-zA-Z_][a-zA-Z0-9_]*` — qualquer identificador que não seja uma keyword |

### Operadores e Pontuação

| Tag | Lexema | Tag | Lexema |
|-----|--------|-----|--------|
| `plus` | `+` | `minus` | `-` |
| `star` | `*` | `slash` | `/` |
| `percent` | `%` | `eq` | `=` |
| `eq_eq` | `==` | `bang_eq` | `!=` |
| `lt` | `<` | `gt` | `>` |
| `lt_eq` | `<=` | `gt_eq` | `>=` |
| `amp_amp` | `&&` | `pipe_pipe` | `\|\|` |
| `bang` | `!` | `dot` | `.` |
| `dot_dot` | `..` | `dot_dot_dot` | `...` |
| `question` | `?` | `colon` | `:` |
| `semicolon` | `;` | `comma` | `,` |
| `at` | `@` | `hash` | `#` |
| `l_paren` | `(` | `r_paren` | `)` |
| `l_brace` | `{` | `r_brace` | `}` |
| `l_bracket` | `[` | `r_bracket` | `]` |
| `arrow` | `=>` | `question_dot` | `?.` |
| `question_question` | `??` | | |

### Especiais

| Tag | Descrição |
|-----|-----------|
| `.eof` | Sentinela de fim de arquivo — sempre presente na posição `source.len` |
| `.invalid` | Caractere não reconhecido — emitido sem abortar o Lexer |
| `.trivia` | Whitespace ou comentário — emitido apenas em modo LSP |

---

## Tokens Especiais

### Interpolação de Strings

Strings interpoladas são decompostas pelo Lexer em uma sequência de tokens. A gramática define duas formas de interpolação (seção 11):

- `$ident` — interpolação simples de identificador
- `${expr}` — interpolação de expressão arbitrária

O Lexer emite os seguintes tokens para cobrir ambas as formas:

| Token | Representa |
|-------|------------|
| `string_literal_part` | Segmento de texto bruto entre interpolações |
| `interp_ident` | `$identificador` — interpolação simples |
| `interp_expr_begin` | `${` — abertura de interpolação de expressão |
| `interp_expr_end` | `}` — fechamento de interpolação de expressão |

**Exemplo — interpolação simples:**

```bela
"olá $nome!"
```

Sequência de tokens emitida:

```
string_literal_part   → "olá "
interp_ident          → "nome"
string_literal_part   → "!"
```

**Exemplo — interpolação de expressão:**

```bela
"total: ${a + b} itens"
```

Sequência de tokens emitida:

```
string_literal_part   → "total: "
interp_expr_begin     → "${"
ident                 → "a"
plus                  → "+"
ident                 → "b"
interp_expr_end       → "}"
string_literal_part   → " itens"
```

> **Nota:** Os rótulos acima são ilustrativos. Os tokens não carregam payload — o texto de cada token é sempre recuperado via `source[token.start..next_token.start]`.

> **Nota — Lexer stateful durante interpolações:** O Lexer mantém um contador de profundidade de interpolação. Ao emitir `interp_expr_begin` (`${`), entra em modo de interpolação e passa a emitir `interp_expr_end` em vez de `r_brace` para o `}` de fechamento correspondente. Fora desse modo (i.e., em qualquer outro contexto), `}` é sempre emitido como `r_brace`. Isso garante que os dois tokens — `r_brace` e `interp_expr_end` — nunca sejam ambíguos para o Parser.

### Strings Multi-linha

Strings delimitadas por `"""..."""` usam os mesmos tipos de token (`string_literal_part`, `interp_ident`, `interp_expr_begin`, `interp_expr_end`) — não há distinção de tag entre string simples e multi-linha.

---

## Modo LSP vs Modo Compilação

O comportamento do Lexer em relação a whitespace e comentários difere por modo de operação. O modo é selecionado pela [Compilation Database](compilation-db.md) no momento em que o `TokenArray` é populado via `getTokenArray`.

| Elemento | Modo Compilação | Modo LSP |
|----------|-----------------|----------|
| Whitespace | Descartado | Emitido como `.trivia` |
| `// comentário` | Descartado | Emitido como `.trivia` |
| `/* comentário */` | Descartado | Emitido como `.trivia` |

Em modo LSP, tokens de trivia são anexados ao token seguinte. Isso permite ao LSP calcular ranges exatos para comentários e preservar a formatação original do source — necessário para recursos como formatting, hover sobre comentários e ranges de diagnósticos precisos.

---

## Tratamento de Erros

O Lexer nunca aborta. Ao encontrar um caractere não reconhecido:

1. Emite um token `.invalid` com `start` apontando para o caractere problemático
2. Avança um byte e continua o processo de tokenização normalmente

A responsabilidade de reagir a tokens `.invalid` é do [Parser](parser.md), que os trata via seu mecanismo de recuperação de erros (error nodes e pontos de sincronização).
