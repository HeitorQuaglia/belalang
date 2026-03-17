# string — Manipulação de Strings

Strings são **imutáveis** e internamente codificadas em **UTF-8**. Conversão de encoding acontece apenas nas fronteiras de IO.

## string

### Indexação

Strings suportam acesso por índice e por range, reusando a sintaxe de `range<int>`.

```
str = "Hello, World"

first = str[0]            // "H"
begin = str[0..4]         // "Hello"         (inclusive)
pairs = str[0..11..2]     // "Hlo ol"        (step 2)
rev   = str[11..0..-1]    // "dlroW ,olleH"  (reverso)
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
name = "  Bela Lang  "
trimmed = name.trim()                    // "Bela Lang"
upper = trimmed.toUpper()                // "BELA LANG"
words = trimmed.split(" ")               // ["Bela", "Lang"]
joined = string.join(words, "-")         // "Bela-Lang"

csv = "a,b,c,d"
cols = csv.split(",")                    // ["a", "b", "c", "d"]
first = csv[0..0]                        // "a"

num = "42"
parsed = try num.toInt()                 // 42

greeting = "Hello"
repeated = greeting.repeat(3)            // "HelloHelloHello"
padded = greeting.padEnd(10, ".")        // "Hello....."
```

### Interpolação

Definida na gramática. Funciona em strings simples e multi-line.

```
name = "World"
msg = "Hello, $name!"                    // "Hello, World!"
expr = "2 + 2 = ${2 + 2}"               // "2 + 2 = 4"

multi = """
    Olá, $name.
    Resultado: ${2 + 2}
"""
```

