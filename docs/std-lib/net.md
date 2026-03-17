# net — Rede

## Submódulos

| Módulo            | Descrição                                     |
| ----------------- | --------------------------------------------- |
| `net`             | Tipos compartilhados (NetError)               |
| `net.url`         | Parsing e construção de URLs                  |
| `net.http`        | Tipos HTTP compartilhados (Request, Response) |
| `net.http.client` | Cliente HTTP                                  |
| `net.http.server` | Servidor HTTP                                 |
| `net.tcp`         | Conexões TCP                                  |
| `net.udp`         | Datagramas UDP                                |
| `net.dns`         | Resolução de nomes                            |
| `net.tls`         | Conexões TLS sobre TCP                        |
| `net.ip`          | Endereços IP                                  |

---

## net — Tipos Compartilhados

```
enum NetError {
    CONNECTION_REFUSED(string),
    CONNECTION_RESET(string),
    TIMEOUT(string),
    DNS_FAILURE(string),
    TLS_ERROR(string),
    ADDR_IN_USE(string),
    ADDR_NOT_AVAILABLE(string),
    UNEXPECTED(string)
}
```

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
    val port: int                                    // null se ausente
    val path: string                                 // "/api/users"
    val query: string                                // null se ausente
    val fragment: string                             // null se ausente

    // --- Parsing ---

    static fn parse(raw: string): result<Url, string>

    // --- Consultas ---

    fn queryParam(key: string): string              // null se não existe
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

url = try Url.parse("https://example.com:8080/api/users?page=1&limit=10#top")

scheme = url.scheme              // "https"
host = url.host                  // "example.com"
port = url.port                  // 8080
page = url.queryParam("page")   // "1"

next = url
    .withQuery("page", "2")
    .withoutFragment()
    .toString()
// "https://example.com:8080/api/users?page=2&limit=10"
```

---

## net.http — Tipos HTTP

```
from net.http use Method, Headers, Body, Request, Response
```

### Enums

```
enum Method {
    GET,
    POST,
    PUT,
    DELETE,
    PATCH,
    HEAD,
    OPTIONS
}
```

### Classes

```
class Headers {
    fn ()

    fn get(name: string): string                    // null se ausente
    fn getAll(name: string): array<string>
    fn contains(name: string): bool
    fn entries(): array<array<string>>               // [["Content-Type", "application/json"], ...]

    // --- Construção (imutável) ---

    fn withHeader(name: string, value: string): Headers
    fn withoutHeader(name: string): Headers
}

class Body {
    // --- Criação ---

    static fn text(content: string): Body
    static fn bytes(data: array<int>): Body
    static fn empty(): Body

    // --- Leitura ---

    fn text(): result<string, NetError>
    fn bytes(): result<array<int>, NetError>
    fn isEmpty(): bool
}

class Request {
    fn (method: Method, url: Url)

    // --- Propriedades ---

    val method: Method
    val url: Url
    val headers: Headers
    val body: Body

    // --- Construção (imutável) ---

    fn withHeader(name: string, value: string): Request
    fn withBody(body: Body): Request
}

class Response {
    // --- Propriedades ---

    val status: int
    val headers: Headers
    val body: Body

    // --- Criação (server-side) ---

    static fn of(status: int): Response

    // --- Construção (imutável) ---

    fn withStatus(status: int): Response
    fn withHeader(name: string, value: string): Response
    fn withBody(body: Body): Response

    // --- Consultas ---

    fn isOk(): bool                                  // 200-299
    fn isRedirect(): bool                            // 300-399
    fn isClientError(): bool                         // 400-499
    fn isServerError(): bool                         // 500-599
}
```

---

## net.http.client — Cliente HTTP

```
from net.http.client use Client, ClientConfig
```

### Classes

```
class ClientConfig {
    fn ()

    val timeout: int                                 // ms, padrão 30000
    val followRedirects: bool                        // padrão true
    val maxRedirects: int                            // padrão 10

    fn withTimeout(ms: int): ClientConfig
    fn withFollowRedirects(follow: bool): ClientConfig
    fn withMaxRedirects(max: int): ClientConfig
}

class Client {
    fn ()
    fn (config: ClientConfig)

    // --- Envio genérico ---

    async fn send(request: Request): result<Response, NetError>

    // --- Conveniência ---

    async fn get(url: string): result<Response, NetError>
    async fn post(url: string, body: Body): result<Response, NetError>
    async fn put(url: string, body: Body): result<Response, NetError>
    async fn patch(url: string, body: Body): result<Response, NetError>
    async fn delete(url: string): result<Response, NetError>
    async fn head(url: string): result<Response, NetError>
}
```

### Exemplos

```
from net.http use Method, Body, Request, Response
from net.http.client use Client, ClientConfig
from net.url use Url

// GET simples
client = Client()
res = try await client.get("https://api.example.com/users")
if (res.isOk()) {
    println(try res.body.text())
}

// POST com body
body = Body.text("{\"name\": \"Bela\"}")
res = try await client.post("https://api.example.com/users", body)

// Request customizado
url = try Url.parse("https://api.example.com/users")
req = Request(Method.GET, url)
    .withHeader("Authorization", "Bearer token123")
    .withHeader("Accept", "application/json")
res = try await client.send(req)

// Client com configuração
config = ClientConfig()
    .withTimeout(5000)
    .withFollowRedirects(false)
client = Client(config)
res = try await client.get("https://api.example.com/health")
```

---

## net.http.server — Servidor HTTP

```
from net.http.server use Server, ServerConfig, TlsConfig
```

### Tipos

```
type Handler = async fn(req: Request): result<Response, NetError>
```

### Classes

```
class TlsConfig {
    fn (certPath: string, keyPath: string)

    val certPath: string
    val keyPath: string
}

class ServerConfig {
    fn ()

    val host: string                                 // padrão "0.0.0.0"
    val port: int                                    // padrão 8080
    val tls: TlsConfig                                 // null se ausente

    fn withHost(host: string): ServerConfig
    fn withPort(port: int): ServerConfig
    fn withTls(tls: TlsConfig): ServerConfig
}

class Server {
    fn ()
    fn (config: ServerConfig)

    // --- Rotas ---

    fn route(method: Method, path: string, handler: Handler): void
    fn get(path: string, handler: Handler): void
    fn post(path: string, handler: Handler): void
    fn put(path: string, handler: Handler): void
    fn patch(path: string, handler: Handler): void
    fn delete(path: string, handler: Handler): void

    // --- Ciclo de vida ---

    async fn listen(): result<void, NetError>
    fn stop(): result<void, NetError>
}
```

### Exemplos

```
from net.http use Method, Body, Request, Response
from net.http.server use Server, ServerConfig, TlsConfig

// Servidor básico
server = Server()

server.get("/hello", async fn(req: Request): result<Response, NetError> => {
    return result.OK(
        Response.of(200)
            .withHeader("Content-Type", "text/plain")
            .withBody(Body.text("Hello, World!"))
    )
})

server.post("/users", async fn(req: Request): result<Response, NetError> => {
    body = try req.body.text()
    println("Recebido: $body")
    return result.OK(
        Response.of(201)
            .withBody(Body.text("{\"status\": \"created\"}"))
    )
})

try await server.listen()

// Servidor com configuração e TLS
config = ServerConfig()
    .withHost("0.0.0.0")
    .withPort(443)
    .withTls(TlsConfig("cert.pem", "key.pem"))

server = Server(config)
server.get("/", async fn(req: Request): result<Response, NetError> => {
    return result.OK(Response.of(200).withBody(Body.text("Seguro!")))
})
try await server.listen()
```

---

## net.tcp — Conexões TCP

```
from net.tcp use TcpStream, TcpListener
```

Conexões TCP de baixo nível. `TcpStream` implementa `Reader`, `Writer` e `Closer` de `io.stream`.

### Classes

```
class TcpStream implements Reader, Writer, Closer {
    static async fn connect(host: string, port: int): result<TcpStream, NetError>

    val localAddr: string
    val remoteAddr: string

    // --- Reader ---

    async fn read(size: int): result<string, IOError>
    async fn readAll(): result<string, IOError>

    // --- Writer ---

    async fn write(content: string): result<void, IOError>
    async fn flush(): result<void, IOError>

    // --- Closer ---

    fn close(): void
}

class TcpListener {
    static async fn bind(host: string, port: int): result<TcpListener, NetError>

    val addr: string

    async fn accept(): result<TcpStream, NetError>
    fn close(): void
}
```

### Exemplos

```
from net.tcp use TcpStream, TcpListener

// Cliente TCP
stream = try await TcpStream.connect("example.com", 80)
try await stream.write("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n")
response = try await stream.readAll()
stream.close()

// Servidor TCP
listener = try await TcpListener.bind("0.0.0.0", 9000)
loop {
    conn = try await listener.accept()
    data = try await conn.read(1024)
    try await conn.write("Echo: $data")
    conn.close()
}
```

---

## net.udp — Datagramas UDP

```
from net.udp use UdpSocket, Datagram
```

### Classes

```
class Datagram {
    val data: string
    val addr: string
    val port: int
}

class UdpSocket {
    static async fn bind(host: string, port: int): result<UdpSocket, NetError>

    val addr: string

    async fn sendTo(data: string, host: string, port: int): result<int, NetError>
    async fn recv(size: int): result<Datagram, NetError>
    fn close(): void
}
```

### Exemplos

```
from net.udp use UdpSocket

// Enviar datagrama
socket = try await UdpSocket.bind("0.0.0.0", 0)
try await socket.sendTo("ping", "192.168.1.1", 9000)
socket.close()

// Receber datagramas
server = try await UdpSocket.bind("0.0.0.0", 9000)
loop {
    dgram = try await server.recv(1024)
    println("De ${dgram.addr}:${dgram.port} → ${dgram.data}")
    try await server.sendTo("pong", dgram.addr, dgram.port)
}
```

---

## net.dns — Resolução de Nomes

```
from net.dns use resolve, resolveV4, resolveV6, reverseLookup
```

### Funções

```
async fn resolve(hostname: string): result<array<IpAddr>, NetError>
async fn resolveV4(hostname: string): result<array<IpAddr>, NetError>
async fn resolveV6(hostname: string): result<array<IpAddr>, NetError>
async fn reverseLookup(addr: IpAddr): result<string, NetError>
```

### Exemplos

```
from net.dns use resolve, reverseLookup

addrs = try await resolve("example.com")
for (addr in addrs) {
    println(addr.toString())               // "93.184.216.34", "2606:2800:220:1:..."
}

host = try await reverseLookup(addrs[0])
println(host)                              // "example.com"
```

---

## net.tls — Conexões TLS

```
from net.tls use TlsStream, TlsConfig
```

Wrapper TLS sobre `TcpStream`. Implementa `Reader`, `Writer` e `Closer` de `io.stream`.

### Classes

```
class TlsConfig {
    fn ()

    fn withCert(path: string): TlsConfig
    fn withKey(path: string): TlsConfig
    fn withCa(path: string): TlsConfig
    fn withVerify(verify: bool): TlsConfig           // padrão true
}

class TlsStream implements Reader, Writer, Closer {
    static async fn connect(host: string, port: int): result<TlsStream, NetError>
    static async fn wrap(stream: TcpStream, host: string): result<TlsStream, NetError>
    static async fn connectWith(host: string, port: int,
                                config: TlsConfig): result<TlsStream, NetError>

    val localAddr: string
    val remoteAddr: string

    // --- Reader ---

    async fn read(size: int): result<string, IOError>
    async fn readAll(): result<string, IOError>

    // --- Writer ---

    async fn write(content: string): result<void, IOError>
    async fn flush(): result<void, IOError>

    // --- Closer ---

    fn close(): void
}
```

### Exemplos

```
from net.tls use TlsStream
from net.tcp use TcpStream

// Conexão TLS direta
stream = try await TlsStream.connect("example.com", 443)
try await stream.write("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n")
response = try await stream.readAll()
stream.close()

// Upgrade de TcpStream existente
tcp = try await TcpStream.connect("example.com", 443)
tls = try await TlsStream.wrap(tcp, "example.com")
try await tls.write("Hello TLS")
tls.close()
```

---

## net.ip — Endereços IP

```
from net.ip use IpAddr
```

### Classe

```
class IpAddr {
    // --- Parsing ---

    static fn parse(raw: string): result<IpAddr, string>
    static fn v4(a: int, b: int, c: int, d: int): IpAddr

    // --- Propriedades ---

    val raw: string

    // --- Consultas ---

    fn isV4(): bool
    fn isV6(): bool
    fn isLoopback(): bool                            // 127.0.0.0/8 ou ::1
    fn isPrivate(): bool                             // 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
    fn isMulticast(): bool

    // --- Conversão ---

    fn toString(): string
}
```

### Exemplos

```
from net.ip use IpAddr

addr = try IpAddr.parse("192.168.1.1")
println(addr.isV4())                   // true
println(addr.isPrivate())             // true
println(addr.isLoopback())            // false

loopback = try IpAddr.parse("127.0.0.1")
println(loopback.isLoopback())        // true

local = IpAddr.v4(192, 168, 0, 1)
println(local.toString())             // "192.168.0.1"

ipv6 = try IpAddr.parse("::1")
println(ipv6.isV6())                  // true
println(ipv6.isLoopback())           // true
```