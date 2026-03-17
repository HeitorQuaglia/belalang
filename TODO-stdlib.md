# Stdlib — Módulos Ausentes

## Prioridade Alta

### array — Métodos do Tipo `array<T>`

O tipo mais usado na linguagem e na stdlib, sem documentação alguma.

Deve cobrir:

- Propriedades: `length`, `isEmpty`
- Acesso: `get`, `first`, `last`, `contains`, `indexOf`, `lastIndexOf`
- Transformação: `map`, `filter`, `reduce`, `forEach`, `flatMap`, `flatten`
- Ordenação: `sort`, `sortBy`, `reverse`, `reversed`
- Modificação: `push`, `pop`, `shift`, `unshift`, `insert`, `removeAt`
- Fatiamento: `slice`, `concat`, `splice`
- Utilitários: `zip`, `enumerate`, `find`, `findIndex`, `any`, `all`, `count`, `distinct`
- Conversão: `toString`, `toSet`

> **Decisão pendente:** mutável (como `Stack`/`Queue`) ou imutável (como `Map`/`Set`)?

---

### concurrency — Primitivas de Concorrência

A linguagem já tem `async`/`await` mas não há primitivas para coordenar múltiplas operações.

Deve cobrir:

- `all(tasks: array<Future>)` — aguarda todas
- `race(tasks: array<Future>)` — aguarda a primeira
- `any(tasks: array<Future>)` — aguarda a primeira que não falha
- `Channel<T>` — comunicação entre coroutines
- `Mutex` — exclusão mútua (se houver paralelismo real)
- `Semaphore` — controle de concorrência
- `timeout(ms: int, task: Future)` — cancela após timeout

---

## Prioridade Média

### crypto — Criptografia e Hashing

| Submódulo | Conteúdo |
|---|---|
| `crypto.hash` | SHA-256, SHA-512, MD5, BLAKE2 |
| `crypto.hmac` | HMAC com algoritmos de hash |
| `crypto.random` | Bytes aleatórios criptograficamente seguros |
| `crypto.aes` | Cifração simétrica AES |

---

### encoding — Codificação

| Submódulo | Conteúdo |
|---|---|
| `encoding.base64` | Encode/decode Base64 |
| `encoding.hex` | Encode/decode hexadecimal |
| `encoding.url` | Encode/decode URL (percent-encoding) |

---

### testing — Framework de Testes

Deve cobrir:

- `test(name, callback)` — define um teste
- `describe(name, callback)` — agrupa testes
- `assert(condition)` — asserção básica
- `assertEqual(expected, actual)` — igualdade
- `assertNotEqual(a, b)`
- `assertThrows(callback)` — espera panic
- `assertError(result)` — espera `result.ERROR`
- `assertOk(result)` — espera `result.OK`
- Runner via CLI: `bela test`

---

### log — Logging

Deve cobrir:

- Níveis: `debug`, `info`, `warn`, `error`
- Configuração de nível mínimo
- Formatação customizável
- Output para stdout/stderr/arquivo
- Contexto estruturado (key-value)

---

## Prioridade Baixa

### fmt — Formatação Avançada

Se a interpolação de strings (`"$var"` / `"${expr}"`) não for suficiente para todos os casos:

- Formatação numérica: casas decimais, separadores de milhar
- Padding e alinhamento
- Formatação de datas (pode ficar em `time`)

> **Nota:** Talvez a interpolação cubra tudo e este módulo não seja necessário.

---

### compress — Compressão

| Submódulo | Conteúdo |
|---|---|
| `compress.gzip` | Compressão/descompressão gzip |
| `compress.zlib` | Compressão/descompressão zlib |
| `compress.zip` | Leitura/escrita de arquivos zip |

---

### uuid — Identificadores Únicos

- `uuid.v4()` — UUID aleatório
- `uuid.v7()` — UUID com timestamp
- `uuid.parse(raw)` — parse de string
- `uuid.isValid(raw)` — validação

---

## Resumo

| Módulo | Prioridade | Motivo |
|---|---|---|
| **array** | Alta | Tipo mais usado, zero documentação |
| **concurrency** | Alta | async/await existe mas sem coordenação |
| **crypto** | Média | Essencial para apps reais |
| **encoding** | Média | Base64/hex são comuns |
| **testing** | Média | Sem testes não há ecossistema |
| **log** | Média | Básico para qualquer app |
| **fmt** | Baixa | Interpolação pode ser suficiente |
| **compress** | Baixa | Nicho |
| **uuid** | Baixa | Nicho |
