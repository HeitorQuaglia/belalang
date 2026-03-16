# os — Sistema Operacional

## Submódulos

| Módulo    | Descrição          |
| --------- | ------------------ |
| `os.path` | Caminhos de arquivos |

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

    fn parent(): optional<Path>                      // "/home/user" → "/home"
    fn join(other: string): Path                     // path.join("file.txt")
    fn resolve(base: Path): Path                     // resolve relativo contra base

    // --- Componentes ---

    fn basename(): string                            // "/home/user/file.txt" → "file.txt"
    fn stem(): string                                // "file.txt" → "file"
    fn extension(): optional<string>                 // "file.txt" → "txt"
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

val p = Path("/home/user/docs/file.txt")

val dir = p.dirname()                // Path("/home/user/docs")
val name = p.basename()              // "file.txt"
val ext = p.extension()              // optional("txt")
val stem = p.stem()                  // "file"

val sibling = p.dirname().join("other.txt")
// Path("/home/user/docs/other.txt")

val messy = Path("/home/user/../user/./docs")
val clean = messy.normalize()        // Path("/home/user/docs")

val parts = p.segments()             // ["home", "user", "docs", "file.txt"]
```
