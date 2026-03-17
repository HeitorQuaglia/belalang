# math — Matemática

## Submódulos

| Módulo        | Descrição                            |
| ------------- | ------------------------------------ |
| `math`        | Constantes e funções matemáticas     |
| `math.random` | Geração de números aleatórios        |

---

## math — Constantes e Funções

```
from math use PI, E, INF, NAN, abs, ceil, floor, round, min, max, pow, sqrt,
             log, log2, log10, sin, cos, tan, asin, acos, atan, atan2,
             clamp, lerp, MathError
```

### Enums

```
enum MathError {
    DOMAIN(string),
    OVERFLOW(string),
    DIVISION_BY_ZERO(string),
    UNEXPECTED(string)
}
```

### Constantes

```
const PI: float                          // 3.141592653589793
const E: float                           // 2.718281828459045
const INF: float                         // infinito positivo
const NEG_INF: float                     // infinito negativo
const NAN: float                         // not a number
const MAX_INT: int                       // valor máximo de int
const MIN_INT: int                       // valor mínimo de int
const MAX_FLOAT: float                   // valor máximo de float
const EPSILON: float                     // menor diferença representável
```

### Funções — Básicas

```
fn abs(x: float): float
fn ceil(x: float): int
fn floor(x: float): int
fn round(x: float): int
fn round(x: float, decimals: int): float // round(3.14159, 2) => 3.14
fn min(a: float, b: float): float
fn max(a: float, b: float): float
fn clamp(x: float, low: float, high: float): float   // limita x entre low e high
fn lerp(a: float, b: float, t: float): float         // interpolação linear
```

### Funções — Potências e Logaritmos

```
fn pow(base: float, exp: float): float
fn sqrt(x: float): result<float, MathError>            // erro se x < 0
fn cbrt(x: float): float
fn log(x: float): result<float, MathError>             // logaritmo natural, erro se x <= 0
fn log2(x: float): result<float, MathError>
fn log10(x: float): result<float, MathError>
fn exp(x: float): float                                // e^x
```

### Funções — Trigonometria

```
fn sin(x: float): float                                // x em radianos
fn cos(x: float): float
fn tan(x: float): float
fn asin(x: float): result<float, MathError>            // erro se |x| > 1
fn acos(x: float): result<float, MathError>            // erro se |x| > 1
fn atan(x: float): float
fn atan2(y: float, x: float): float
fn toRadians(degrees: float): float
fn toDegrees(radians: float): float
```

### Funções — Utilidades

```
fn isNan(x: float): bool
fn isInf(x: float): bool
fn isFinite(x: float): bool
fn sign(x: float): int                                 // -1, 0, ou 1
```

### Exemplos

```
from math use PI, abs, ceil, floor, round, min, max, sqrt, pow, sin, cos,
             toRadians, clamp, lerp, isNan, NAN

// Básicas
println(abs(-42))                        // 42
println(ceil(3.2))                       // 4
println(floor(3.9))                      // 3
println(round(3.5))                      // 4
println(round(3.14159, 2))              // 3.14
println(min(10, 20))                     // 10
println(max(10, 20))                     // 20

// Potências
raiz = try sqrt(144.0)                  // 12.0
println(pow(2.0, 10.0))                 // 1024.0

// Trigonometria
angulo = toRadians(90.0)
println(sin(angulo))                     // 1.0
println(cos(angulo))                     // 0.0 (aprox.)

// Utilidades
println(clamp(15.0, 0.0, 10.0))        // 10.0
println(lerp(0.0, 100.0, 0.5))         // 50.0
println(isNan(NAN))                      // true
```

---

## math.random — Números Aleatórios

```
from math.random use random, randomInt, randomFloat, seed, shuffle, choice
```

### Funções

```
fn random(): float                                      // [0.0, 1.0)
fn randomInt(min: int, max: int): int                   // [min, max] inclusivo
fn randomFloat(min: float, max: float): float           // [min, max)
fn seed(value: int): void                               // define semente para reprodutibilidade
fn shuffle(items: array<dynamic>): array<dynamic>       // retorna novo array embaralhado
fn choice(items: array<dynamic>): dynamic               // retorna elemento aleatório
```

### Exemplos

```
from math.random use random, randomInt, randomFloat, seed, shuffle, choice

// Números aleatórios
r = random()                             // 0.7312... (entre 0.0 e 1.0)
n = randomInt(1, 100)                    // 42 (entre 1 e 100)
f = randomFloat(0.0, 10.0)             // 3.721...

// Reprodutibilidade
seed(12345)
a = randomInt(1, 100)                    // sempre o mesmo valor com mesma semente

// Arrays
nums = [1, 2, 3, 4, 5]
embaralhado = shuffle(nums)              // [3, 1, 5, 2, 4] (exemplo)
escolhido = choice(nums)                // 3 (exemplo)
```
