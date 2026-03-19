# Compiler — Compilation Database (CDB)

A Compilation Database (CDB) é o componente central da `libbela-frontend`. Todos os consumidores — compilador CLI, REPL e LSP — são clientes desta biblioteca através da CDB; nenhum deles contém lógica de compilação própria. O protocolo de comunicação entre o LSP e a `libbela-frontend` (JSON-RPC sobre stdio vs. socket) está fora do escopo deste documento.

---

## Arquitetura

```
┌──────────────────────────────────────────────┐
│               libbela-frontend               │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │       Compilation Database (CDB)        │ │
│  │  - Mapa: SourceId → SourceSnapshot      │ │
│  │  - Mapa: SourceId → TokenArray          │ │
│  │  - Mapa: SourceId → Ast                 │ │
│  │  - Mapa: SourceId → SemaResult          │ │
│  │  - Mapa: SourceId → Bytecode            │ │
│  │  Invalidação por hash do source         │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  Lexer   Parser   Sema   Codegen             │
└──────────────────────────────────────────────┘
       ↑               ↑              ↑
   bela CLI          REPL           LSP
```

---

## SourceId

`SourceId` é o identificador único de uma unidade de código na CDB. É derivado do caminho canônico do arquivo (para arquivos em disco) ou de um hash do conteúdo (para strings em memória, como entradas do REPL).

| Origem | Geração do SourceId |
|--------|---------------------|
| Arquivo em disco | A definir — hash de conteúdo ou caminho canônico (ver TBD) |
| String em memória (REPL, LSP hover) | Hash do conteúdo da string |

Todas as queries da CDB recebem um `SourceId` como chave. A CDB usa esse identificador para localizar o resultado em cache e detectar mudanças de conteúdo.

> **TBD:** Estratégia definitiva de geração do `SourceId` — hash do conteúdo vs. caminho canônico normalizado, e tratamento de colisões. Será definido antes da implementação.

---

## Pipeline de Queries

A CDB expõe cinco queries. Cada query verifica se o hash do source mudou desde a última computação: se não mudou, retorna o resultado em cache; se mudou, executa as fases necessárias, armazena e retorna.

| Query | Dependência | Retorno | Descrição |
|-------|-------------|---------|-----------|
| `getSnapshot` | — | `SourceSnapshot` (bytes crus) | Lê e armazena o conteúdo do source. Ponto de entrada de toda a pipeline. |
| `getTokenArray` | `getSnapshot` | `TokenArray` | Executa o [Lexer](lexer.md) sobre o snapshot e armazena o array de tokens. |
| `getAst` | `getTokenArray` | `Ast` (com error nodes) | Executa o [Parser](parser.md) e retorna a AST. Erros de sintaxe são representados como error nodes — a parse nunca aborta. |
| `getSemaResult` | `getAst` | `SemaResult` | Executa a análise semântica ([sema.md](sema.md)): resolução de nomes, inferência de tipos (best-effort), diagnósticos. |
| `getBytecode` | `getSemaResult` | bytecode (`.belc`) | Executa o [Codegen](codegen.md) e retorna o bytecode. Só executado se `SemaResult` não contiver erros; ver [bytecode.md](../toolchain/bytecode.md). |

---

## Invalidação

A CDB usa o **hash do conteúdo** do source como mecanismo de invalidação. A cada query, a CDB compara o hash atual do source com o hash armazenado no momento da última computação:

- **Hash idêntico** → resultado em cache é válido, retorna sem recomputar.
- **Hash diferente** → a arena do `SourceId` correspondente é descartada integralmente e recriada; todas as fases são reexecutadas a partir do `getSnapshot`.

Não há invalidação granular de fases individuais ou nós de AST — quando o source muda, toda a cadeia para aquele `SourceId` é descartada. Esse modelo simplifica a implementação sem custo prático: as fases são rápidas e a invalidação total por arquivo é aceitável.

> **TBD:** Serialização da CDB em disco — se o cache de resultados (TokenArray, Ast, SemaResult) persiste entre invocações do compilador para reuso incremental. Será definido antes da implementação.

---

## Modelo de Arena

A CDB usa arena allocators para gerenciar a memória das estruturas produzidas pela pipeline.

### Arquivos normais

Cada `SourceId` correspondente a um arquivo em disco tem **uma arena própria**. Quando o hash do source muda (arquivo editado), a arena é descartada integralmente — sem liberação individual de nós — e recriada na próxima query.

### REPL

O REPL usa uma **arena de sessão única**, compartilhada por todos os fragmentos da sessão. Entradas são append-only: cada fragmento aceito é acrescentado à arena, e a arena nunca é invalidada durante a sessão.

A Sema, ao analisar um fragmento novo, recebe o `SemaResult.symbol_table` acumulado das entradas anteriores como escopo externo — sem re-executar as fases sobre fragmentos já aceitos. Fragmentos rejeitados (com erros) são descartados sem afetar a arena ou o estado acumulado.

> **Limitação conhecida (v1):** a arena do REPL cresce sem limite durante a sessão — não há mecanismo de eviction de fragmentos individuais. Qualquer estratégia futura de liberação de memória por fragmento exigiria revisão do modelo de arena única.
