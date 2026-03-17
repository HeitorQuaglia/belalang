# os — Sistema Operacional

## Submódulos

| Módulo       | Descrição                                       |
| ------------ | ----------------------------------------------- |
| `os`         | Tipos compartilhados (OSError)                  |
| `os.path`    | Caminhos de arquivos                            |
| `os.env`     | Variáveis de ambiente                           |
| `os.fs`      | Operações estruturais no filesystem             |
| `os.process` | Processo atual e subprocessos                   |
| `os.info`    | Informações do sistema                          |
| `os.signal`  | Tratamento de sinais do SO                      |

---

## os — Tipos Compartilhados

```
enum OSError {
    NOT_FOUND(string),
    PERMISSION_DENIED(string),
    ALREADY_EXISTS(string),
    NOT_EMPTY(string),
    UNEXPECTED(string)
}
```

---

## os.path — Caminhos de Arquivos

```
from os.path use Path
```

`Path` representa um caminho no sistema de arquivos. Imutável — cada operação retorna um novo `Path`.

### Classe

```
class Path {
    fn (raw: string)

    // --- Propriedades ---

    val raw: string                                  // caminho original

    static val SEPARATOR: string                     // "/" em Unix, "\" em Windows

    // --- Navegação ---

    fn parent(): Path                                 // null se raiz
    fn join(other: string): Path                     // path.join("file.txt")
    fn resolve(base: Path): Path                     // resolve relativo contra base

    // --- Componentes ---

    fn basename(): string                            // "/home/user/file.txt" → "file.txt"
    fn stem(): string                                // "file.txt" → "file"
    fn extension(): string                            // null se sem extensão
    fn dirname(): Path                               // "/home/user/file.txt" → "/home/user"
    fn segments(): array<string>                     // "/home/user" → ["home", "user"]

    // --- Consultas ---

    fn isAbsolute(): bool
    fn isRelative(): bool
    fn normalize(): Path                             // resolve "." e ".."

    // --- Conversão ---

    fn toString(): string
    fn toUrl(): string                               // "file:///home/user/file.txt"
}
```

### Exemplos

```
from os.path use Path

p = Path("/home/user/docs/file.txt")

dir = p.dirname()                    // Path("/home/user/docs")
name = p.basename()                  // "file.txt"
ext = p.extension()                  // "txt"
stem = p.stem()                      // "file"

sibling = p.dirname().join("other.txt")
// Path("/home/user/docs/other.txt")

messy = Path("/home/user/../user/./docs")
clean = messy.normalize()            // Path("/home/user/docs")

parts = p.segments()                 // ["home", "user", "docs", "file.txt"]
```

---

## os.env — Variáveis de Ambiente

```
from os.env use get, set, remove, contains, entries
```

### Funções

```
fn get(key: string): string                          // null se não existe
fn set(key: string, value: string): void
fn remove(key: string): void
fn contains(key: string): bool
fn entries(): array<array<string>>                   // [["KEY", "value"], ...]
```

### Exemplos

```
from os.env use get, set, remove, contains

home = get("HOME")                    // "/home/user"
missing = get("UNDEFINED")            // null

set("APP_ENV", "production")
exists = contains("APP_ENV")          // true

remove("APP_ENV")
```

---

## os.fs — Sistema de Arquivos

```
from os.fs use mkdir, readDir, stat, FileType, FileStat, DirEntry
```

Operações estruturais no filesystem. Para leitura/escrita de conteúdo de arquivos, veja `io.file`.

### Enums

```
enum FileType {
    FILE,
    DIRECTORY,
    SYMLINK
}
```

### Classes

```
class FileStat {
    val path: Path
    val fileType: FileType
    val size: int                                    // bytes
    val createdAt: int                               // timestamp unix
    val modifiedAt: int                              // timestamp unix
    val permissions: int                             // bits de permissão
    val isReadonly: bool
}

class DirEntry {
    val name: string
    val path: Path
    val fileType: FileType
}
```

### Funções

```
// --- Diretórios ---

fn mkdir(path: string): result<void, OSError>
fn mkdirAll(path: string): result<void, OSError>     // cria intermediários
fn rmdir(path: string): result<void, OSError>         // somente se vazio

// --- Remoção ---

fn remove(path: string): result<void, OSError>        // arquivo ou symlink
fn removeAll(path: string): result<void, OSError>     // recursivo

// --- Consultas ---

fn exists(path: string): bool
fn isFile(path: string): bool
fn isDir(path: string): bool
fn stat(path: string): result<FileStat, OSError>

// --- Listagem ---

fn readDir(path: string): result<array<DirEntry>, OSError>

// --- Links simbólicos ---

fn symlink(target: string, link: string): result<void, OSError>
fn readLink(path: string): result<string, OSError>

// --- Renomear ---

fn rename(from: string, to: string): result<void, OSError>
```

### Exemplos

```
from os.fs use mkdir, mkdirAll, readDir, stat, exists, isDir, remove, removeAll, rename

// Criar diretórios
try mkdir("output")
try mkdirAll("data/cache/images")

// Consultar
e = exists("output")                  // true
d = isDir("output")                   // true

// Metadata
info = try stat("config.json")
println("Tamanho: ${info.size} bytes")
println("Modificado: ${info.modifiedAt}")

// Listar diretório
entries = try readDir("src")
for (entry in entries) {
    println("${entry.name} (${entry.fileType})")
}

// Renomear e remover
try rename("old.txt", "new.txt")
try remove("temp.txt")
try removeAll("data/cache")
```

---

## os.process — Processos

```
from os.process use pid, args, exit, cwd, exec, spawn, Process, ProcessResult
```

### Funções — Processo Atual

```
fn pid(): int
fn args(): array<string>                             // argumentos da linha de comando
fn cwd(): result<string, OSError>
fn exit(code: int): void                             // encerra o programa
```

### Classes — Subprocessos

```
class ProcessResult {
    val exitCode: int
    val stdout: string
    val stderr: string

    fn success(): bool                               // exitCode == 0
}

class Process {
    val pid: int

    fn wait(): result<ProcessResult, OSError>
    fn kill(): result<void, OSError>

    // --- Streams (implementam io.stream) ---

    val stdin: Writer
    val stdout: Reader
    val stderr: Reader
}
```

### Funções — Subprocessos

```
fn exec(command: string, args: array<string> = []): result<ProcessResult, OSError>
fn spawn(command: string, args: array<string> = []): result<Process, OSError>
```

`exec` executa e aguarda o término. `spawn` inicia o processo e retorna imediatamente — use `wait()` para aguardar.

### Exemplos

```
from os.process use pid, args, cwd, exec, spawn

// Processo atual
myPid = pid()                         // 12345
myArgs = args()                       // ["--verbose", "input.txt"]
dir = try cwd()                       // "/home/user/project"

// Executar e aguardar
res = try exec("ls", ["-la"])
if (res.success()) {
    println(res.stdout)
} else {
    eprintln("Erro: ${res.stderr}")
}

// Spawn e aguardar depois
proc = try spawn("long-task", ["--flag"])
// ... fazer outras coisas ...
output = try proc.wait()
println("Saiu com código: ${output.exitCode}")
```

---

## os.info — Informações do Sistema

```
from os.info use osName, arch, hostname, cpus, totalMemory, homeDir, tempDir
```

### Funções

```
fn osName(): string                                  // "linux", "macos", "windows"
fn arch(): string                                    // "x86_64", "aarch64"
fn hostname(): string
fn cpus(): int                                       // número de CPUs lógicas
fn totalMemory(): int                                // bytes de RAM total
fn homeDir(): result<string, OSError>
fn tempDir(): string
```

### Exemplos

```
from os.info use osName, arch, hostname, cpus, totalMemory, homeDir, tempDir

println("OS: ${osName()} (${arch()})")    // "OS: linux (x86_64)"
println("Host: ${hostname()}")
println("CPUs: ${cpus()}")
println("RAM: ${totalMemory() / 1024 / 1024} MB")

home = try homeDir()                      // "/home/user"
tmp = tempDir()                           // "/tmp"
```

---

## os.signal — Sinais do SO

```
from os.signal use Signal, onSignal
```

Permite registrar handlers para sinais do sistema operacional.

### Enum

```
enum Signal {
    INT,                                             // Ctrl+C (SIGINT)
    TERM,                                            // encerramento gracioso (SIGTERM)
    HUP,                                             // hangup (SIGHUP)
    USR1,                                            // definido pelo usuário
    USR2                                             // definido pelo usuário
}
```

### Funções

```
fn onSignal(signal: Signal, handler: fn(): void): void
```

### Exemplo

```
from os.signal use Signal, onSignal

onSignal(Signal.INT, fn(): void => {
    println("Encerrando...")
    cleanup()
    exit(0)
})

onSignal(Signal.TERM, fn(): void => {
    println("SIGTERM recebido")
    exit(0)
})
```