# string — Manipulação de Strings

Strings são **imutáveis** e internamente codificadas em **UTF-8**. Conversão de encoding acontece apenas nas fronteiras de IO.

## string

### Indexação

Strings suportam acesso por índice e por range, reusando a sintaxe de `range<int>`.

```
val str = "Hello, World"

val first = str[0]            // "H"
val begin = str[0..4]         // "Hello"         (inclusive)
val pairs = str[0..11..2]     // "Hlo ol"        (step 2)
val rev   = str[11..0..-1]    // "dlroW ,olleH"  (reverso)
```

- `str[i]` retorna uma `string` de tamanho 1.
- `str[range]` retorna uma substring seguindo as regras de `range`.
- Índice fora dos limites é erro de runtime.

> ⚠️ Segue as mesmas regras de `range`: `str[11..0]` sem step negativo é erro. Use `str[11..0..-1]` para reverso.

### Propriedades

```
val length: int
val isEmpty: bool
```

### Métodos — Acesso

```
fn indexOf(sub: string): optional<int>
fn lastIndexOf(sub: string): optional<int>
fn contains(sub: string): bool
fn startsWith(prefix: string): bool
fn endsWith(suffix: string): bool
```

### Métodos — Transformação

```
fn toUpper(): string
fn toLower(): string
fn trim(): string
fn trimStart(): string
fn trimEnd(): string
fn replace(old: string, new: string): string
fn replaceFirst(old: string, new: string): string
fn repeat(count: int): string
fn reverse(): string
fn padStart(length: int, pad: string = " "): string
fn padEnd(length: int, pad: string = " "): string
fn chars(): array<string>
```

### Métodos — Divisão e Junção

```
fn split(delimiter: string): array<string>
static fn join(parts: array<string>, separator: string = ""): string
```

### Métodos — Conversão

```
fn toInt(): result<int, string>
fn toFloat(): result<float, string>
fn toBytes(): array<int>
static fn fromBytes(data: array<int>): result<string, string>
```

### Exemplos

```
val name = "  Bela Lang  "
val trimmed = name.trim()                    // "Bela Lang"
val upper = trimmed.toUpper()                // "BELA LANG"
val words = trimmed.split(" ")               // ["Bela", "Lang"]
val joined = string.join(words, "-")         // "Bela-Lang"

val csv = "a,b,c,d"
val cols = csv.split(",")                    // ["a", "b", "c", "d"]
val first = csv[0..0]                        // "a"

val num = "42"
val parsed = try num.toInt()                 // 42

val greeting = "Hello"
val repeated = greeting.repeat(3)            // "HelloHelloHello"
val padded = greeting.padEnd(10, ".")        // "Hello....."
```

### Interpolação

Definida na gramática. Funciona em strings simples e multi-line.

```
val name = "World"
val msg = "Hello, $name!"                    // "Hello, World!"
val expr = "2 + 2 = ${2 + 2}"               // "2 + 2 = 4"

val multi = """
    Olá, $name.
    Resultado: ${2 + 2}
"""
```

