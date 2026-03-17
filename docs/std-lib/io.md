# io — Entrada e Saída

> **Nota:** Belalang usa tipagem dinâmica. As anotações de tipo nas assinaturas
> (incluindo `result<T, E>`) são mantidas como documentação — não são impostas
> pelo compilador.

## Submódulos

| Módulo       | Descrição                              |
| ------------ | -------------------------------------- |
| `io`         | Tipos compartilhados (IOError, Encoding) |
| `io.console` | Entrada e saída padrão (stdin/stdout/stderr) |
| `io.file`    | Operações com arquivos (sync e async)  |
| `io.stream`  | Interfaces Reader, Writer, Closer      |

---

## io — Tipos Compartilhados

```
enum IOError {
    NOT_FOUND(string),
    PERMISSION_DENIED(string),
    ALREADY_EXISTS(string),
    UNEXPECTED(string)
}

enum Encoding {
    UTF8,
    UTF16,
    ASCII,
    LATIN1,
    PLATFORM        // UTF-8 em Linux/macOS, UTF-16 em Windows
}
```

---

## io.console

```
from io.console use print, println, eprint, eprintln, readln
```

### Funções

```
fn print(value: string): void               // stdout, sem newline
fn println(value: string): void             // stdout, com newline
fn eprint(value: string): void              // stderr, sem newline
fn eprintln(value: string): void            // stderr, com newline
fn readln(): result<string, IOError>        // lê uma linha de stdin
```

### Exemplo

```
from io.console use print, println, readln

fn main(): result<void, IOError> {
    print("Nome: ")
    name = try readln()
    println("Olá, $name!")
    return result.OK(())
}
```

---

## io.file

```
from io.file use File, AsyncFile, FileMode, SeekOrigin
```

### Enums

```
enum FileMode {
    READ,
    WRITE,
    APPEND,
    READ_WRITE
}

enum SeekOrigin {
    BEGIN,
    CURRENT,
    END
}
```

### File (sync)

```
class File implements Reader, Writer, Closer {

    // --- Conveniência (static) ---

    static fn open(path: string, mode: FileMode,
                   encoding: Encoding = Encoding.PLATFORM): result<File, IOError>

    static fn read(path: string,
                   encoding: Encoding = Encoding.PLATFORM): result<string, IOError>

    static fn readBytes(path: string): result<array<int>, IOError>

    static fn write(path: string, content: string,
                    encoding: Encoding = Encoding.PLATFORM): result<void, IOError>

    static fn append(path: string, content: string,
                     encoding: Encoding = Encoding.PLATFORM): result<void, IOError>

    static fn exists(path: string): bool

    static fn delete(path: string): result<void, IOError>

    static fn copy(src: string, dest: string): result<void, IOError>

    static fn move(src: string, dest: string): result<void, IOError>

    // --- Instância ---

    fn read(size: int): result<string, IOError>
    fn readln(): result<string, IOError>
    fn readAll(): result<string, IOError>
    fn readBytes(size: int): result<array<int>, IOError>
    fn write(content: string): result<void, IOError>
    fn writeln(content: string): result<void, IOError>
    fn writeBytes(data: array<int>): result<void, IOError>
    fn seek(offset: int, origin: SeekOrigin): result<void, IOError>
    fn flush(): result<void, IOError>
    fn close(): void
}
```

### AsyncFile (async)

Mesma API que `File`, mas todas as operações são `async`.

```
class AsyncFile implements Reader, Writer, Closer {

    // --- Conveniência (static) ---

    static async fn open(path: string, mode: FileMode,
                         encoding: Encoding = Encoding.PLATFORM): result<AsyncFile, IOError>

    static async fn read(path: string,
                         encoding: Encoding = Encoding.PLATFORM): result<string, IOError>

    static async fn readBytes(path: string): result<array<int>, IOError>

    static async fn write(path: string, content: string,
                          encoding: Encoding = Encoding.PLATFORM): result<void, IOError>

    static async fn append(path: string, content: string,
                           encoding: Encoding = Encoding.PLATFORM): result<void, IOError>

    static async fn delete(path: string): result<void, IOError>

    static async fn copy(src: string, dest: string): result<void, IOError>

    static async fn move(src: string, dest: string): result<void, IOError>

    // --- Instância ---

    async fn read(size: int): result<string, IOError>
    async fn readln(): result<string, IOError>
    async fn readAll(): result<string, IOError>
    async fn readBytes(size: int): result<array<int>, IOError>
    async fn write(content: string): result<void, IOError>
    async fn writeln(content: string): result<void, IOError>
    async fn writeBytes(data: array<int>): result<void, IOError>
    async fn seek(offset: int, origin: SeekOrigin): result<void, IOError>
    async fn flush(): result<void, IOError>
    async fn close(): void
}
```

### Exemplos

#### Leitura rápida

```
from io.file use File

fn main(): result<void, IOError> {
    content = try File.read("data.txt")
    println(content)
    return result.OK(())
}
```

#### Escrita com File aberto

```
from io.file use File, FileMode

fn writeLog(msg: string): result<void, IOError> {
    f = try File.open("app.log", FileMode.APPEND)
    try f.writeln(msg)
    f.close()
    return result.OK(())
}
```

#### Async

```
from io.file use AsyncFile

async fn processFiles(): result<void, IOError> {
    content = try await AsyncFile.read("input.txt")
    try await AsyncFile.write("output.txt", content)
    return result.OK(())
}
```

#### Encoding explícito

```
from io use Encoding
from io.file use File

fn readWindows(): result<string, IOError> {
    return File.read("legacy.txt", Encoding.UTF16)
}
```

---

## io.stream

Interfaces para composição. `File` e `AsyncFile` implementam estas interfaces.

```
interface Reader {
    fn read(size: int): result<string, IOError>
    fn readAll(): result<string, IOError>
}

interface Writer {
    fn write(content: string): result<void, IOError>
    fn flush(): result<void, IOError>
}

interface Closer {
    fn close(): void
}
```

### Exemplo com interfaces

```
from io.stream use Reader, Writer
from io use IOError

fn copy(src: Reader, dest: Writer): result<void, IOError> {
    content = try src.readAll()
    try dest.write(content)
    try dest.flush()
    return result.OK(())
}
```