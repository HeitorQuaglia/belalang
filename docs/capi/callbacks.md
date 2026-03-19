# C API — Callbacks e Handlers

Handlers são opcionais — sem registro, o comportamento é o fallback padrão. Nenhum handler cancela a operação que o disparou: são apenas notificações. Todos os handlers devem ser registrados antes do primeiro `bela_vm_load_*` para garantir que estejam ativos desde o início da execução.

---

## Log Handler

```c
typedef void (*BelaLogHandler)(int level, const char *msg, const char *context);
void bela_set_log_handler(BelaVM *vm, BelaLogHandler handler);
```

Constantes de nível:

```c
#define BELA_LOG_DEBUG  0
#define BELA_LOG_INFO   1
#define BELA_LOG_WARN   2
#define BELA_LOG_ERROR  3
```

O parâmetro `context` carrega pares `key=value` estruturados — mesmo formato do módulo `log` da stdlib (ver `docs/std-lib/log.md`). Exemplo: `"method=GET path=/users ms=42"`. Pode ser `NULL` ou string vazia quando não há contexto adicional.

Se nenhum handler for registrado, o fallback escreve em `stderr`.

```c
// Redirecionar logs de scripts para o sistema de log do engine
void meu_log_handler(int level, const char *msg, const char *context) {
    const char *prefix[] = {"DEBUG", "INFO", "WARN", "ERROR"};
    engine_log("[bela][%s] %s", prefix[level], msg);
    if (context && context[0]) {
        engine_log(" {%s}", context);
    }
}

bela_set_log_handler(vm, meu_log_handler);
```

---

## Panic Handler

```c
typedef void (*BelaPanicHandler)(BelaVM *vm, const char *msg);
void bela_set_panic_handler(BelaVM *vm, BelaPanicHandler handler);
```

> **Nota:** O handler retorna `void` — não pode cancelar o panic. Após o panic, a VM entra em estado `DEAD`. O handler é apenas uma notificação para fins de log ou telemetria; o host deve chamar `bela_vm_free()` e recriar a VM em seguida.

```c
// Logar o panic e sinalizar ao sistema que a VM precisa ser recriada
static volatile bool vm_precisa_recriar = false;

void meu_panic_handler(BelaVM *vm, const char *msg) {
    engine_log("[bela] PANIC: %s\n", msg);
    vm_precisa_recriar = true;
    // VM está DEAD — não fazer mais chamadas a ela
}

bela_set_panic_handler(vm, meu_panic_handler);
```

Após o retorno do handler, qualquer chamada à VM (exceto `bela_vm_free`) tem comportamento indefinido. Ver `embedding.md` para o padrão de recriar a VM após `BELA_PANIC`.

---

## Funções Registradas pelo Host

```c
typedef BelaValue (*BelaCFunction)(BelaVM *vm, BelaValue *argv, int argc);
void bela_register_fn(BelaVM *vm, const char *name, BelaCFunction fn);
```

Funções C registradas ficam disponíveis como funções globais no escopo Bela. O script as chama pelo nome passado em `bela_register_fn`.

> **Nota:** Funções registradas via `bela_register_fn` são sempre consideradas confiáveis e bypass o sandbox. O sandbox controla apenas a stdlib — código nativo do host é soberano. Ver `capi/sandboxing.md`.

```c
// Registrar função que expõe uma capacidade do engine ao script
BelaValue engine_get_fps(BelaVM *vm, BelaValue *argv, int argc) {
    float fps = engine_current_fps();
    return bela_value_float(fps);
}

bela_register_fn(vm, "getFPS", engine_get_fps);
// Agora o script Bela pode chamar: const fps = getFPS()
```

### Reportar Erros de BelaCFunction

> **TBD:** Mecanismo para uma `BelaCFunction` indicar erro ao script Bela. Candidatos:
> - `bela_throw_error(vm, msg)` — script vê `result.ERROR`
> - `bela_panic(vm, msg)` — equivalente a chamar `panic()` no script
>
> A função C chamaria um desses e retornaria `bela_value_null()`. Será definido antes da implementação.

### Type Safety de Argumentos

> **TBD:** Em v1, a responsabilidade de validar tipos em `argv` antes de chamar `bela_as_*()` é do host. Se o script passar `string` onde `int` é esperado, `bela_as_int()` retorna lixo sem aviso. Documentar como responsabilidade do host até que um mecanismo de validação seja definido. Será definido antes da implementação.

### Funções C Variádicas e `fixed<T>`

> **TBD:** Funções registradas via C são opacas para o type checker de Bela. `fixed<T>` não consegue verificar chamadas a funções C registradas. Aceitar como limitação conhecida, ou permitir que o host declare a assinatura via `bela_register_fn_typed(vm, name, signature, fn)`? Será definido antes da implementação.

---

## Handlers Faltando

> **TBD:** Os seguintes handlers estão planejados mas ainda não têm assinatura ou semântica definidas:
> - `bela_set_oom_handler` — notificação antes de OOM terminar a VM
> - `bela_set_timeout_handler` — notificação ao atingir `max_instructions`
> - `bela_set_module_load_handler` — interceptar imports em runtime, permitindo ao host fornecer módulos customizados ou bloquear módulos específicos
>
> Serão definidos antes da implementação.
