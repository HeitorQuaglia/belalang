# regex — Expressões Regulares

## regex

```
from regex use Regex, Match, RegexFlag, RegexError
```

### Enums

```
enum RegexError {
    INVALID_PATTERN(string),
    UNEXPECTED(string)
}

enum RegexFlag {
    CASE_INSENSITIVE,
    MULTILINE,
    DOT_ALL,
    UNICODE
}
```

### Match

```
class Match {

    // --- Propriedades ---

    val value: string                                    // texto correspondente
    val start: int                                       // índice de início no texto
    val end: int                                         // índice de fim no texto
    val groups: array<string>                            // grupos de captura (índice 0 = match completo)

    // --- Grupos nomeados ---

    fn group(name: string): string                       // null se grupo não existe
}
```

### Regex

```
class Regex {
    fn (pattern: string, flags: array<RegexFlag> = [])

    // --- Criação (static) ---

    static fn compile(pattern: string,
                      flags: array<RegexFlag> = []): result<Regex, RegexError>

    // --- Propriedades ---

    val pattern: string
    val flags: array<RegexFlag>

    // --- Teste ---

    fn test(text: string): bool

    // --- Busca ---

    fn find(text: string): Match                         // null se não encontrou
    fn findAll(text: string): array<Match>

    // --- Substituição ---

    fn replace(text: string, replacement: string): string
    fn replaceAll(text: string, replacement: string): string
    fn replaceWith(text: string,
                   callback: fn(match: Match): string): string

    // --- Divisão ---

    fn split(text: string): array<string>
    fn split(text: string, limit: int): array<string>
}
```

### Exemplos

#### Teste e busca

```
from regex use Regex, RegexFlag

// Compilar e testar
re = try Regex.compile("[0-9]+")
println(re.test("abc123"))               // true
println(re.test("abc"))                  // false

// Buscar primeira ocorrência
m = re.find("abc123def456")
println(m.value)                         // "123"
println(m.start)                         // 3
println(m.end)                           // 6

// Buscar todas
matches = re.findAll("abc123def456")
for (m in matches) {
    println(m.value)                     // "123", "456"
}
```

#### Grupos de captura

```
from regex use Regex

re = try Regex.compile("(?<ano>[0-9]{4})-(?<mes>[0-9]{2})-(?<dia>[0-9]{2})")
m = re.find("Data: 2025-07-15")

println(m.value)                         // "2025-07-15"
println(m.groups[1])                     // "2025"
println(m.groups[2])                     // "07"
println(m.group("ano"))                  // "2025"
println(m.group("dia"))                  // "15"
```

#### Substituição

```
from regex use Regex, RegexFlag

re = try Regex.compile("[aeiou]", [RegexFlag.CASE_INSENSITIVE])

result = re.replaceAll("Hello World", "*")
println(result)                          // "H*ll* W*rld"

// Substituição com callback
re2 = try Regex.compile("[0-9]+")
result2 = re2.replaceWith("item1 item2 item3", fn(m: Match): string => {
    num = try m.value.toInt()
    return "${num * 10}"
})
println(result2)                         // "item10 item20 item30"
```

#### Divisão

```
from regex use Regex

re = try Regex.compile("[,;\\s]+")
parts = re.split("a, b; c   d")
println(parts)                           // ["a", "b", "c", "d"]
```
