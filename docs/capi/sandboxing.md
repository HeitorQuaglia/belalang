# C API — Sandboxing

O sandbox de Bela controla exclusivamente o acesso à **stdlib**. Funções e módulos registrados pelo host via `bela_register_fn()` são sempre confiáveis — o host é soberano sobre o que expõe ao script. Toda configuração de sandbox deve ocorrer **antes do primeiro `bela_vm_load_*()`**: após o primeiro carregamento, o sandbox é travado e qualquer tentativa de modificação retorna `BELA_SANDBOX_LOCKED`. A regra de precedência "última chamada vence" aplica-se apenas pré-lock; pós-lock, todas as tentativas retornam erro.

---

## Níveis de Conveniência (BelaSandboxLevel)

A API de conveniência define cinco níveis progressivos de acesso. Cada nível inclui tudo que o nível anterior libera.

```c
typedef enum {
    BELA_SANDBOX_PURE,        // apenas computação (default de embedding)
    BELA_SANDBOX_CONTROLLED,  // + console, os.env (read-only), os.info
    BELA_SANDBOX_FILESYSTEM,  // + io.file, os.fs (com root confinado)
    BELA_SANDBOX_NETWORK,     // + net.http.client (com domain whitelist)
    BELA_SANDBOX_FULL         // tudo (modo standalone)
} BelaSandboxLevel;

void bela_sandbox_set_level(BelaVM *vm, BelaSandboxLevel level);
```

| Nível                      | O que libera                                                                    |
|----------------------------|---------------------------------------------------------------------------------|
| `BELA_SANDBOX_PURE`        | Apenas módulos de computação pura (ver tabela abaixo). Nenhum I/O.              |
| `BELA_SANDBOX_CONTROLLED`  | `PURE` + `console`, `os.env` (read-only), `os.info`.                           |
| `BELA_SANDBOX_FILESYSTEM`  | `CONTROLLED` + `io.file`, `os.fs` (confinado ao root definido por `bela_sandbox_set_root`). |
| `BELA_SANDBOX_NETWORK`     | `FILESYSTEM` + `net.http.client` (restrito à whitelist de hosts).               |
| `BELA_SANDBOX_FULL`        | Tudo. Equivalente ao modo standalone — sem restrições de stdlib.                |

```c
// Exemplo: embedding de scripts de IA em jogo (acesso controlado)
bela_sandbox_set_level(vm, BELA_SANDBOX_CONTROLLED);

// Exemplo: plugin de processamento de arquivos (filesystem confinado)
bela_sandbox_set_root(vm, "/var/game/mods/");
bela_sandbox_set_level(vm, BELA_SANDBOX_FILESYSTEM);
```

---

## Módulos por Nível de Acesso

| Módulo         | Sempre disponível | Requer allow |
|----------------|:-----------------:|:------------:|
| `math`         | ✓                 |              |
| `string`       | ✓                 |              |
| `collections`  | ✓                 |              |
| `encoding`     | ✓                 |              |
| `regex`        | ✓                 |              |
| `random`       | ✓                 |              |
| `log`          | ✓                 |              |
| `time.now()`   | ✓                 |              |
| `time.sleep()` |                   | ✓            |
| `io`           |                   | ✓            |
| `os.*`         |                   | ✓            |
| `net.*`        |                   | ✓            |
| `concurrency`  |                   | ✓            |

Módulos "sempre disponíveis" funcionam em qualquer nível de sandbox, incluindo `BELA_SANDBOX_PURE`. Eles não realizam I/O e não acessam recursos externos.

---

## API Granular

Para controle fino, além dos níveis de conveniência, é possível habilitar ou desabilitar módulos individualmente:

```c
void bela_vm_allow_module(BelaVM *vm, const char *module);
void bela_vm_deny_module(BelaVM *vm, const char *module);
```

> **Nota:** A última chamada vence (pré-lock). É possível combinar nível de conveniência com deny granular. Exemplo: `bela_sandbox_set_level(vm, BELA_SANDBOX_FULL)` seguido de `bela_vm_deny_module(vm, "os.process")` resulta em acesso total exceto `os.process`.

```c
// Liberar tudo exceto os.process
bela_sandbox_set_level(vm, BELA_SANDBOX_FULL);
bela_vm_deny_module(vm, "os.process");

// Liberar apenas net, sem filesystem
bela_vm_allow_module(vm, "net.http.client");
```

---

## Confinamento de Filesystem

Quando `BELA_SANDBOX_FILESYSTEM` (ou superior) está ativo, o acesso a arquivos é restrito a um diretório raiz:

```c
void bela_sandbox_set_root(BelaVM *vm, const char *path);
```

O script não consegue acessar caminhos fora do root configurado. Tentativas de leitura ou escrita fora do root resultam em erro de runtime no script.

> **TBD:** Path traversal e canonicalização estão pendentes de definição. Questões em aberto: canonicalização na VM ou no OS? Comportamento em Windows com UNC paths (`\\server\share`), drive letters (`C:\`) e junction points. Será definido antes da implementação.

```c
// Confinar scripts ao diretório de mods do jogo
bela_sandbox_set_root(vm, "/var/game/mods/");
bela_sandbox_set_level(vm, BELA_SANDBOX_FILESYSTEM);
```

---

## Domain Whitelist para net

Mesmo com `net` habilitado, é possível restringir quais hosts o script pode acessar:

```c
void bela_sandbox_allow_hosts(BelaVM *vm, const char **hosts, int count);
```

A whitelist é uma restrição adicional sobre o nível de sandbox — sem `bela_sandbox_allow_hosts`, todos os hosts são permitidos quando `net` está ativo. Com a whitelist configurada, qualquer tentativa de acesso a host não listado resulta em erro de runtime.

```c
const char *hosts[] = { "api.meuservico.com", "cdn.meuservico.com" };
bela_sandbox_allow_hosts(vm, hosts, 2);
bela_sandbox_set_level(vm, BELA_SANDBOX_NETWORK);
```

---

## Limites de Recursos

Os limites de recursos são configurados em `BelaConfig` ao criar a VM, não via funções de sandbox separadas:

```c
BelaConfig cfg = {
    .max_memory       = 64 * 1024 * 1024,  // 64 MB
    .max_instructions = 1000000,            // 1 M instruções por execução
};
BelaVM *vm = bela_vm_new(&cfg);
```

| Campo              | Tipo     | Descrição                                            | Default          |
|--------------------|----------|------------------------------------------------------|------------------|
| `max_memory`       | `size_t` | Limite de heap em bytes. `0` = sem limite.           | `0` (sem limite) |
| `max_instructions` | `size_t` | Limite de instruções por execução. `0` = sem limite. | `0` (sem limite) |

> **Nota:** Não existem funções separadas de sandboxing para limites de recursos. Toda configuração de limites ocorre em `BelaConfig` e é imutável após `bela_vm_new`.

> **TBD:** Unificação de `bela_sandbox_set_root` e `bela_sandbox_allow_hosts` em `BelaConfig` está em aberto. Atualmente alguns settings de sandbox são em `BelaConfig` (limites de recursos) e outros são chamadas separadas (root, whitelist de hosts). Será definido antes da implementação.

---

## Violation Handler

> **TBD:** `bela_set_sandbox_violation_handler` — notificação quando um script tenta acessar um módulo não permitido. A assinatura candidata é `(BelaVM *vm, const char *module_name, const char *script_location)`, útil para auditoria e logging de segurança. Será definido antes da implementação.

---

## Host-Registered Functions e Sandbox

Funções registradas via `bela_register_fn()` **não são controladas pelo sandbox**. O sandbox controla exclusivamente a stdlib de Bela. Código C registrado pelo host é sempre tratado como confiável porque o host o registrou explicitamente.

```c
// Esta função C é sempre acessível ao script, independente do nível de sandbox
bela_register_fn(vm, "lerEstadoJogo", minha_fn_ler_estado);

// Mesmo com BELA_SANDBOX_PURE, o script pode chamar lerEstadoJogo()
bela_sandbox_set_level(vm, BELA_SANDBOX_PURE);
```

> **Atenção de segurança:** Se uma função C registrada pelo host realiza operações privilegiadas (acesso a arquivos, rede, processos), o sandbox não impedirá que o script as utilize. O controle de acesso de funções host-registered é responsabilidade exclusiva do host. Registre apenas o que o script deve ter acesso.

```c
// ERRADO: registrar função que lê arquivo arbitrário em sandbox restrito
bela_register_fn(vm, "lerArquivo", fn_que_le_arquivo_qualquer);  // bypass total de sandbox
bela_sandbox_set_level(vm, BELA_SANDBOX_PURE);                   // não protege contra o acima

// CORRETO: registrar função com lógica de controle de acesso própria
bela_register_fn(vm, "lerAsset", fn_que_le_apenas_assets_do_jogo);
bela_sandbox_set_level(vm, BELA_SANDBOX_PURE);
```
