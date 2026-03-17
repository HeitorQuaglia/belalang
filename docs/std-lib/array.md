# array — Tipo Built-in

`array<T>` é o tipo sequencial mutável fundamental de Bela. Literais de array usam colchetes. O tipo é built-in — não requer import.

```
nums = [1, 2, 3, 4, 5]
words = ["bela", "lang"]
mixed = [1, "dois", true]     // array<dynamic>
empty: array<int> = []
```

`array` implementa `Iterable<T>` — pode ser usado em `for...in` diretamente.

---

## Propriedades

```
val length: int       // número de elementos
val isEmpty: bool     // true se length == 0
```

---

## Métodos — Acesso

```
fn get(index: int): T                          // acesso por índice (erro se fora dos limites)
fn first(): result<T, string>                  // primeiro elemento
fn last(): result<T, string>                   // último elemento
fn contains(value: T): bool
fn indexOf(value: T): result<int, string>      // índice da primeira ocorrência
fn lastIndexOf(value: T): result<int, string>  // índice da última ocorrência
```

> Acesso direto via `arr[i]` também é suportado. Índice negativo não é válido — use `arr[arr.length - 1]` para o último elemento.

---

## Métodos — Transformação (retornam novo array)

```
fn map<U>(f: fn(T): U): array<U>
fn filter(f: fn(T): bool): array<T>
fn reduce<U>(initial: U, f: fn(U, T): U): U
fn forEach(f: fn(T): void): void
fn flatMap<U>(f: fn(T): array<U>): array<U>
fn flatten(): array<dynamic>               // array<array<T>> → array<T>
fn reversed(): array<T>                    // cópia em ordem inversa
fn sorted(): array<T>                      // cópia ordenada (T deve implementar Comparable)
fn sortedBy<K: Comparable>(key: fn(T): K): array<T>
fn distinct(): array<T>                    // remove duplicatas (usa .equals())
fn zip<U>(other: array<U>): array<array<dynamic>>
fn enumerate(): array<array<dynamic>>      // [[0, v0], [1, v1], ...]
fn slice(start: int, end: int): array<T>   // subarray [start..end] (end exclusivo)
fn concat(other: array<T>): array<T>       // novo array concatenado
```

---

## Métodos — Modificação (in-place, retornam void)

```
fn push(value: T): void               // adiciona ao final
fn pop(): result<T, string>           // remove e retorna o último
fn shift(): result<T, string>         // remove e retorna o primeiro
fn unshift(value: T): void            // adiciona ao início
fn insert(index: int, value: T): void // insere na posição
fn removeAt(index: int): T            // remove e retorna o elemento na posição
fn reverse(): void                    // inverte in-place
fn sort(): void                       // ordena in-place (T deve implementar Comparable)
fn sortBy<K: Comparable>(key: fn(T): K): void
fn clear(): void                      // remove todos os elementos
fn splice(start: int, deleteCount: int, ...items: T): array<T>
                                      // remove e insere; retorna removidos
```

---

## Métodos — Busca e Predicados

```
fn find(f: fn(T): bool): result<T, string>
fn findIndex(f: fn(T): bool): result<int, string>
fn any(f: fn(T): bool): bool
fn all(f: fn(T): bool): bool
fn count(f: fn(T): bool): int          // contagem de elementos que satisfazem o predicado
```

---

## Métodos — Conversão

```
fn toString(): string                  // serializa para JSON (ex: "[1,2,3]")
fn toSet(): Set<T>                     // converte para Set (duplicatas removidas)
```

---

## Exemplos

```
from io use println

nums = [3, 1, 4, 1, 5, 9, 2, 6]

// Transformação
doubles = nums.map(fn(n) => n * 2)           // [6,2,8,2,10,18,4,12]
evens = nums.filter(fn(n) => n % 2 == 0)     // [4,2,6]
sum = nums.reduce(0, fn(acc, n) => acc + n)  // 31

// Ordenação
sorted = nums.sorted()                        // [1,1,2,3,4,5,6,9] — cópia
nums.sort()                                   // modifica nums in-place

// Busca
found = nums.find(fn(n) => n > 5)
match found {
    result.OK(v) => println("encontrado: $v"),
    result.ERROR(_) => println("não encontrado")
}

// Modificação
nums.push(7)
removed = nums.pop()     // result.OK(7)

// Iteração
for (n in nums) {
    println(n)
}

// Enumerate
for (pair in nums.enumerate()) {
    println("${pair[0]}: ${pair[1]}")
}

// Encadeamento
result = [1, 2, 3, 4, 5]
    .filter(fn(n) => n % 2 != 0)   // [1, 3, 5]
    .map(fn(n) => n * n)            // [1, 9, 25]
    .reduce(0, fn(acc, n) => acc + n) // 35
```
