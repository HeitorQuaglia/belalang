# Toolchain — MCP Server

`bela-mcp.a` é uma biblioteca separada que o host linka explicitamente para expor as capacidades do debugger via Model Context Protocol (MCP). Disponível apenas em conjunto com `bela-dev.a` — MCP depende do DAP estar ativo. Exclusivamente para desenvolvimento; nunca expor em produção.

---

## Dependência e Ciclo de Vida

| Build | Bibliotecas | MCP disponível |
|---|---|---|
| Produção | `bela-vm.a` | Não |
| Desenvolvimento (DAP) | `bela-dev.a` | Não |
| Desenvolvimento (DAP + MCP) | `bela-dev.a` + `bela-mcp.a` | Sim |

> **Nota:** o host linka as bibliotecas adequadas — sem `#ifdef` no código do host.

---

## C API

```c
// Inicia o servidor MCP na porta especificada.
// Requer que o DAP já esteja ativo (bela-dev.a).
// Retorna BELA_ERROR se o DAP não estiver ativo, se a porta estiver em uso,
// ou se for detectada tentativa de bind fora de localhost.
BelaStatus bela_mcp_start(BelaVM* vm, uint16_t port);

// Para o servidor MCP e libera recursos.
void bela_mcp_stop(BelaVM* vm);
```

Exemplo de uso típico:

```c
BelaVM* vm = bela_vm_new(&config);
bela_vm_load_file(vm, "app.bela");

// TBD: assinaturas exatas de bela_dap_start/stop — C API do bela-dev.a ainda não especificada
bela_dap_start(vm, 4711);   // bela-dev.a
bela_mcp_start(vm, 4712);   // bela-mcp.a

bela_vm_call(vm, "main", 0, NULL);

bela_mcp_stop(vm);
bela_dap_stop(vm);
bela_vm_free(vm);
```

Para o ciclo de vida completo da VM, ver [embedding.md](../capi/embedding.md).

> **TBD:** Assinaturas exatas de `bela_dap_start` e `bela_dap_stop` — a C API do `bela-dev.a` ainda não está especificada. Será definido antes da implementação.

---

## Protocolo

- MCP via TCP em `127.0.0.1:<porta>`
- Porta configurável
- Sem autenticação — localhost é o único controle de acesso

> **⚠ Atenção:** Somente para desenvolvimento. Nunca expor em produção. Nunca fazer bind em `0.0.0.0`.

---

## Tools

Tools são ações que o agente de IA invoca; mapeiam sobre capabilities do DAP (ver [debugger.md](debugger.md)).

### Controle de Execução

| Tool | Equivalente DAP | Descrição |
|---|---|---|
| `continue` | `continue` | Retoma execução até o próximo breakpoint |
| `pause` | `pause` | Pausa a execução |
| `step_over` | `next` | Executa a linha atual e para na próxima |
| `step_into` | `stepIn` | Entra na função chamada na linha atual |
| `step_out` | `stepOut` | Sai da função atual |

### Breakpoints

| Tool | Equivalente DAP | Descrição |
|---|---|---|
| `set_breakpoint` | `setBreakpoints` | Define breakpoint em arquivo:linha |
| `remove_breakpoint` | `setBreakpoints` (sem o alvo) | Remove breakpoint existente |
| `set_conditional_breakpoint` | `setBreakpoints` com `condition` | Breakpoint com expressão condicional |

### Inspeção

| Tool | Equivalente DAP | Descrição |
|---|---|---|
| `get_stack_trace` | `stackTrace` | Stack frames atuais com arquivo, linha, coluna |
| `get_variables` | `variables` | Variáveis em escopo de um frame |
| `evaluate` | `evaluate` | Avalia expressão no contexto do frame atual |
| `set_variable` | `setVariable` | Modifica o valor de uma variável em tempo de execução |
| `inspect_rc` | `bela/inspect` (custom) | Inspeção de reference counting de um valor |
| `get_async_state` | `bela/asyncState` (custom) | Snapshot do estado do event loop |

Tools de inspeção funcionam apenas com VM pausada em execução normal; invocá-las durante execução retorna erro. Após panic, o snapshot é somente-leitura — `set_variable` e `evaluate` também retornam erro pois a VM está `DEAD`. `set_variable` respeita a semântica de tipagem dinâmica do Bela: qualquer valor pode ser atribuído; a semântica RC é gerenciada pelo debug core, equivalente a uma atribuição normal no runtime. O snapshot de panic não é uma tool — fica disponível via resource `bela://panic` após o evento `stopped` com `reason: "bela/panic"` (ver seção Resources).

---

## Resources

Resources são views somente-leitura acessíveis via URI sem invocar uma tool.

| Resource URI | Disponível quando | Conteúdo |
|---|---|---|
| `bela://stack` | VM pausada | Stack frames atuais com arquivo, linha, coluna |
| `bela://variables/{frameId}` | VM pausada | Variáveis em escopo de um frame específico |
| `bela://source/{file}` | Sempre | Conteúdo do arquivo-fonte Bela |
| `bela://panic` | Após evento `stopped` com `reason: "bela/panic"` | Snapshot de panic capturado antes do estado DEAD |
| `bela://async` | VM pausada (execução normal) | Estado do event loop (equivalente a `bela/asyncState`) |

`bela://panic` retorna 404 se não houve panic. `bela://stack` e `bela://variables/{frameId}` retornam 404 se a VM não está pausada. `bela://async` não está disponível pós-panic — o snapshot de panic não inclui estado do event loop.

---

## Segurança

- Bind exclusivo em `127.0.0.1` — sem binding em `0.0.0.0` ou interfaces externas
- Sem autenticação — localhost é o único controle de acesso
- `bela_mcp_start` retorna `BELA_ERROR` se o DAP não estiver ativo, se a porta estiver em uso ou se detectar tentativa de bind fora de localhost
- Não presente em `bela-vm.a` — só em `bela-mcp.a`, que o host linka explicitamente

> **⚠ Atenção:** `bela-mcp.a` não deve ser linkada em builds de produção. Expor o MCP server em redes não-locais representa acesso irrestrito à VM em execução.
