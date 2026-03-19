# Compiler — Parser

O Parser é a segunda fase do frontend de compilação. Recursive descent **hand-written**, sem gerador de parser, consome o `TokenArray` produzido pelo [Lexer](lexer.md) e emite uma AST index-based armazenada na [Compilation Database](compilation-db.md) sob a chave `getAst`. O Parser nunca aborta — erros de sintaxe são representados como `error_node` na própria árvore.

---

## Estrutura da AST

A AST é inspirada em `std.zig.Ast`. Todos os nós vivem em arrays contíguos indexados por `Node.Index` (`u32`) — não há ponteiros entre nós.

### `Ast`

```zig
const Ast = struct {
    source:      []const u8,
    tokens:      TokenArray,
    nodes:       Node.List,      // MultiArrayList de Node
    extra_data:  []Node.Index,   // dados extras referenciados por índice
    errors:      []ParseError,
    arena:       ArenaAllocator,
};
```

### `Node`

```zig
const Node = struct {
    tag:        Node.Tag,    // tipo do nó (enum)
    main_token: Token.Index, // token "principal" do nó (usado para spans e diagnósticos)
    data: struct {
        lhs: Node.Index,
        rhs: Node.Index,
    },
};
```

Nós com mais de dois filhos — por exemplo, chamadas de função com N argumentos ou listas de parâmetros — armazenam `{start, end}` em `lhs`/`rhs` e referenciam `extra_data[start..end]` para os índices dos filhos adicionais.

> **TBD:** A lista completa de `Node.Tag` (uma tag por construct da gramática) está reservada para o spec de implementação. Será definida antes da implementação.

---

## Índices e Cache

Todos os nós da AST são referenciados por `Node.Index` (`u32`), nunca por ponteiro. Isso garante:

| Propriedade | Descrição |
|-------------|-----------|
| Memória contígua | Todos os nós vivem em `MultiArrayList` — campos `tag`, `main_token` e `data` em arrays separados e paralelos |
| Cache-friendly | Traversal acessa memória sequencial, não alocações espalhadas no heap |
| Arena única | Um `ArenaAllocator` por `Ast` é dono de toda a memória dos nós — descartado integralmente quando o source muda |
| Compatibilidade com CDB | `Node.Index` pode ser armazenado e comparado entre queries sem realocar ponteiros |

---

## Error Nodes e Recuperação de Erros

Quando o Parser encontra um token inesperado, o processo é:

1. Registra um `ParseError { tag, token }` no array `errors` da `Ast`
2. Insere um `Node.Tag.error_node` na AST com o span do material problemático
3. Avança tokens até um **ponto de sincronização** — exatamente: `;`, `}`, `fn`, `class`, `struct`, `enum`, `interface`, `if`, `for`, `while`, `loop`, `return`
4. Retoma o parsing normalmente a partir desse ponto

Esse mecanismo garante que um erro em uma função não interrompe o parsing do restante do arquivo. Em modo LSP, a AST resultante pode conter múltiplos error nodes e ainda assim é utilizável para diagnósticos, completions e hover.

---

## Desambiguação `IDENT<`

Conforme a seção 14 da gramática ("Regras de Desambiguação do Parser"), a sequência `IDENT <` é ambígua entre um tipo genérico e um operador de comparação. A resolução é feita por lookahead no Parser — o Lexer emite os tokens normalmente sem distinção.

### Processo

1. Salvar a posição atual no `TokenArray`
2. Tentar parsear como `generic_type`
3. Aceitar como genérico **se e somente se** o conteúdo entre `<` e `>` for uma sequência separada por `,` de produções `type` válidas — sem operadores binários, sem expressões. Regra positiva: o token após `<` deve ser `IDENT` ou `primitive_type`, seguido de `,` (outro tipo) ou `>` (fechamento direto). Qualquer outro token invalida a leitura como genérico.
4. Caso contrário: retroceder (backtrack) e interpretar `<` como operador de comparação

### Exemplos

```bela
foo<int>        // generic_type — int é um tipo válido
foo<Bar, Baz>   // generic_type — Bar e Baz são tipos válidos
a < b           // comparação — b não é seguido de >
a < b > c       // comparação — contexto de expressão, não tipo
```
