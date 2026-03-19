# Compiler — Geração de Código (Codegen)

O Codegen é a quarta e última fase do frontend de compilação. Ele consome a `Ast` produzida pelo [Parser](parser.md) e o `SemaResult` produzido pela [Sema](sema.md) e emite bytecode `.belc`. É a **única fase de `libbela-frontend` que conhece a VM** — Lexer, Parser e Sema são completamente agnósticos ao backend.

---

## Pré-condições

O Codegen só executa se `SemaResult.diagnostics` não contiver nenhum diagnóstico com `severity == .err`. Warnings são tolerados. Para a definição exata de erro vs. warning, ver [sema.md](sema.md).

| Condição | Resultado |
|---|---|
| Sem erros em `diagnostics` | Codegen executa |
| Com erros em `diagnostics` | Codegen não executa — pipeline aborta |
| Com warnings em `diagnostics` | Codegen executa — warnings são tolerados |

---

## Input e Output

| | Descrição |
|---|---|
| Input | `Ast` + `SemaResult` (ver [parser.md](parser.md) e [sema.md](sema.md)) |
| Output | Bytecode `.belc` — formato em [`bytecode.md`](../toolchain/bytecode.md) |

O formato do arquivo `.belc` — header, versionamento, bundle único v1, source map `.belmap` — é documentado exclusivamente em [`bytecode.md`](../toolchain/bytecode.md). Este documento não duplica essas informações.

---

## Isolamento do Backend

Lexer, Parser e Sema produzem estruturas de dados puras sem qualquer referência a construtos da VM. Esse isolamento é uma propriedade arquitetural central da `libbela-frontend` e viabiliza dois casos de uso importantes:

| Consumidor | Como usa a pipeline |
|---|---|
| LSP | Executa até a Sema (`getSemaResult`) para diagnósticos, completions e go-to-definition — sem instanciar nada relacionado à VM |
| REPL | Compartilha o mesmo frontend com o compilador CLI, incluindo o Codegen, sem código de compilação próprio |
| Compilador CLI | Executa a pipeline completa até o Codegen (`getBytecode`) |

A separação também significa que qualquer futura mudança na arquitetura da VM não afeta as fases de Lexer, Parser ou Sema.

---

## REPL como Cliente Especial

O REPL é um consumidor especial do Codegen. Ao contrário do compilador CLI, que processa arquivos completos, o REPL opera sobre **fragmentos incrementais** e mantém dois estados persistentes entre entradas do usuário:

1. **Contexto acumulado** — o conjunto de declarações top-level aceitas até agora na sessão
2. **Instância persistente da VM** — o estado de execução atual

Para cada entrada do usuário, o REPL executa o seguinte ciclo:

| Passo | Ação |
|---|---|
| 1 | Injeta o input como novo fragmento no contexto acumulado |
| 2 | Executa a pipeline completa (Lexer → Parser → Sema → Codegen) apenas sobre o fragmento |
| 3 (sem erros) | Emite bytecode incremental e o executa na VM persistente |
| 3 (com erros) | Descarta o fragmento e exibe os diagnósticos — o estado da VM não é afetado |

### Resolução de nomes no REPL

Ao analisar um fragmento novo, a Sema recebe o `SemaResult.symbol_table` acumulado de todas as entradas anteriores aceitas como escopo externo. As fases anteriores **não são re-executadas** sobre fragmentos já aceitos — apenas o novo fragmento percorre a pipeline.

Isso resolve referências a declarações de entradas anteriores sem recompilar o histórico da sessão.

### Ciclo de vida dos fragmentos

Fragmentos aceitos são append-only na CDB — nunca são invalidados durante a sessão. O REPL usa uma arena de sessão única, compartilhada por todos os fragmentos, em lugar de arenas individuais por fragmento. Fragmentos rejeitados (com erros) são descartados sem afetar a arena ou o estado acumulado.

> **TBD:** Comportamento de fragmentos contendo `await` no top-level — fora do escopo da v1. Depende da integração do event loop da VM com o REPL. Será definido antes da implementação.
