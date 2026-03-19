# C API — Valores

`BelaValue` é uma tagged union opaca que representa qualquer valor Bela no lado C. O host nunca acessa a representação interna diretamente — toda leitura e criação de valores é feita por meio das funções da API descritas neste documento.

> **Nota:** A maior parte da API para tipos complexos (arrays, objetos, result) ainda está TBD. O que está decidido são os primitivos e as regras de ownership de strings.

---

## Primitivos (Decidido)

### Construtores

```c
BelaValue bela_value_int(int64_t v);
BelaValue bela_value_float(double v);
BelaValue bela_value_bool(bool v);
BelaValue bela_value_null(void);
BelaValue bela_value_string(BelaVM *vm, const char *s);
```

`bela_value_string` requer a VM pois aloca memória gerenciada internamente. Bela **sempre copia** a string `s` — o host pode liberar ou reutilizar o ponteiro imediatamente após a chamada.

```c
// s pode ser liberado logo após — Bela já copiou
char *s = strdup("olá mundo");
BelaValue v = bela_value_string(vm, s);
free(s);  // seguro

// Literal: também seguro
BelaValue nome = bela_value_string(vm, "Bela");
```

### Acessores

```c
int64_t     bela_as_int(BelaValue v);
double      bela_as_float(BelaValue v);
bool        bela_as_bool(BelaValue v);
const char *bela_as_string(BelaValue v);  // ponteiro owned pela VM
int         bela_is_null(BelaValue v);
```

O ponteiro retornado por `bela_as_string` é owned pela VM. O host não deve liberá-lo nem armazená-lo além da duração do valor Bela correspondente.

> **Nota:** Chamar `bela_as_int` em um valor que não é int é comportamento indefinido. Use as funções de introspection (TBD abaixo) para verificar o tipo antes de acessar.

```c
// Padrão seguro de acesso (com introspection TBD)
// if (bela_is_int(v)) {
//     int64_t n = bela_as_int(v);
// }

// Hoje, se o host controla o tipo (ex: retorno de função conhecida):
int64_t n = bela_as_int(resultado);
```

---

## Tipos Complexos

> **TBD:** `bela_value_array`, `bela_array_push`, `bela_value_object`, `bela_value_result_ok`, `bela_value_result_error`. Necessários para interop real com arrays, objetos e `result<T,E>` do lado C. Será definido antes da implementação.

---

## Introspection

> **TBD:** `bela_typeof()` retornando enum `BelaType` (`BELA_TYPE_INT`, `BELA_TYPE_FLOAT`, `BELA_TYPE_STRING`, `BELA_TYPE_OBJECT`, etc.), ou variantes `bela_is_int(v)`, `bela_is_string(v)`, `bela_is_array(v)`. Necessário para uso seguro dos acessores. Será definido antes da implementação.

---

## RC Interop — CRÍTICO

> **TBD:** `bela_value_retain(BelaVM *vm, BelaValue v)` e `bela_value_release(BelaVM *vm, BelaValue v)`. Sem essas funções, qualquer objeto Bela (array, instância de classe) retornado ao C é uma bomba-relógio: se o script perder todas as suas referências para o objeto, o RC o coleta e o host fica com ponteiro inválido (use-after-free). Será definido antes da implementação.

---

## Acesso a Campos

> **TBD:** `bela_object_get(BelaVM *vm, BelaValue obj, const char *field)` e `bela_object_set(BelaVM *vm, BelaValue obj, const char *field, BelaValue v)`. Necessários para ler e escrever campos de objetos Bela recebidos do script. Será definido antes da implementação.

---

## Stacktrace (Decidido)

```c
const char *bela_vm_stacktrace(BelaVM *vm);
```

Retorna o stacktrace textual do último erro ou panic da VM. O ponteiro é owned pela VM e permanece válido até a próxima chamada à API. O host não deve liberá-lo.

```c
BelaStatus status = bela_vm_call(vm, "onUpdate", 1, args);
if (status == BELA_PANIC) {
    fprintf(stderr, "panic:\n%s\n", bela_vm_stacktrace(vm));
    bela_vm_free(vm);
}
```

> **Nota:** A string retornada por `bela_vm_stacktrace` invalida na próxima chamada à API. Se o host precisar preservá-la, deve copiar com `strdup` antes de fazer outra chamada.
