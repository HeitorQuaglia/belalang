# Compiler — Análise Semântica (Sema)

A Sema é a terceira fase do frontend de compilação. Ela percorre a AST produzida pelo [Parser](parser.md) e produz um `SemaResult` separado com diagnósticos, tabela de símbolos, tipos inferidos e referências de definição. A Sema **não modifica a AST** — apenas a anota com informações derivadas.

---

## SemaResult

```zig
const SemaResult = struct {
    diagnostics:  []Diagnostic,  // erros e warnings com span (token index)
    symbol_table: SymbolTable,   // mapa de escopo → símbolos declarados
    node_types:   []TypeInfo,    // Node.Index → tipo inferido (best-effort)
    refs:         []Ref,         // Node.Index → Node.Index da definição
};
```

| Campo | Tipo | Descrição |
|---|---|---|
| `diagnostics` | `[]Diagnostic` | Erros e warnings com span (token index) |
| `symbol_table` | `SymbolTable` | Mapa de escopo → símbolos declarados |
| `node_types` | `[]TypeInfo` | `Node.Index` → tipo inferido (best-effort) |
| `refs` | `[]Ref` | `Node.Index` → `Node.Index` da definição |

O campo `refs` habilita diretamente o recurso de "go-to-definition" do [LSP](../toolchain/lsp.md).

---

## Diagnostic

Um `Diagnostic` representa um erro ou warning emitido durante o walk semântico.

```zig
const Diagnostic = struct {
    tag:      DiagnosticTag,  // identifica o tipo de diagnóstico (enum)
    severity: Severity,       // .err ou .warning
    token:    Token.Index,    // posição no source (span via TokenArray)
    message:  []const u8,     // mensagem legível
};
```

| Campo | Descrição |
|---|---|
| `tag` | Identifica o tipo de diagnóstico — permite filtros e formatação diferenciada |
| `severity` | `.err` bloqueia o Codegen; `.warning` não bloqueia |
| `token` | `Token.Index` que aponta para o token principal do construct problemático — o span é derivado do `TokenArray` |
| `message` | Mensagem textual destinada ao usuário |

Diagnósticos com `severity == .err` impedem a execução da fase de Codegen. Warnings são reportados mas não bloqueiam a geração de bytecode.

---

## Verificações

A Sema executa as seguintes verificações durante o walk da AST:

| Verificação | Resultado |
|---|---|
| Resolução de nomes | Vincula cada `IDENT` à sua declaração, respeitando shadowing |
| `outer` em variável inexistente no escopo pai | Erro |
| `await` fora de `async fn` | Erro |
| `try` sem contexto `result` | Warning |
| `try` fora de um corpo de função | Erro — não há alvo para o early return |
| `fixed<T>` violável estaticamente | Warning ou Erro (dual severity — ambos podem ser emitidos dependendo do grau de certeza estática) |
| Bound `T: Interface` violado estaticamente | Warning |
| `match` sem wildcard `_` (best-effort) | Warning de exaustividade |
| Error nodes herdados do Parser | Diagnósticos propagados |

---

## Modelo de Escopos

A Sema mantém uma **pilha de escopos** durante o walk da AST. Cada escopo é um mapa `name → SymbolInfo`. Ao entrar em um bloco (`{}`), um novo escopo é empurrado na pilha; ao sair, é removido.

### Resolução de nomes (shadowing)

A resolução de um `IDENT` percorre a pilha de baixo para cima — o escopo mais interno vence. Uma nova atribuição no escopo atual **cria uma nova variável** (shadow); ela não modifica a variável de mesmo nome em escopo pai.

### `outer`

`outer` é uma atribuição explícita ao escopo pai imediato. A Sema sobe exatamente um nível na pilha e busca o nome lá. Se o nome não existir no escopo pai imediato, é erro de compilação.

```bela
total = 0
for (i in 1..10) {
    outer total = total + i   // modifica total do escopo pai
}
// total é 55
```

Sem `outer`, a atribuição `total = total + i` dentro do `for` criaria uma nova variável `total` local ao bloco, sem afetar o `total` externo.

---

## Tipos Inferidos

O campo `node_types` mapeia cada `Node.Index` para um `TypeInfo` que representa o tipo inferido pela Sema para aquele nó.

A inferência é **best-effort**: Bela é dinamicamente tipada, e a Sema não tem garantia de conhecer o tipo concreto de toda expressão em tempo de compilação. Quando o tipo não pode ser determinado estaticamente, `TypeInfo` assume o valor `dynamic`.

> **TBD:** Representação interna de `TypeInfo` — estrutura de dados, encoding de tipos compostos (`array<T>`, `result<T,E>`, `Future<T>`) e sentinel para `dynamic`. Reservado para o spec de implementação.

### Anotações de tipo

| Forma | Efeito na Sema |
|---|---|
| `a = 10` | Tipo inferido como `int` (literal) |
| `a: int = 10` | Anotação ignorada para enforcement — registrada como hint no `symbol_table` |
| `a: fixed<int> = 10` | Sema verifica violações estáticas — emite warning ou erro conforme certeza |

`: type` sem `fixed` é apenas documentação: a Sema registra a anotação no `symbol_table` mas não a usa para enforcement de tipo. Apenas `fixed<T>` dispara verificação semântica.

> **TBD:** Profundidade da inferência de tipos para inlay hints do LSP — até que nível a Sema tenta propagar tipos concretos em expressões compostas. Será definido antes da implementação.
