# concurrency — Primitivas de Concorrência

O módulo `concurrency` fornece as primitivas para coordenar múltiplas operações assíncronas no modelo de **event loop single-thread** de Bela.

> Para entender o modelo de concorrência (async/await, Future<T>, integração com host), ver seção 23 da gramática.

```
from concurrency use all, race, any, timeout, Channel
```

---

## Future\<T\>

`Future<T>` é o tipo retornado implicitamente por `async fn`. Representa um valor que será produzido no futuro pelo event loop.

```
async fn buscar(url: string): string {
    // ...
}

f = buscar("https://api.exemplo.com")    // f: Future<string>
resultado = await f                       // suspende até resolver
```

`Future<T>` não é instanciado diretamente — é produzido pelo runtime quando uma `async fn` é chamada.

---

## Coordenação de Futures

### `all`

Aguarda **todas** as futures. Retorna um array com os resultados na mesma ordem. Se qualquer future falhar com `result.ERROR`, retorna o primeiro erro.

```
fn all(tasks: array<Future<dynamic>>): Future<array<dynamic>>
```

```
from concurrency use all
from net.http.client use Client

client = Client()

results = await all([
    client.get("/users"),
    client.get("/products"),
    client.get("/orders")
])

users    = results[0]
products = results[1]
orders   = results[2]
```

### `race`

Retorna o resultado da **primeira** future a completar (com sucesso ou erro). As demais são descartadas.

```
fn race(tasks: array<Future<dynamic>>): Future<dynamic>
```

```
from concurrency use race

winner = await race([
    fetchFromPrimary(),
    fetchFromReplica()
])
```

### `any`

Retorna o resultado da **primeira future que completar com sucesso**. Ignora erros das demais. Se todas falharem, retorna `result.ERROR` com array de todos os erros.

```
fn any(tasks: array<Future<dynamic>>): Future<result<dynamic, array<dynamic>>>
```

```
from concurrency use any

res = await any([
    tryServer("us-east"),
    tryServer("eu-west"),
    tryServer("ap-south")
])

match res {
    result.OK(data) => processar(data),
    result.ERROR(erros) => println("todos os servidores falharam")
}
```

### `timeout`

Envolve uma future com um limite de tempo. Retorna `result.ERROR("timeout")` se o tempo esgota antes da future completar.

```
fn timeout(ms: int, task: Future<dynamic>): Future<result<dynamic, string>>
```

```
from concurrency use timeout

res = await timeout(5000, fetchData())    // máximo 5 segundos

match res {
    result.OK(data) => processar(data),
    result.ERROR("timeout") => println("operação muito lenta"),
    result.ERROR(e) => println("erro: $e")
}
```

---

## Channel\<T\>

`Channel<T>` é um canal de comunicação entre coroutines. Baseado no modelo produtor/consumidor. Canais são **assíncronos** — `send` e `receive` são operações await-ables.

```
class Channel<T> {
    constructor(capacity: int = 0)    // 0 = unbuffered (síncrono), N = buffered

    async fn send(value: T): void         // bloqueia se canal cheio (buffered)
    async fn receive(): result<T, string> // bloqueia até ter valor, ou ERROR se fechado
    fn close(): void                       // fecha o canal — receive retornará ERROR
    val isClosed: bool
    val length: int                        // itens no buffer (buffered apenas)
}
```

### Exemplo: Produtor/Consumidor

```
from concurrency use Channel
from io use println

async fn produtor(ch: Channel<int>) {
    for (i in 1..5) {
        await ch.send(i)
        println("enviou $i")
    }
    ch.close()
}

async fn consumidor(ch: Channel<int>) {
    loop {
        msg = await ch.receive()
        match msg {
            result.OK(v) => println("recebeu $v"),
            result.ERROR(_) => break    // canal fechado
        }
    }
}

ch = Channel(3)    // buffered com capacidade 3
produtor(ch)
consumidor(ch)
```

### Exemplo: Fan-out com múltiplos workers

```
from concurrency use Channel, all

async fn worker(id: int, jobs: Channel<int>, results: Channel<int>) {
    loop {
        job = await jobs.receive()
        match job {
            result.OK(n) => await results.send(n * n),
            result.ERROR(_) => break
        }
    }
}

jobs    = Channel(10)
results = Channel(10)

// Iniciar 3 workers
all([worker(1, jobs, results), worker(2, jobs, results), worker(3, jobs, results)])

// Enviar trabalho
for (i in 1..9) { await jobs.send(i) }
jobs.close()
```

---

## `await_sync` — Modo Bloqueante

Para uso em contexto síncrono (fora de `async fn`). **Bloqueia a thread atual** até a future completar. Evitar em aplicações com event loop próprio.

```
fn await_sync<T>(task: Future<T>): T
```

```
from concurrency use await_sync

// Útil em scripts simples sem async main
data = await_sync(fetchData())
println(data)
```
