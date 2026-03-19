# Toolchain — Source Maps

O `.belmap` é um arquivo binário separado, produzido opcionalmente pelo compilador junto com o `.belc`. O host decide se distribui o arquivo na aplicação final ou não — em produção é tipicamente omitido; em desenvolvimento é essencial para stack traces úteis e navegação de breakpoints.

---

## Convenção de Nomes

A convenção de nomenclatura permite que o debugger localize o source map automaticamente:

- `game.bela` → `game.belc` + `game.belmap` (mesmo diretório, mesmo nome base)
- O debugger busca o `.belmap` por esta convenção no `launch`/`attach`

Se não encontrar o arquivo, o debugger prossegue sem source map — stack traces mostrarão offsets de bytecode em vez de linhas do código-fonte.

---

## Estrutura

O arquivo `.belmap` segue a estrutura binária abaixo (pseudo-código):

```
Header:
  magic:       [4]u8   // "BELM"
  version:     u16     // versão do formato
  source_file: []u8    // path do .bela original (null-terminated)

Entries: []Entry
  Entry:
    bytecode_offset: u32  // offset no .belc
    line:            u32  // linha no .bela (1-based)
    column:          u16  // coluna no .bela (1-based)
```

- `magic`: sempre `"BELM"` (4 bytes ASCII)
- `version`: versão do formato `.belmap` (u16 big-endian)
- `source_file`: caminho original do arquivo `.bela`, terminado em null byte (`\0`)
- `Entries`: lista de mapeamentos bytecode-to-source, uma entrada por instrução mapeada

Cada `Entry` relaciona uma posição no bytecode a uma localização no arquivo-fonte (linha e coluna). Múltiplas instruções podem apontar para a mesma linha (e.g., compilação otimizada). Linhas e colunas são 1-based.

---

## Comportamento por Presença do `.belmap`

| Situação | Stack traces e breakpoints |
|----------|----------------------------------|
| **Sem `.belmap`** | Mostram offset de bytecode — `game.belc:0x00A4` |
| **Com `.belmap`** | Mostram localização no source — `game.bela:42:7` |

O debugger carrega o `.belmap` durante `launch`/`attach`. Se o arquivo não existir, o comportamento degrada para offsets de bytecode — a execução prossegue normalmente.

---

## Distribuição

- **Em produção:** Tipicamente omitido da distribuição final. Inclui localização de código-fonte que o desenvolvedor pode considerar proprietária. Sem o `.belmap`, erros mostram apenas offsets, dificultando análise de stack traces mas reduzindo exposição.
- **Em desenvolvimento:** Essencial. Permite que o debugger mapeie breakpoints e stepping para linhas do código, e que mensagens de erro identifiquem a localização exata do problema.

---

## Itens em Aberto

> **TBD:** Campos adicionais do `.belmap` (nomes de variáveis para inline hints, scope boundaries). Será definido antes da implementação.

---

## Cross-References

O `.belmap` é consumido principalmente pelo debugger — ver [debugger.md](debugger.md) para uso em breakpoints e evaluate at breakpoint. É produzido pelo compilador junto com o `.belc` — ver [bytecode.md](bytecode.md) para contexto sobre a cadeia de compilação.
