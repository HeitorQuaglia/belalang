# Linguagem — Lacunas a Definir

## 1. Sistema de Módulos: Visibilidade e Exports

A gramática define `from X use Y` (imports), mas **não define o outro lado**:

- Como um módulo declara o que é público? `export`? `pub`? Tudo é público por padrão?
- `access_modifier ::= "public" | "private" | "protected"` existe, mas só aparece no contexto de classes. E em top-level declarations (funções, variáveis, enums, structs)?
- Mapeamento módulo → arquivo/diretório não está definido. `from io.file use File` implica `io/file.bela`? Uma pasta `io/` com `file.bela`?

### Decisões pendentes

- Default público vs. default privado para top-level declarations
- Keyword explícita (`export`, `pub`) ou inferência por convenção
- Estrutura de diretórios e resolução de módulos

---

## 2. `array<T>` — Tipo Built-in Sem Documentação

`array` é usado em **todos** os módulos da stdlib, mas não existe documentação dos seus métodos.

### Métodos esperados (mínimo)

- Acesso: `length`, `isEmpty`, `get`, `first`, `last`, `contains`, `indexOf`
- Transformação: `map`, `filter`, `reduce`, `forEach`, `flatMap`, `flatten`
- Ordenação: `sort`, `sortBy`, `reverse`
- Modificação: `push`, `pop`, `shift`, `unshift`, `insert`, `removeAt`, `slice`, `concat`
- Conversão: `toString`, `toSet`
- Utilitários: `zip`, `enumerate`, `find`, `findIndex`, `any`, `all`, `count`

### Decisão pendente

- `array` é mutável ou imutável? Se imutável, métodos de "modificação" retornam novo array (como `Map` e `Set`). Se mutável (como `Stack` e `Queue`), modifica in-place.

---

## 3. `optional<T>` — Aparece Mas Nunca é Definido

Em `string.md`:

```
fn indexOf(sub: string): optional<int>
fn lastIndexOf(sub: string): optional<int>
```

Mas `optional` não aparece na gramática nem em nenhum outro lugar.

### Decisões pendentes

- É um tipo built-in? É açúcar sintático para `T | null`?
- Qual a relação com `result`? `result` é para erros, `optional` é para ausência?
- Tem métodos próprios (`unwrap`, `map`, `orElse`)?
- Ou simplesmente não existe e esses métodos deveriam retornar `null`?

---

## 4. `panic` — Usado Mas Não Definido

Aparece no exemplo de factory na gramática:

```
panic("porta inválida")
```

Mas não há definição formal.

### Decisões pendentes

- É uma função built-in?
- É recuperável (catch/recover) ou encerra o programa?
- Quando usar `panic` vs. `result.ERROR`?
- Diferença semântica: `panic` = bug do programador, `result.ERROR` = erro esperado?

---

## 5. Modelo de Concorrência

`async`/`await` estão na gramática e na stdlib (`AsyncFile`, `Client`, `Server`), mas falta definir a semântica.

### Decisões pendentes

- Qual é o modelo? Event loop single-thread? Coroutines? Green threads?
- Existe um tipo `Future`/`Promise` explícito?
- Execução paralela de múltiplas operações async (equivalente a `Promise.all`, `select`, `race`)
- Primitivas de sincronização: channels, mutexes, semáforos?
- Cancelamento de operações async?

---

## 6. Interfaces Built-in

`Comparable` aparece como exemplo de generic bound mas nunca é definida.

### Interfaces a definir

| Interface | Propósito |
|---|---|
| `Comparable` | Ordenação e operadores `<`, `>`, `<=`, `>=` |
| `Equatable` | Igualdade e operador `==` |
| `Hashable` | Permite uso como chave em `Map` e `Set` |
| `Iterable` | Permite uso em `for...in` |
| `Stringable` | Define `toString()` |

### Decisões pendentes

- Quais são obrigatórias vs. opcionais?
- Tipos primitivos implementam quais interfaces implicitamente?
- O compilador gera implementações automáticas para structs?

---

## 7. Protocolo de Iteração

`for (x in expr)` existe, mas:

- Qualquer tipo pode ser iterável? Qual interface implementar?
- `range` é iterável, `array` é iterável — mas e classes customizadas?
- Existe um tipo `Iterator` separado de `Iterable`?
- Lazy iteration (geradores, `yield`)?

---

## 8. Conversão de Tipos / Casting

Não há mecanismo documentado para casting ou type checking em runtime.

### Decisões pendentes

- Operador `is` para type checking? (`if (x is string) { ... }`)
- Operador `as` para casting? (`x as int`)
- `typeof(x)` retorna string ou tipo?
- Smart casting (após `is`, a variável assume o tipo verificado)?

---

## 9. Closures e Captura de Variáveis

Lambdas estão definidas na gramática, mas a semântica de captura não está.

### Decisões pendentes

- Captura por valor ou por referência?
- `outer` funciona dentro de lambdas para modificar variáveis do escopo externo?
- Variáveis capturadas mantêm o objeto vivo (interação com reference counting)?

---

## 10. Prelude / Builtins Implícitos

Vários exemplos usam `println` sem import. O que está disponível globalmente sem `from X use Y`?

### Candidatos a prelude

- I/O básico: `print`, `println`, `eprint`, `eprintln`
- Erro fatal: `panic`
- Tipos: `array`, `string`, `int`, `float`, `bool`, `void`, `null`, `dynamic`
- Inspeção: `typeof`
- Result: `result.OK`, `result.ERROR`

### Decisão pendente

- O prelude é um módulo implícito ou são funções do compilador?
- O usuário pode fazer shadowing de nomes do prelude?
