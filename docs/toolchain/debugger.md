# Toolchain — Debugger

O debugger implementa o Debug Adapter Protocol (DAP) via TCP. É v2 — depende de bytecode e `.belmap` estabilizados (ver [source-maps.md](source-maps.md)). Disponível apenas em `bela-dev.a`.

---

## Protocolo

- DAP via TCP em `127.0.0.1:<porta>`
- Sem autenticação
- ⚠ Somente para desenvolvimento. Nunca expor em produção. Nunca bind em `0.0.0.0`.

> **TBD:** Porta padrão do DAP server e se é configurável via `BelaConfig` ou argumento de CLI. Será definido antes da implementação.

---

## Split Produção vs. Desenvolvimento

| Biblioteca | Uso | Inclui DAP server |
|---|---|---|
| `bela-vm.a` | Produção | Não — zero overhead |
| `bela-dev.a` | Desenvolvimento | Sim |

> **Nota:** o host linka a biblioteca adequada para cada contexto — sem `#ifdef` no código do host.

---

## Custom Requests

DAP padrão cobre breakpoints, stepping, `stackTrace`, `scopes`, `variables` e `evaluate`. Os `customRequests` cobrem o que é Bela-específico.

### `bela/inspect` — RC Inspection

Inspeciona o reference count de um objeto em pausa. Útil para diagnosticar memory leaks e ciclos RC.

Request:
```json
{ "variableReference": 42 }
```

Response:
```json
{
  "address": "0x7f3c1a",
  "rc": 3,
  "fields": [
    { "name": "nome", "type": "string", "variableReference": 43 }
  ],
  "possible_cycle": false
}
```

> **TBD:** Comportamento de `bela/inspect` quando `possible_cycle: true` — visualização do grafo de referências. Será definido antes da implementação.

### `bela/asyncState` — Event Loop Snapshot

Retorna o estado do event loop no momento da pausa. Útil porque o event loop congela completamente durante breakpoints — timers param, futures não resolvem.

Response:
```json
{
  "pending_timers": [{ "id": 1, "remaining_ms": 150 }],
  "pending_futures": 3,
  "microtask_queue_length": 0
}
```

---

## Panic Stop

Quando ocorre um `BELA_PANIC`, o DAP server captura um **snapshot** dos frames de stack e variáveis locais antes que a VM entre em estado `DEAD`. A inspeção posterior acontece sobre esse snapshot — não sobre a VM morta.

Após o snapshot, a VM segue o ciclo normal de `BELA_PANIC` descrito em `[../capi/embedding.md](../capi/embedding.md)`: apenas `bela_vm_free` é chamada sobre ela. O usuário inspeciona o snapshot; ao encerrar a sessão, a VM é liberada.

Sequência:

1. Panic dispara → DAP server captura snapshot (frames, variáveis locais, mensagem do panic)
2. DAP emite `stopped` com `reason: "bela/panic"` + mensagem do panic
3. Usuário navega no snapshot: `stackTrace`, `scopes`, `variables` respondem do snapshot (read-only)
4. `evaluate`, `continue`, `next`, `stepIn`, `stepOut` retornam erro — VM está DEAD
5. `terminate` ou `disconnect` → `bela_vm_free` é chamada; sessão encerrada

Não há `customRequest` para panic — usa o evento `stopped` padrão do DAP com uma `reason` customizada.

---

## Evaluate at Breakpoint

Usa o request `evaluate` padrão do DAP. A implementação reutiliza o pipeline do REPL (ver [../compiler/codegen.md](../compiler/codegen.md)):

1. O fragmento de expressão é processado por Lexer → Parser → Sema → Codegen
2. O escopo local do frame atual está disponível como escopo externo para a Sema — o mesmo mecanismo que o REPL usa para o `symbol_table` acumulado
3. O resultado é retornado como `Variable` padrão do DAP, com tipo e valor
4. Erros de compilação ou runtime retornam `result` com a descrição do erro — a sessão de debug não é encerrada

### Breakpoints Condicionais

Usam o campo `condition` nativo do DAP em `setBreakpoints`. A condição é uma expressão Bela avaliada pelo mesmo sub-evaluator do evaluate at breakpoint — sem mecanismo separado.

| Condição | Comportamento |
|---|---|
| Condição verdadeira | Pausa normalmente |
| Condição falsa | Continua execução sem parar |
| Condição com erro de compilação | Pausa + diagnóstico no output |
| Condição com erro de runtime | Pausa + diagnóstico no output |

---

## Event Loop no Breakpoint

Quando a VM pausa em um breakpoint, o event loop congela completamente.

| Componente | Comportamento durante pausa |
|---|---|
| Timers (`setTimeout`, `setInterval`) | Congelados — o tempo não avança |
| Futures / `await` | Não resolvem — callbacks não são invocados |
| Conexões de rede | Podem dar timeout dependendo do tempo de pausa |
| Microtask queue | Congelada — não drena |

O estado do event loop fica visível via `bela/asyncState`. Ao retomar, timers não compensam o tempo pausado — intencional para evitar disparos em cascata ao retomar uma sessão longa de debug.
