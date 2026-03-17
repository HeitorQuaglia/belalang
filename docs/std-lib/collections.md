# collections — Coleções

> **Nota:** Belalang usa tipagem dinâmica. As anotações de tipo nas assinaturas
> (incluindo generics como `Map<K, V>`) são mantidas como documentação — não
> são impostas pelo compilador.

## collections

```
from collections use CollectionError, Map, Set, Stack, Queue, Deque
```

### Enums

```
enum CollectionError {
    EMPTY(string),
    NOT_FOUND(string),
    OUT_OF_BOUNDS(string),
    UNEXPECTED(string)
}
```

### Map\<K, V\>

`Map` armazena pares chave-valor. Imutável — métodos de modificação retornam um novo `Map`.

```
class Map<K, V> {
    fn ()
    fn (entries: array<array<dynamic>>)

    // --- Propriedades ---

    val size: int
    val isEmpty: bool

    // --- Criação (static) ---

    static fn of(entries: array<array<dynamic>>): Map    // Map.of([["a", 1], ["b", 2]])

    // --- Acesso ---

    fn get(key: K): V                                    // null se ausente
    fn getOrDefault(key: K, default: V): V
    fn containsKey(key: K): bool
    fn containsValue(value: V): bool

    // --- Modificação (retorna novo Map) ---

    fn set(key: K, value: V): Map
    fn remove(key: K): Map
    fn merge(other: Map): Map                            // other sobrescreve conflitos

    // --- Iteração ---

    fn keys(): array<K>
    fn values(): array<V>
    fn entries(): array<array<dynamic>>                  // [["key", value], ...]
    fn forEach(callback: fn(key: K, value: V): void): void

    // --- Conversão ---

    fn toString(): string
}
```

### Set\<T\>

`Set` armazena valores únicos. Imutável — métodos de modificação retornam um novo `Set`.

```
class Set<T> {
    fn ()
    fn (items: array<T>)

    // --- Propriedades ---

    val size: int
    val isEmpty: bool

    // --- Criação (static) ---

    static fn of(items: array<T>): Set                   // Set.of([1, 2, 3])

    // --- Consultas ---

    fn contains(item: T): bool

    // --- Modificação (retorna novo Set) ---

    fn add(item: T): Set
    fn remove(item: T): Set

    // --- Operações de conjunto ---

    fn union(other: Set): Set
    fn intersection(other: Set): Set
    fn difference(other: Set): Set                       // elementos em this que não estão em other
    fn isSubsetOf(other: Set): bool
    fn isSupersetOf(other: Set): bool

    // --- Iteração ---

    fn toArray(): array<T>
    fn forEach(callback: fn(item: T): void): void

    // --- Conversão ---

    fn toString(): string
}
```

### Stack\<T\>

`Stack` é uma pilha (LIFO). Mutável — operações alteram a instância.

```
class Stack<T> {
    fn ()

    // --- Propriedades ---

    val size: int
    val isEmpty: bool

    // --- Operações ---

    fn push(item: T): void
    fn pop(): result<T, CollectionError>
    fn peek(): result<T, CollectionError>                // retorna sem remover
    fn clear(): void

    // --- Conversão ---

    fn toArray(): array<T>
    fn toString(): string
}
```

### Queue\<T\>

`Queue` é uma fila (FIFO). Mutável — operações alteram a instância.

```
class Queue<T> {
    fn ()

    // --- Propriedades ---

    val size: int
    val isEmpty: bool

    // --- Operações ---

    fn enqueue(item: T): void
    fn dequeue(): result<T, CollectionError>
    fn peek(): result<T, CollectionError>                // retorna sem remover
    fn clear(): void

    // --- Conversão ---

    fn toArray(): array<T>
    fn toString(): string
}
```

### Deque\<T\>

`Deque` é uma fila de duas pontas. Mutável — operações alteram a instância.

```
class Deque<T> {
    fn ()

    // --- Propriedades ---

    val size: int
    val isEmpty: bool

    // --- Operações ---

    fn pushFront(item: T): void
    fn pushBack(item: T): void
    fn popFront(): result<T, CollectionError>
    fn popBack(): result<T, CollectionError>
    fn peekFront(): result<T, CollectionError>
    fn peekBack(): result<T, CollectionError>
    fn clear(): void

    // --- Conversão ---

    fn toArray(): array<T>
    fn toString(): string
}
```

### Exemplos

#### Map

```
from collections use Map

// Criar e usar
m = Map.of([["nome", "Bela"], ["versao", "1.0"]])
nome = m.get("nome")                    // "Bela"
missing = m.get("autor")                // null

m2 = m.set("autor", "Lang Team")
println(m2.containsKey("autor"))        // true
println(m.containsKey("autor"))         // false (imutável)

// Iterar
for (entry in m2.entries()) {
    println("${entry[0]}: ${entry[1]}")
}
```

#### Set

```
from collections use Set

a = Set.of([1, 2, 3, 4])
b = Set.of([3, 4, 5, 6])

union = a.union(b)                       // Set {1, 2, 3, 4, 5, 6}
inter = a.intersection(b)               // Set {3, 4}
diff = a.difference(b)                  // Set {1, 2}

println(a.contains(3))                  // true
println(a.isSubsetOf(union))            // true
```

#### Stack e Queue

```
from collections use Stack, Queue

// Stack (LIFO)
stack = Stack()
stack.push(1)
stack.push(2)
stack.push(3)
top = try stack.pop()                   // 3

// Queue (FIFO)
queue = Queue()
queue.enqueue("a")
queue.enqueue("b")
queue.enqueue("c")
first = try queue.dequeue()             // "a"
```
