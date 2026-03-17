# time — Tempo

> **Nota:** Belalang usa tipagem dinâmica. As anotações de tipo nas assinaturas
> (incluindo `result<T, E>`) são mantidas como documentação — não são impostas
> pelo compilador.

## Submódulos

| Módulo          | Descrição                                |
| --------------- | ---------------------------------------- |
| `time`          | Tipos compartilhados e funções básicas   |
| `time.duration` | Representação de durações                |
| `time.zone`     | Fusos horários                           |

---

## time — Tipos Compartilhados e Funções

```
from time use DateTime, TimeError, Weekday, Month, now, sleep, sleepAsync, timestamp
```

### Enums

```
enum TimeError {
    INVALID_FORMAT(string),
    INVALID_DATE(string),
    INVALID_ZONE(string),
    OVERFLOW(string),
    UNEXPECTED(string)
}

enum Weekday {
    MONDAY,
    TUESDAY,
    WEDNESDAY,
    THURSDAY,
    FRIDAY,
    SATURDAY,
    SUNDAY
}

enum Month {
    JANUARY,
    FEBRUARY,
    MARCH,
    APRIL,
    MAY,
    JUNE,
    JULY,
    AUGUST,
    SEPTEMBER,
    OCTOBER,
    NOVEMBER,
    DECEMBER
}
```

### Funções

```
fn now(): DateTime                                       // data/hora atual em UTC
fn timestamp(): int                                      // unix timestamp em segundos
fn sleep(ms: int): void                                  // pausa a execução
async fn sleepAsync(ms: int): void                       // pausa sem bloquear
```

### DateTime

`DateTime` representa um instante no tempo. Imutável — métodos `with*` retornam um novo `DateTime`.

```
class DateTime {

    // --- Criação (static) ---

    static fn of(year: int, month: int, day: int,
                 hour: int = 0, minute: int = 0, second: int = 0,
                 ms: int = 0): result<DateTime, TimeError>

    static fn parse(text: string,
                    format: string = "yyyy-MM-dd'T'HH:mm:ss"): result<DateTime, TimeError>

    static fn fromTimestamp(seconds: int): DateTime

    // --- Propriedades ---

    val year: int
    val month: int                                       // 1-12
    val day: int                                         // 1-31
    val hour: int                                        // 0-23
    val minute: int                                      // 0-59
    val second: int                                      // 0-59
    val millisecond: int                                 // 0-999
    val weekday: Weekday
    val dayOfYear: int                                   // 1-366
    val timestamp: int                                   // unix timestamp em segundos

    // --- Construção (imutável) ---

    fn withYear(year: int): result<DateTime, TimeError>
    fn withMonth(month: int): result<DateTime, TimeError>
    fn withDay(day: int): result<DateTime, TimeError>
    fn withHour(hour: int): result<DateTime, TimeError>
    fn withMinute(minute: int): result<DateTime, TimeError>
    fn withSecond(second: int): result<DateTime, TimeError>
    fn withZone(zone: Zone): DateTime

    // --- Aritmética ---

    fn add(duration: Duration): DateTime
    fn subtract(duration: Duration): DateTime
    fn diff(other: DateTime): Duration                   // diferença absoluta

    // --- Consultas ---

    fn isBefore(other: DateTime): bool
    fn isAfter(other: DateTime): bool
    fn isEqual(other: DateTime): bool

    // --- Formatação ---

    fn format(pattern: string = "yyyy-MM-dd'T'HH:mm:ss"): string
    fn toIso(): string                                   // ISO 8601
    fn toString(): string
}
```

### Exemplos

#### Data/hora atual

```
from time use DateTime, now, sleep, timestamp

agora = now()
println(agora.format())                  // "2025-07-15T14:30:00"
println(agora.year)                      // 2025
println(agora.weekday)                   // Weekday.TUESDAY
```

#### Criar data específica

```
from time use DateTime

natal = try DateTime.of(2025, 12, 25)
println(natal.format("dd/MM/yyyy"))      // "25/12/2025"
```

#### Parse

```
from time use DateTime

dt = try DateTime.parse("2025-07-15T10:30:00")
println(dt.hour)                         // 10
```

#### Aritmética com Duration

```
from time use DateTime, now
from time.duration use Duration

agora = now()
amanha = agora.add(Duration.hours(24))
diff = amanha.diff(agora)
println(diff.toHours())                  // 24.0
```

#### Comparação

```
from time use DateTime, now

agora = now()
natal = try DateTime.of(2025, 12, 25)
println(natal.isAfter(agora))            // true
```

#### Timestamp

```
from time use DateTime, timestamp

ts = timestamp()                         // 1752588600
dt = DateTime.fromTimestamp(ts)
println(dt.toIso())                      // "2025-07-15T14:30:00Z"
```

#### Sleep

```
from time use sleep

sleep(1000)                              // pausa 1 segundo
```

---

## time.duration — Durações

```
from time.duration use Duration
```

`Duration` representa um intervalo de tempo. Imutável.

### Classe

```
class Duration {

    // --- Criação (static) ---

    static fn milliseconds(ms: int): Duration
    static fn seconds(s: int): Duration
    static fn minutes(m: int): Duration
    static fn hours(h: int): Duration
    static fn days(d: int): Duration

    // --- Propriedades ---

    val totalMilliseconds: int

    // --- Conversão ---

    fn toMilliseconds(): int
    fn toSeconds(): float
    fn toMinutes(): float
    fn toHours(): float
    fn toDays(): float

    // --- Aritmética ---

    fn add(other: Duration): Duration
    fn subtract(other: Duration): Duration
    fn multiply(factor: int): Duration

    // --- Consultas ---

    fn isZero(): bool
    fn isNegative(): bool

    // --- Formatação ---

    fn toString(): string                                // "2h 30m 15s"
}
```

### Exemplos

```
from time.duration use Duration

// Criar durações
d1 = Duration.hours(2)
d2 = Duration.minutes(30)
d3 = d1.add(d2)
println(d3.toString())                   // "2h 30m 0s"
println(d3.toMinutes())                  // 150.0
println(d3.toMilliseconds())            // 9000000

// Operações
dobro = d1.multiply(2)
println(dobro.toHours())                 // 4.0

// Consultas
zero = Duration.seconds(0)
println(zero.isZero())                   // true
```

---

## time.zone — Fusos Horários

```
from time.zone use Zone
```

### Classe

```
class Zone {

    // --- Criação (static) ---

    static fn utc(): Zone
    static fn local(): Zone                              // fuso do sistema
    static fn of(name: string): result<Zone, TimeError>  // "America/Sao_Paulo", "US/Eastern"

    // --- Propriedades ---

    val name: string                                     // "America/Sao_Paulo"
    val offset: int                                      // offset em minutos (-180 para BRT)

    // --- Conversão ---

    fn toString(): string                                // "America/Sao_Paulo (-03:00)"
}
```

### Exemplos

```
from time use DateTime, now
from time.zone use Zone

// Converter entre fusos
utc = now()
sp = try Zone.of("America/Sao_Paulo")
local = utc.withZone(sp)
println(local.format("HH:mm"))          // "11:30" (se UTC for 14:30)

// Informações do fuso
println(sp.name)                         // "America/Sao_Paulo"
println(sp.offset)                       // -180 (minutos)

// Fuso do sistema
sistema = Zone.local()
println(sistema.toString())              // "America/Sao_Paulo (-03:00)"
```
