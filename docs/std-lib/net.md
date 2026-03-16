# net — Rede

## Submódulos

| Módulo    | Descrição                    |
| --------- | ---------------------------- |
| `net.url` | Parsing e construção de URLs |

---

## net.url — URLs

```
from net.url use Url
```

`Url` representa uma URL parseada. Imutável — métodos `with*` retornam uma nova `Url`.

### Classe

```
class Url {
    // --- Propriedades ---

    val scheme: string                               // "https"
    val host: string                                 // "example.com"
    val port: optional<int>                          // 443
    val path: string                                 // "/api/users"
    val query: optional<string>                      // "page=1&limit=10"
    val fragment: optional<string>                   // "section"

    // --- Parsing ---

    static fn parse(raw: string): result<Url, string>

    // --- Consultas ---

    fn queryParam(key: string): optional<string>
    fn queryParams(): array<array<string>>           // [["page", "1"], ["limit", "10"]]
    fn hasQuery(key: string): bool

    // --- Construção (imutável) ---

    fn withScheme(scheme: string): Url
    fn withHost(host: string): Url
    fn withPort(port: int): Url
    fn withPath(path: string): Url
    fn withQuery(key: string, value: string): Url
    fn withFragment(fragment: string): Url
    fn withoutQuery(key: string): Url
    fn withoutFragment(): Url

    // --- Conversão ---

    fn toString(): string
}
```

### Exemplos

```
from net.url use Url

val url = try Url.parse("https://example.com:8080/api/users?page=1&limit=10#top")

val scheme = url.scheme              // "https"
val host = url.host                  // "example.com"
val port = url.port                  // optional(8080)
val page = url.queryParam("page")   // optional("1")

val next = url
    .withQuery("page", "2")
    .withoutFragment()
    .toString()
// "https://example.com:8080/api/users?page=2&limit=10"
```
