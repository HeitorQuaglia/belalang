# Bela

**Uma linguagem embarcável moderna — a simplicidade com o poder que você precisa.**

Bela é uma linguagem de programação projetada para ser **embarcada em aplicações host**, oferecendo um equilíbrio entre facilidade de uso e recursos modernos que linguagens como Lua não oferecem.

```
from net.http.server use Server, Response

server = Server()

server.get("/hello", async fn(req) => {
    name = req.queryParam("name") ?? "mundo"
    return Response.ok("Olá, $name!")
})

try server.listen(8080)
```

---

## Por que Bela?

Lua provou que linguagens embarcáveis têm um espaço enorme — jogos, editores, ferramentas, automação, configuração. Mas quem embarca Lua hoje aceita limitações reais: sem classes, sem sistema de tipos, stdlib mínima, debugging limitado.

Bela resolve isso sem abrir mão da leveza.

|                 | Lua                 | Bela                                           |
| --------------- | ------------------- | ---------------------------------------------- |
| **Tipagem**     | Dinâmica pura       | Dinâmica com tipagem gradual (`fixed<T>`)      |
| **OOP**         | Metatables (manual) | Classes, interfaces, structs, enums nativos    |
| **Erros**       | `pcall` / strings   | `result<T, E>` + `try` + pattern matching      |
| **Memória**     | GC com pausas       | Reference counting — previsível, sem pausas    |
| **Stdlib**      | Mínima              | Completa: HTTP, I/O async, JSON, regex, encoding |
| **Null safety** | Nenhuma             | `?.` safe navigation, `??` coalescing          |
| **Debugging**   | Básico              | WebAPI, LSP, CLI attach remoto                 |
| **Execução**    | Embarcado, CLI      | Embarcado, CLI, REPL                           |

---

## Características

### Tipagem gradual — você escolhe o nível de rigor

```
// Dinâmico por padrão — rápido para prototipar
nome = "Bela"
valor = 42

// Anotações como documentação (warning se violado)
idade: int = 25

// Enforçado em runtime quando importa
porta: fixed<int> = 8080
porta = "oops"              // erro!
```

### Erros são valores, não exceções

```
fn divide(a: float, b: float): result<float, string> {
    if (b == 0) {
        return result.ERROR("divisão por zero")
    }
    return result.OK(a / b)
}

// Três formas de consumir
valor = try divide(10, 3)              // propaga erro automaticamente
valor = divide(10, 0) ?? -1            // fallback
match divide(10, 0) {                  // controle total
    result.OK(v) => println(v),
    result.ERROR(e) => println("erro: $e")
}
```

### Pattern matching expressivo

```
match resposta {
    Response(200, body) => processar(body),
    Response(code, _) if code >= 400 => println("erro HTTP $code"),
    _ => println("resposta inesperada")
}
```

### Structs para dados, classes para comportamento

```
// Structs: imutáveis, stack-allocated, cópia por valor
struct Ponto {
    x: float
    y: float
}

p1 = Ponto { x: 1.0, y: 2.0 }
p2 = p1.with(x: 10.0)                 // novo struct, p1 intacto

// Classes: heap, reference counted, herança + interfaces
class Circulo implements Drawable {
    val centro: Ponto
    val raio: float

    constructor(centro, raio) {
        this.centro = centro
        this.raio = raio
    }

    fn area(): float {
        return math.PI * raio ** 2
    }
}
```

### Async/await nativo

```
from net.http.client use Client

async fn buscarUsuarios() {
    client = Client()
    resp = try await client.get("https://api.example.com/users")
    return try resp.body.text()
}
```

### Null safety sem burocracia

```
// Safe navigation — para a cadeia se encontrar null
cidade = usuario?.endereco?.cidade

// Coalescing — valor padrão
nome = usuario?.nome ?? "Anônimo"
```

---

## Modos de Execução

Bela foi pensada para funcionar em três contextos:

### Execução direta

```bash
bela run app.bela
```

### REPL interativo

```bash
bela repl
>>> 1 + 1
2
>>> from math use sqrt
>>> sqrt(144)
12.0
```

### Embarcado no seu software

```c
// C Host — API de embedding
BelaVM *vm = bela_vm_new();
bela_vm_load_file(vm, "scripts/config.bela");

// Expor funções do host para Bela
bela_register_fn(vm, "getPlayerHP", my_get_hp);

// Executar
bela_vm_call(vm, "onPlayerDamage", damage_amount);
bela_vm_free(vm);
```

---

## Debugging

Bela oferece debugging de primeira classe, não como um afterthought:

- **WebAPI** — inspecione uma VM Bela rodando em qualquer host via interface web
- **LSP** — integração direta com VS Code, Neovim e qualquer editor com suporte LSP
- **CLI attach** — conecte-se a uma VM Bela remota para depuração ao vivo

```bash
# Attach em uma VM rodando no host
bela debug attach --port 9229

# Debug local
bela debug app.bela
```

---

## Stdlib Completa

Diferente de linguagens embarcáveis tradicionais, Bela vem com uma stdlib rica e **modular** — o host escolhe quais módulos disponibilizar.

| Módulo        | Descrição                                           |
| ------------- | --------------------------------------------------- |
| `string`      | Manipulação de strings UTF-8                        |
| `math`        | Funções matemáticas, constantes, números aleatórios |
| `array`       | Tipo built-in `array<T>` — acesso, transformação, busca |
| `collections` | Map, Set, Stack, Queue, Deque                       |
| `io`          | Console, arquivos (sync e async), streams           |
| `os`          | Paths, env, filesystem, processos, signals          |
| `net`         | HTTP client/server, TCP, UDP, DNS, TLS              |
| `time`        | DateTime, Duration, fusos horários                  |
| `concurrency` | Futures, Channels, coordenação async                |
| `regex`       | Expressões regulares com grupos nomeados            |
| `encoding`    | Base64, Hex, URL percent-encoding                   |
| `marshal`     | Serialização JSON, TOML, YAML                       |
| `log`         | Logging estruturado com níveis e handler do host    |

> A stdlib é modular: ao embarcar Bela, o host pode restringir quais módulos estão disponíveis. Um jogo pode expor apenas `math` e `collections`, enquanto uma ferramenta CLI pode expor tudo.

---

## Gerenciamento de Memória

Bela usa **reference counting** (RC), não garbage collection. Isso significa:

- **Previsibilidade** — sem pausas de GC, ideal para jogos e sistemas real-time
- **Determinismo** — destrutores executam imediatamente quando a última referência é liberada
- **Simplicidade** — sem tuning de GC, sem gerações, sem compactação

```
a = Conexao("db.local", 5432)    // RC = 1
b = a                              // RC = 2
// fim do escopo de b              → RC = 1
// fim do escopo de a              → RC = 0 → liberado automaticamente
```

---

## Licença

MIT — veja [LICENSE](LICENSE).
