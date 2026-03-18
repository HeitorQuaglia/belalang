# TODO — Bytecode, Debugger, Hot Reload, LSP

Decisões de design em aberto para o toolchain de Bela.
Cada item é uma pergunta ou escolha que precisa ser resolvida antes de implementar.

---

## Bytecode

- [x] **Versioning strategy** — **Decisão: strict rejection. VM v0.2 recusa bytecode de v0.1, mensagem "please recompile".** Host controla versão da VM embarcada, não há legado real.

- [x] **Source maps: embedded vs separado** — **Decisão: arquivo `.belmap` separado, distribuído independentemente do `.belc`.** Convenção: `.bela` → `.belc` + `.belmap` opcional.

- [x] **Compilation unit** — **Decisão v1: bundle único (tudo em um `.belc`).** Simples para distribuição. Compilação por módulo + linker fica para versão futura.

- [ ] **Bytecode encryption** — suportar chave de decriptação fornecida pelo host via C API para proteção adicional de IP? Fora do escopo v1, mas precisa ser planejado no formato do header para não quebrar futuramente.

- [ ] **Constant pool** — strings devem ser internadas? O pool é compartilhado entre módulos num bundle ou por módulo?

---

## Debugger

- [x] **Segurança do debug port** — **Decisão v1: localhost-only, sem auth (desenvolvimento apenas). Documentar explicitamente que nunca deve ser usado em produção.**

- [ ] **RC inspection via custom DAP** — DAP não tem request nativa para reference counts. Implementar como `customRequest` (`bela/inspect`). Definir schema da resposta: address, RC, campos, flag de possível ciclo.

- [ ] **Evaluate at breakpoint — sub-evaluator sandboxado** — se a expressão avaliada causa panic, o que acontece? Precisa de modo de avaliação que captura errors/panics sem encerrar a sessão de debug.

- [ ] **Event loop pausado no breakpoint** — quando parado num breakpoint, timers param, promises não resolvem, conexões podem dar timeout. Comportamento correto, mas precisa ser documentado para não surpreender devs debugando código assíncrono.

- [ ] **Breakpoint condicional** — requer evaluator acessível durante pause. Mesma infra do evaluate at breakpoint. Definir se condição é avaliada com acesso ao escopo local completo.

---

## Hot Reload

- [x] **Escopo do reload** — **Decisão v1: function-body-only.** Aceita reload se assinaturas e campos não mudaram. Se mudou, retorna `BELA_RELOAD_SCHEMA_CHANGE`. Cobre 90% do uso (tweaking lógica). Method propagation e full migration ficam para versões futuras.

- [ ] **Re-execução de top-level code** — hot reload NÃO deve re-executar side effects do init (`db = Database.connect(...)`). Deve substituir definições de função e constantes literais sem re-executar código com efeitos. Como distinguir? Implícito (expressão computada vs. definição pura) ou explícito?

- [ ] **`BelaReloadStatus` enum** — definir valores de retorno de `bela_vm_reload()`:
  ```c
  BELA_RELOAD_OK
  BELA_RELOAD_COMPILE_ERROR   // sintaxe/tipo inválido
  BELA_RELOAD_SCHEMA_CHANGE   // campos/assinaturas mudaram
  BELA_RELOAD_NOT_FOUND       // arquivo não encontrado
  ```

- [x] **File watching** — **Decisão: responsabilidade do host (inotify/FSEvents/ReadDirectoryChanges). VM não faz watching.** Host chama `bela_vm_reload()` quando detecta mudança.

- [x] **Hot reload + bytecode** — **Decisão: mutuamente exclusivos em produção por design.** Produção usa `.belc` sem source, desenvolvimento usa `.bela`. Documentar explicitamente.

---

## LSP

- [ ] **Parser tolerante a erros** — compilador de produção pode abortar no primeiro erro. LSP precisa parsear código incompleto e continuar análise depois de erros. Opções: (a) modo "lenient" no mesmo parser, (b) parser separado para LSP. Qual abordagem?

- [ ] **Completions para `dynamic`** — código sem type hints não tem tipo estático. LSP pode: (a) oferecer completions de membros do módulo importado (sempre possível), (b) inferir tipo por contexto/uso (best-effort, como TypeScript). Definir expectativa de profundidade de inferência.

- [ ] **`fixed<T>` violations como diagnóstico principal** — assignment de tipo incompatível para variável `fixed<T>` deve aparecer sublinhado em vermelho imediatamente. É o maior value add do LSP para Bela. Confirmar como feature prioritária.

- [x] **Formatting opinionated** — **Decisão: sem configuração (gofmt style). Elimina style wars, código Bela sempre igual.**

- [ ] **Module resolution** — LSP precisa resolver `from io use X` (stdlib, path fixo), `from http-router use Y` (dependência externa, depende de `bela.toml`) e `from utils use Z` (módulo local). LSP e package manager precisam ser co-designed. Definir protocolo de resolução.

- [ ] **Inlay hints para tipos inferidos** — mostrar tipo inferido de expressões `dynamic` onde o LSP consegue determinar. Incentiva anotações de tipo onde importam. Confirmar como feature de alta prioridade.

---

## Cross-cutting

- [ ] **Split produção vs. desenvolvimento** — código de debug/LSP/hot reload deve ser excluído de builds de produção. Mecanismo: `#ifdef BELA_DEBUG` na VM? Binários separados (`bela-vm.a` vs `bela-dev.a`)? Definir antes de implementar.

- [x] **Sequenciamento de implementação** — **Decisão:**
  1. **Bytecode (v1 obrigatório)** — sem bytecode, IP não está protegido
  2. **Hot Reload function-body-only (v1)** — diferencial para game dev, escopo limitado
  3. **LSP: Diagnostics + Completion (v1 básico)** — DX mínimo aceitável
  4. **Debugger (v2)** — mais complexo, depende de bytecode + source maps estabilizados

- [ ] **Source maps compartilhados entre Bytecode e Debugger** — formato `.belmap` precisa ser definido uma vez e consumido pelos dois. Não duplicar esquema.
