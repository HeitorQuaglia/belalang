# marshall — Serialização

> **Nota:** Belalang usa tipagem dinâmica. As anotações de tipo nas assinaturas
> (incluindo `result<T, E>`) são mantidas como documentação — não são impostas
> pelo compilador.

## Submódulos

| Módulo          | Descrição                                |
| --------------- | ---------------------------------------- |
| `marshall`      | Tipos compartilhados (MarshallError)     |
| `marshall.json` | Serialização e desserialização JSON      |
| `marshall.toml` | Serialização e desserialização TOML      |
| `marshall.yaml` | Serialização e desserialização YAML      |

---

## marshall — Tipos Compartilhados

```
enum MarshallError {
    INVALID_SYNTAX(string),
    INVALID_TYPE(string),
    MISSING_FIELD(string),
    UNEXPECTED(string)
}
```

---

## marshall.json

```
from marshall.json use encode, decode, encodeFormatted
```

### Funções

```
fn encode(value: dynamic): result<string, MarshallError>
fn encodeFormatted(value: dynamic, indent: int = 2): result<string, MarshallError>
fn decode(source: string): result<dynamic, MarshallError>
```

`encode` produz JSON compacto. `encodeFormatted` produz JSON indentado para leitura humana.

Mapeamento de tipos:

| Belalang        | JSON        |
| --------------- | ----------- |
| `int`, `float`  | number      |
| `string`        | string      |
| `bool`          | boolean     |
| `null`          | null        |
| `array`         | array       |
| `Map`           | object      |

### Exemplos

```
from marshall.json use encode, decode, encodeFormatted
from collections use Map

// Encode
data = Map.of([
    ["nome", "Bela"],
    ["versao", 1],
    ["tags", ["lang", "nova"]]
])
json = try encode(data)
// "{\"nome\":\"Bela\",\"versao\":1,\"tags\":[\"lang\",\"nova\"]}"

pretty = try encodeFormatted(data)
// {
//   "nome": "Bela",
//   "versao": 1,
//   "tags": ["lang", "nova"]
// }

// Decode
parsed = try decode("{\"ativo\": true, \"contagem\": 42}")
ativo = parsed.get("ativo")             // true
contagem = parsed.get("contagem")       // 42

// Round-trip
original = Map.of([["x", 1], ["y", 2]])
json = try encode(original)
restored = try decode(json)
println(restored.get("x"))              // 1
```

---

## marshall.toml

```
from marshall.toml use encode, decode
```

### Funções

```
fn encode(value: dynamic): result<string, MarshallError>
fn decode(source: string): result<dynamic, MarshallError>
```

### Exemplos

```
from marshall.toml use encode, decode
from collections use Map

tomlStr = """
[servidor]
host = "localhost"
porta = 8080
debug = true
"""

config = try decode(tomlStr)
servidor = config.get("servidor")
println(servidor.get("host"))            // "localhost"
println(servidor.get("porta"))           // 8080

// Encode
data = Map.of([
    ["titulo", "Meu Projeto"],
    ["versao", "0.1.0"]
])
output = try encode(data)
// titulo = "Meu Projeto"
// versao = "0.1.0"
```

---

## marshall.yaml

```
from marshall.yaml use encode, decode
```

### Funções

```
fn encode(value: dynamic): result<string, MarshallError>
fn decode(source: string): result<dynamic, MarshallError>
```

### Exemplos

```
from marshall.yaml use encode, decode
from collections use Map

yamlStr = """
servidor:
  host: localhost
  porta: 8080
  debug: true
"""

config = try decode(yamlStr)
servidor = config.get("servidor")
println(servidor.get("host"))            // "localhost"

// Encode
data = Map.of([
    ["nome", "Bela"],
    ["ativo", true]
])
output = try encode(data)
// nome: Bela
// ativo: true
```
