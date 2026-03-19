# C API — Embedding

A C API de Bela segue um ciclo de vida linear: criar a VM, configurar sandbox e callbacks, carregar código, executar funções e liberar a VM. Todas as configurações devem ser feitas antes do primeiro `bela_vm_load_*` — após o carregamento, o sandbox é travado e tentativas de alteração retornam `BELA_SANDBOX_LOCKED`.

> **Nota:** Após um `BELA_PANIC`, a VM entra em estado `DEAD` e não pode ser reutilizada. Para hosts long-running, a abordagem correta é liberar a VM com `bela_vm_free` e recriar com `bela_vm_new`.

---

## Ciclo de Vida da VM

```
bela_vm_new()
    → [configurar BelaConfig]
    → [bela_set_log_handler / bela_set_panic_handler]
    → [bela_sandbox_set_level / bela_sandbox_set_root]
    → bela_vm_load_file() | bela_vm_load_string() | bela_vm_load_string_named()
        ↓  (sandbox travado aqui — BELA_SANDBOX_LOCKED se tentar modificar após)
    → bela_vm_call()
    → bela_vm_free()
```

> Todas as configurações de sandbox e handlers **devem** ser feitas antes do primeiro `bela_vm_load_*`. Após o carregamento, o sandbox é imutável.

---

## BelaConfig

`BelaConfig` define os limites de recursos da VM. É passada para `bela_vm_new` e não pode ser alterada após a criação.

```c
typedef struct {
    size_t max_memory;
    size_t max_instructions;
    int    max_stack;
    char  *debug_host;
    char  *module_search_path;
    bool   strict_mode;
} BelaConfig;
```

| Campo                | Tipo     | Descrição                                                     | Default          |
|----------------------|----------|---------------------------------------------------------------|------------------|
| `max_memory`         | `size_t` | Limite de memória em bytes. `0` = sem limite.                 | `0` (sem limite) |
| `max_instructions`   | `size_t` | Limite de instruções por execução. `0` = sem limite.          | `0` (sem limite) |
| `max_stack`          | `int`    | Profundidade máxima de call frames.                           | `256`            |
| `debug_host`         | `char *` | Endereço do debugger a conectar.                              | —                |
| `module_search_path` | `char *` | Caminho adicional para busca de módulos além da stdlib.       | —                |
| `strict_mode`        | `bool`   | Força `fixed<T>` em todo o código, sem uso de `dynamic`.      | —                |

> **TBD:** Os campos `debug_host`, `module_search_path` e `strict_mode` estão reservados na struct mas sua semântica exata será definida antes da implementação.

> `max_stack` conta **call frames** (não bytes de stack). Uma profundidade de `256` frames é suficiente para a maioria dos scripts.

---

## Funções

### bela_vm_new / bela_vm_free

```c
BelaVM *bela_vm_new(const BelaConfig *config);
void    bela_vm_free(BelaVM *vm);
```

`bela_vm_new` aloca e inicializa a VM com as configurações fornecidas. Passar `NULL` em `config` usa todos os defaults. `bela_vm_free` libera todos os recursos — deve ser chamada mesmo após `BELA_PANIC`.

```c
// Exemplo mínimo
BelaVM *vm = bela_vm_new(NULL);
// ... carregar e executar ...
bela_vm_free(vm);

// Com configuração de limites
BelaConfig cfg = {
    .max_memory       = 64 * 1024 * 1024,  // 64 MB
    .max_instructions = 1000000,
    .max_stack        = 128,
};
BelaVM *vm = bela_vm_new(&cfg);
```

---

### bela_vm_load_file

```c
BelaStatus bela_vm_load_file(BelaVM *vm, const char *path);
```

Carrega código a partir de um arquivo. A VM detecta automaticamente o formato pelo header do arquivo:

- `.bela` — código-fonte (compilado em tempo de carregamento)
- `.belc` — bytecode pré-compilado

A detecção é baseada no header binário do arquivo, não na extensão. Um arquivo `.bela` com header de bytecode é carregado como bytecode; um `.belc` corrompido ou de versão incompatível retorna `BELA_COMPILE_ERROR`.

> Esta função é a referência autoritativa para o comportamento de auto-detecção source vs. bytecode. `toolchain/bytecode.md` faz referência cruzada a esta seção.

Após o retorno desta função, o sandbox é travado. Chamadas subsequentes a `bela_sandbox_set_level` ou similares retornam `BELA_SANDBOX_LOCKED`.

```c
BelaStatus status = bela_vm_load_file(vm, "scripts/ai.bela");
if (status != BELA_OK) {
    fprintf(stderr, "falha ao carregar: %s\n", bela_vm_last_error(vm));
    bela_vm_free(vm);
    return;
}
```

---

### bela_vm_load_string / bela_vm_load_string_named

```c
BelaStatus bela_vm_load_string(BelaVM *vm, const char *source);
BelaStatus bela_vm_load_string_named(BelaVM *vm, const char *source, const char *name);
```

Carrega código-fonte a partir de uma string em memória. `bela_vm_load_string` é equivalente a `bela_vm_load_string_named(vm, source, "??")`.

`bela_vm_load_string_named` permite fornecer um nome para o chunk. O nome aparece nas mensagens de erro e stack traces em vez de `??`:

```
// Sem nome:   ??:42: variável 'x' não definida
// Com nome:   <repl>:42: variável 'x' não definida
```

Use `"<repl>"`, `"<stdin>"` ou um identificador descritivo. Para scripts embutidos em assets, use o caminho do asset como nome.

```c
// REPL: cada linha digitada pelo usuário
BelaStatus s = bela_vm_load_string_named(vm, line, "<repl>");

// Script inline com nome descritivo
BelaStatus s = bela_vm_load_string_named(vm, code, "config/startup");
```

---

### bela_vm_call

```c
BelaStatus bela_vm_call(BelaVM *vm, const char *fn_name, int argc, BelaValue *argv);
```

Chama uma função Bela pelo nome. A função deve ter sido definida no código carregado anteriormente via `bela_vm_load_*`.

`argc` e `argv` correspondem aos argumentos posicionais. Para chamar função sem argumentos, passe `0` e `NULL`.

> **TBD:** A assinatura atual não possui parâmetro de saída para o valor de retorno da função Bela. Candidatos: `(a)` adicionar `BelaValue *out` como último parâmetro; `(b)` variante `bela_vm_call_v` que retorna `BelaValue`; `(c)` `bela_vm_last_result(vm)` para ler o retorno após a chamada. Será definido antes da implementação.

```c
BelaValue args[2] = {
    bela_value_int(42),
    bela_value_string(vm, "olá"),
};
BelaStatus status = bela_vm_call(vm, "minhaFuncao", 2, args);
if (status == BELA_PANIC) {
    // VM está DEAD — liberar e recriar
    bela_vm_free(vm);
}
```

---

### bela_vm_eval

> **TBD:** Se a expressão avaliada causa runtime error, `bela_vm_eval` não pode usar `null` como
> sentinela de erro — `null` é um valor Bela válido. Candidatos:
> - `BelaValue bela_vm_eval(BelaVM *vm, const char *expr)` com introspection pós-chamada via `bela_vm_last_error`
> - `BelaStatus bela_vm_eval_safe(BelaVM *vm, const char *expr, BelaValue *out)`
>
> Será definido antes da implementação.

---

### bela_vm_call_method

> **TBD:** Chamada de método em objeto Bela do lado C. `bela_vm_call` só chama funções top-level. Para chamar `enemy.takeDamage(10)` do C seria necessário `bela_vm_call_method(vm, obj, method_name, argc, argv)`. Será definido antes da implementação.

---

### Múltiplos loads

> **TBD:** A semântica de múltiplos `bela_vm_load_*` no mesmo VM não está definida. Questões em aberto: os namespaces se mesclam? É possível carregar uma biblioteca base e depois vários scripts de usuário compartilhando o mesmo escopo global? Será definido antes da implementação.

---

### async fn do C

> **TBD:** Se `bela_vm_call` chama uma função `async`, o retorno é um `Future<T>`. O host precisaria de `bela_vm_tick(vm)` em loop até o future resolver, mais API para detectar se o retorno é um Future, aguardar resolução e obter o valor final. Será definido antes da implementação.

---

### bela_vm_last_error

> **TBD:** Retorna detalhes do último erro da VM (mensagem de texto). Assinatura e
> ownership da string a definir. Será definido antes da implementação.

---

## BelaStatus

Enum de retorno usado por todas as funções `bela_vm_load_*` e `bela_vm_call`.

```c
typedef enum {
    BELA_OK,
    BELA_ERROR,
    BELA_PANIC,
    BELA_COMPILE_ERROR,
    BELA_SANDBOX_LOCKED,
} BelaStatus;
```

| Valor                 | Quando ocorre                                                                       |
|-----------------------|-------------------------------------------------------------------------------------|
| `BELA_OK`             | Sucesso.                                                                            |
| `BELA_ERROR`          | Erro de runtime recuperável. VM permanece operacional.                              |
| `BELA_PANIC`          | Panic irrecuperável. VM entra em estado `DEAD` — deve ser freed e recriada.         |
| `BELA_COMPILE_ERROR`  | Erro de sintaxe no código-fonte, ou versão de bytecode incompatível.                |
| `BELA_SANDBOX_LOCKED` | Tentativa de modificar o sandbox após o primeiro `bela_vm_load_*`.                  |

Após `BELA_PANIC`, qualquer chamada à VM (exceto `bela_vm_free`) tem comportamento indefinido.

---

## Exemplo Completo

```c
#include "bela.h"

static void meu_log_handler(int level, const char *msg, const char *context) {
    const char *prefix[] = {"DEBUG", "INFO", "WARN", "ERROR"};
    fprintf(stderr, "[bela][%s] %s", prefix[level], msg);
    if (context && context[0]) fprintf(stderr, " {%s}", context);
    fprintf(stderr, "\n");
}

static void meu_panic_handler(BelaVM *vm, const char *msg) {
    fprintf(stderr, "[bela] PANIC: %s\n", msg);
    // handler é só notificação — VM já está DEAD aqui, apenas logar e recriar
}

int main(void) {
    // 1. Criar VM com limites de recursos
    BelaConfig cfg = {
        .max_memory       = 32 * 1024 * 1024,  // 32 MB
        .max_instructions = 500000,
        .max_stack        = 256,
    };
    BelaVM *vm = bela_vm_new(&cfg);

    // 2. Registrar callbacks (antes do load)
    bela_set_log_handler(vm, meu_log_handler);
    bela_set_panic_handler(vm, meu_panic_handler);

    // 3. Configurar sandbox (antes do load)
    bela_sandbox_set_level(vm, BELA_SANDBOX_CONTROLLED);

    // 4. Carregar código — sandbox travado após este ponto
    BelaStatus status = bela_vm_load_file(vm, "scripts/ai.bela");
    if (status != BELA_OK) {
        fprintf(stderr, "falha ao carregar: %s\n", bela_vm_last_error(vm));
        bela_vm_free(vm);
        return 1;
    }

    // 5. Chamar função Bela
    BelaValue args[1] = { bela_value_float(0.016) };  // delta_time
    status = bela_vm_call(vm, "onUpdate", 1, args);

    if (status == BELA_PANIC) {
        // VM está DEAD — não reutilizar
        bela_vm_free(vm);
        return 1;
    }

    // 6. Liberar
    bela_vm_free(vm);
    return 0;
}
```
