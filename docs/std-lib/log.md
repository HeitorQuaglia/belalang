# log — Logging

O módulo `log` fornece logging estruturado com níveis. Em contexto de embedding, o host pode registrar um handler personalizado para interceptar todas as mensagens de log. Em modo standalone (REPL, `bela run`), o fallback padrão escreve em stderr.

```
from log use logger
```

---

## Níveis de Log

```
enum LogLevel {
    DEBUG,    // diagnóstico detalhado — desabilitado por padrão
    INFO,     // informações gerais de execução
    WARN,     // situação anormal mas recuperável
    ERROR     // erro que afeta a operação
}
```

---

## logger — Logger Global

O logger global está disponível via import. É configurado pelo host ou usa o fallback padrão.

### Métodos

```
fn debug(msg: string, ...args: dynamic): void
fn info(msg: string, ...args: dynamic): void
fn warn(msg: string, ...args: dynamic): void
fn error(msg: string, ...args: dynamic): void

fn setLevel(level: LogLevel): void        // filtra mensagens abaixo do nível
fn getLevel(): LogLevel
```

### Formato de mensagem

```
from log use logger
from io use println

logger.info("servidor iniciando na porta $port")
logger.warn("conexão lenta detectada: ${ms}ms")
logger.error("falha ao conectar: $err")

// Com argumentos adicionais (contexto estruturado key=value)
logger.info("requisição processada", "method", "GET", "path", "/users", "ms", 42)
```

Saída padrão (fallback stderr):

```
[INFO]  servidor iniciando na porta 8080
[WARN]  conexão lenta detectada: 2341ms
[ERROR] falha ao conectar: connection refused
[INFO]  requisição processada method=GET path=/users ms=42
```

---

## Handler Personalizado (Embedding)

Em contexto de embedding, o host registra um handler via C API. O handler recebe cada mensagem de log e decide o que fazer com ela (redirecionar para o sistema de log do jogo, gravar em arquivo, enviar para um servidor de telemetria, etc.).

```c
// C API — host registra handler antes de executar scripts
typedef void (*BelaLogHandler)(int level, const char *msg, const char *context);

void bela_set_log_handler(BelaVM *vm, BelaLogHandler handler);
```

```c
// Exemplo: redirecionar para o sistema de log do engine
void my_log_handler(int level, const char *msg, const char *context) {
    switch (level) {
        case BELA_LOG_DEBUG: engine_log_debug("[script] %s", msg); break;
        case BELA_LOG_INFO:  engine_log_info("[script] %s", msg);  break;
        case BELA_LOG_WARN:  engine_log_warn("[script] %s", msg);  break;
        case BELA_LOG_ERROR: engine_log_error("[script] %s", msg); break;
    }
}

BelaVM *vm = bela_vm_new();
bela_set_log_handler(vm, my_log_handler);
bela_vm_load_file(vm, "scripts/ai.bela");
```

O script não muda — usa `logger.info(...)` normalmente. O handler do host intercepta todas as chamadas.

---

## Logger Customizado

Para criar loggers com configuração própria (ex: prefixo de contexto), use `Logger`:

```
from log use Logger, LogLevel

// Logger com prefixo
httpLog = Logger("http")
httpLog.setLevel(LogLevel.DEBUG)

httpLog.info("requisição recebida: $method $path")
// [INFO][http] requisição recebida: GET /users

// Logger que filtra DEBUG em produção
prodLog = Logger("app")
prodLog.setLevel(LogLevel.INFO)
prodLog.debug("detalhe interno")    // silenciado
prodLog.info("tudo certo")          // aparece
```

### Classe Logger

```
class Logger {
    constructor(name: string)

    fn debug(msg: string, ...args: dynamic): void
    fn info(msg: string, ...args: dynamic): void
    fn warn(msg: string, ...args: dynamic): void
    fn error(msg: string, ...args: dynamic): void
    fn setLevel(level: LogLevel): void
    fn getLevel(): LogLevel

    val name: string
}
```

---

## Modo Standalone (REPL / `bela run`)

Quando nenhum handler é registrado pelo host, o fallback padrão:

- `DEBUG`, `INFO` → stderr (ou stdout se `-v` flag ativa)
- `WARN`, `ERROR` → stderr sempre
- Nível padrão: `INFO` (DEBUG silenciado)

```bash
bela run app.bela              # INFO+ para stderr
bela run app.bela --log=debug  # DEBUG+ para stderr
bela run app.bela --log=error  # só ERROR para stderr
```
