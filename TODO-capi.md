# TODO — C API e Sandboxing

Decisões de design em aberto para a C API de embedding e o sistema de sandboxing de Bela.
Cada item é uma pergunta ou escolha que precisa ser resolvida antes de documentar formalmente.

---

## C API — Tipos de Retorno e Consistência

- [x] **Consistência de retorno em `bela_vm_load_*`** — as funções de carregamento retornam `int`, mas `BelaStatus` já existe. **Decisão: retornar `BelaStatus` para consistência. `load` e `call` usam o mesmo enum.**

- [ ] **`bela_vm_call` não tem parâmetro de saída** — a assinatura atual é `int bela_vm_call(vm, fn_name, argc, argv)`. Não há como obter o valor de retorno da função Bela. Opções:
  - (a) `BelaStatus bela_vm_call(vm, fn_name, argc, argv, BelaValue *out)`
  - (b) `BelaValue bela_vm_call_v(vm, fn_name, argc, argv)` (variante que retorna valor)
  - (c) `bela_vm_last_result(vm)` para ler o último retorno após a chamada

- [ ] **`bela_vm_eval` e erros** — se a expressão avaliada causa runtime error, o que `bela_vm_eval` retorna? `null` é um valor Bela válido, não pode ser sentinela. Precisa de `bela_vm_eval_safe(vm, expr, BelaValue *out)` retornando `BelaStatus`, ou introspection pós-chamada.

- [x] **`bela_vm_load_string` sem nome** — erros em código carregado via string mostram `??:42`. **Decisão: adicionar `bela_vm_load_string_named(vm, source, name)` para mensagens de erro úteis (ex: `"<repl>:5"`).**

---

## C API — Valores e Interoperabilidade

- [ ] **`BelaValue` para tipos complexos** — o spec define construtores apenas para primitivos (int, float, string, bool, null). Não há como criar arrays, objetos ou `result<T,E>` do lado C. Necessário para interop real:
  - `bela_value_array(vm)` + `bela_array_push(arr, v)`
  - `bela_value_result_ok(v)` / `bela_value_result_error(msg)`
  - `bela_value_object(vm)` + `bela_object_set(obj, key, v)`

- [ ] **`bela_value_retain` / `bela_value_release` — RC interop (CRÍTICO)** — quando o host recebe um `BelaValue` que é um objeto Bela (array, instância de classe), o RC não sabe que o host está segurando uma referência. Se o script perder todas as suas referências, o objeto é coletado e o host fica com ponteiro inválido. Precisa de:
  - `void bela_value_retain(BelaVM *vm, BelaValue v)` — incrementa RC
  - `void bela_value_release(BelaVM *vm, BelaValue v)` — decrementa RC
  Sem isso, qualquer objeto Bela retornado ao C é uma bomba-relógio.

- [ ] **Introspection de tipo** — o spec tem `bela_is_null(v)` mas só `bela_as_*` para os demais. Falta:
  - `BelaType bela_typeof(BelaValue v)` retornando enum (`BELA_TYPE_INT`, `BELA_TYPE_STRING`, `BELA_TYPE_OBJECT`, etc.)
  - Ou `bela_is_int(v)`, `bela_is_string(v)`, `bela_is_array(v)` etc.

- [ ] **Acesso a campos de objetos Bela do lado C** — se uma função Bela retorna um objeto, o host não consegue ler campos. Falta:
  - `BelaValue bela_object_get(BelaVM *vm, BelaValue obj, const char *field)`
  - `void bela_object_set(BelaVM *vm, BelaValue obj, const char *field, BelaValue v)`

- [x] **Ownership de strings em `bela_value_string`** — quando o host cria `bela_value_string(vm, s)`, Bela copia a string ou guarda ponteiro? **Decisão: Bela sempre copia. Documentar explicitamente.**

- [x] **`bela_vm_stacktrace` — ownership da string retornada** — retorna `const char *`. **Decisão: VM owna, válido até próxima chamada API. Documentar explicitamente.**

---

## C API — Funções Registradas pelo Host

- [ ] **Reportar erros de `BelaCFunction`** — como uma função C registrada indica erro para o script Bela? Sem mecanismo explícito. Falta:
  - `void bela_throw_error(BelaVM *vm, const char *msg)` — script vê `result.ERROR`
  - `void bela_panic(BelaVM *vm, const char *msg)` — equivalente a chamar `panic()`
  A função C chama um desses e retorna `bela_value_null()`.

- [ ] **Type safety de argumentos em `BelaCFunction`** — a função C recebe `BelaValue *argv` sem garantia de tipos. Se o script passa string onde int é esperado, `bela_as_int()` retorna lixo. Opções:
  - (a) C function valida manualmente e chama `bela_throw_error` se inválido
  - (b) Sistema de declaração de assinatura da função C que o VM valida antes de chamar
  Para v1, (a) é suficiente. Documentar como responsabilidade do host.

- [ ] **Funções C variádicas e `fixed<T>`** — funções registradas via C são opacas para o type checker de Bela. `fixed<T>` não consegue verificar chamadas a funções C. Aceitar como limitação? Ou permitir que o host declare assinatura via `bela_register_fn_typed(vm, name, signature, fn)`?

- [ ] **Chamada de métodos em objetos Bela do lado C** — `bela_vm_call()` só chama funções top-level. Para chamar `enemy.takeDamage(10)` do C, precisa de `bela_vm_call_method(vm, obj, method_name, argc, argv, out)`.

---

## C API — Configuração e Ciclo de Vida

- [ ] **Campos faltando em `BelaConfig`** — a struct atual não cobre:
  - `const char *debug_host` — default `"127.0.0.1"` (nunca `"0.0.0.0"` por segurança)
  - `const char *module_search_path` — onde procurar módulos além de stdlib
  - `bool strict_mode` — força `fixed<T>` em todo código, sem `dynamic`

- [x] **Unidade de `max_stack`** — o spec não especifica. **Decisão: call frames (mais intuitivo para scripts).**

- [ ] **Múltiplos `bela_vm_load_*` no mesmo VM** — é possível? Os namespaces se mesclam? Ex: carregar uma "base library" e depois vários scripts de usuário que compartilham o mesmo global scope. Definir semântica de múltiplos loads.

- [x] **Estado do VM após `BELA_PANIC`** — **Decisão: VM entra em estado `DEAD` após PANIC, deve ser freed e recriado.** Para hosts long-running, recriar a VM é o caminho seguro.

- [ ] **`async fn` chamada do C** — se `bela_vm_call` chama uma função `async`, o retorno é `Future<T>`. O host precisa chamar `bela_vm_tick(vm)` em loop até o future resolver. Definir API para: (a) detectar se retorno é Future, (b) aguardar resolução, (c) obter valor final.

---

## C API — Callbacks

- [x] **`BelaLogHandler` — assinatura canônica** — `log.md` define `typedef void (*BelaLogHandler)(int level, const char *msg, const char *context)` com `context` para key=value estruturado. **Decisão: usar a assinatura de log.md (com context) — perda de informação estruturada seria um downgrade.**

- [ ] **Callbacks faltando** — o spec tem log handler e panic handler. Faltam:
  - `bela_set_oom_handler(vm, handler)` — notificação antes de OOM terminar a VM
  - `bela_set_timeout_handler(vm, handler)` — notificação quando instruction limit é atingido
  - `bela_set_module_load_handler(vm, handler)` — interceptar imports, permitir ao host fornecer módulos customizados ou bloquear módulos específicos em runtime

- [x] **`BelaPanicHandler` — pode cancelar o panic?** — **Decisão: não. Panic é irrecuperável, handler é só notificação (`void`). Host pode apenas logar e recriar a VM.**

---

## Sandboxing — Estado Padrão e API

- [x] **Estado padrão após `bela_vm_new()`** — **Decisão: módulos de "computação pura" são sempre disponíveis; apenas módulos com I/O precisam de allow explícito.**

- [x] **Enumerar módulos "sempre disponíveis" (Nível 0)** — **Decisão:**
  - Sempre disponível: `math`, `string`, `collections` (array, map, set), `json`, `encoding`, `regex`, `random`, `log`
  - Requer allow: `io`, `os.*`, `net.*`, `concurrency`
  - `time.now()` disponível por padrão; `time.sleep()` requer allow

- [x] **API de nível de sandbox** — **Decisão: implementar enum de conveniência:**
  ```c
  typedef enum {
      BELA_SANDBOX_PURE,        // apenas computação (default embedding)
      BELA_SANDBOX_CONTROLLED,  // + console, os.env (read-only), os.info
      BELA_SANDBOX_FILESYSTEM,  // + io.file, os.fs (com root confinado)
      BELA_SANDBOX_NETWORK,     // + net.http.client (com domain whitelist)
      BELA_SANDBOX_FULL         // tudo (modo standalone)
  } BelaSandboxLevel;

  void bela_sandbox_set_level(BelaVM *vm, BelaSandboxLevel level);
  ```

- [x] **Precedência allow vs. deny** — **Decisão: última chamada vence (mais previsível para quem configura manualmente).** Ex: `allow_all()` + `deny_module("os.process")` = tudo menos os.process.

---

## Sandboxing — Controle de Recursos e Paths

- [ ] **Confinamento de diretório — path traversal** — `bela_vm_sandbox_set_root(vm, "/var/game/")` deve rejeitar `Path("../../etc/passwd")`. A VM precisa canonicalizar caminhos e rejeitar escapes. Atenção especial em Windows: UNC paths (`\\server\share`), drive letters (`C:\`), e junction points. Definir: canonicalização acontece na VM ou no OS?

- [x] **Múltiplos roots de filesystem** — **Decisão v1: root único via `bela_sandbox_set_root(vm, path)`.** Múltiplos paths (`bela_sandbox_add_allowed_path`) ficam para versão futura.

- [x] **Domain whitelist para `net`** — **Decisão: `bela_sandbox_allow_hosts()` é parte do sandboxing API e funciona como restrição adicional — mesmo com net habilitado, só acessa hosts na whitelist.**

- [x] **Unificação de limites de recursos** — **Decisão: limites só em `BelaConfig` (ao criar VM). Remover as funções duplicadas de sandboxing.**

- [x] **Lock de sandbox após load** — **Decisão: sandbox locked após o primeiro `bela_vm_load_*`. Tentativa de modificar após lock retorna `BELA_SANDBOX_LOCKED`.** Previne bugs de segurança por ordem de configuração.

---

## Sandboxing — Segurança e Auditoria

- [ ] **Callback de violação de sandbox** — quando um script tenta acessar um módulo não permitido, o host recebe notificação? Útil para auditoria de segurança e logging. Falta: `bela_set_sandbox_violation_handler(vm, handler)` que recebe `(vm, module_name, script_location)`.

- [x] **Funções C registradas escapam o sandbox — documentar** — **Decisão: host-registered functions são sempre "confiáveis" e bypass sandbox. Documentar claramente que o sandbox controla apenas stdlib, não código nativo do host.**

- [x] **Módulos host-defined e sandboxing** — **Decisão: módulos host-defined são sempre disponíveis. O host é soberano. Sandbox controla apenas stdlib.**

---

## Cross-cutting: C API + Sandboxing

- [ ] **`bela_sandbox_set_root` está fora de `BelaConfig`** — inconsistência: alguns settings de sandbox são em `BelaConfig` (max_memory, max_instructions) e outros são chamadas separadas (`bela_sandbox_set_root`, `bela_sandbox_allow_hosts`). Unificar: tudo em `BelaConfig` (imutável, define comportamento da VM), ou tudo em chamadas dinâmicas (flexível mas perigoso post-load)?
