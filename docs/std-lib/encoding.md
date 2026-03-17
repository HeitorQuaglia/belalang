# encoding — Codificação e Decodificação

O módulo `encoding` fornece codificação e decodificação para formatos comuns de transferência de dados.

```
from encoding.base64 use encode, decode
from encoding.hex use encode, decode
from encoding.url use encode, decode
```

---

## encoding.base64

Codificação e decodificação Base64 (RFC 4648).

```
from encoding.base64 use encode, decode, encodeUrl, decodeUrl
```

### Funções

```
fn encode(data: array<int>): string
fn encode(data: string): string                    // codifica UTF-8 como Base64

fn decode(encoded: string): result<array<int>, string>
fn decodeString(encoded: string): result<string, string>   // decodifica como string UTF-8

fn encodeUrl(data: array<int>): string             // Base64url (sem padding, seguro para URL)
fn encodeUrl(data: string): string

fn decodeUrl(encoded: string): result<array<int>, string>
fn decodeUrlString(encoded: string): result<string, string>
```

### Exemplos

```
from encoding.base64 use encode, decodeString
from io use println

encoded = encode("Olá, Bela!")
println(encoded)    // "T2zDoSwgQmVsYSE="

decoded = try decodeString(encoded)
println(decoded)    // "Olá, Bela!"

// Bytes brutos
bytes = [0xFF, 0x00, 0xAB]
b64 = encode(bytes)    // "/QCr"
```

---

## encoding.hex

Codificação e decodificação hexadecimal.

```
from encoding.hex use encode, decode
```

### Funções

```
fn encode(data: array<int>): string                // bytes → string hex (lowercase)
fn encode(data: string): string                    // string UTF-8 → hex
fn encodeUpper(data: array<int>): string           // bytes → string hex (uppercase)

fn decode(hex: string): result<array<int>, string>         // hex → bytes
fn decodeString(hex: string): result<string, string>       // hex → string UTF-8
```

### Exemplos

```
from encoding.hex use encode, decode
from io use println

encoded = encode("Bela")
println(encoded)    // "42656c61"

bytes = try decode("deadbeef")
println(bytes)      // [222, 173, 190, 239]

// Útil para exibir hashes
hash = computeHash(data)    // array<int>
println("SHA-256: ${encode(hash)}")
```

---

## encoding.url

Codificação e decodificação de URLs (percent-encoding, RFC 3986).

```
from encoding.url use encode, decode, encodeComponent, decodeComponent
```

### Funções

```
fn encode(url: string): string                     // codifica URL completa (preserva :, /, ?, #, etc.)
fn decode(encoded: string): result<string, string> // decodifica URL completa

fn encodeComponent(value: string): string          // codifica componente (codifica tudo exceto unreserved chars)
fn decodeComponent(encoded: string): result<string, string>

fn encodeQuery(params: Map<string, string>): string // monta query string "k=v&k2=v2"
fn parseQuery(query: string): result<Map<string, string>, string>
```

### Exemplos

```
from encoding.url use encodeComponent, decodeComponent, encodeQuery, parseQuery
from io use println

// Codificar valor para uso em query string
nome = "João & Maria"
safe = encodeComponent(nome)
println(safe)    // "Jo%C3%A3o%20%26%20Maria"

decoded = try decodeComponent(safe)
println(decoded)    // "João & Maria"

// Montar query string
params = Map([
    ["nome", "João"],
    ["cidade", "São Paulo"],
    ["page", "1"]
])
qs = encodeQuery(params)
println(qs)    // "nome=Jo%C3%A3o&cidade=S%C3%A3o+Paulo&page=1"

// Fazer parse de query string
parsed = try parseQuery("q=bela+lang&page=2")
println(parsed.get("q"))    // "bela lang"
```
