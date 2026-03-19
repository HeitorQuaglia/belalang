# Toolchain — Source Maps

O `.belmap` é um arquivo binário separado, produzido opcionalmente pelo compilador junto com o `.belc`. O host decide se distribui o arquivo na aplicação final ou não.

---

## Convenção de Nomes

A convenção de nomenclatura permite que o debugger (ver [debugger.md](debugger.md)) localize o source map automaticamente:

- `game.bela` → `game.belc` + `game.belmap` (mesmo diretório, mesmo nome base)
- O debugger busca o `.belmap` por esta convenção no `launch`/`attach`

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

Cada `Entry` relaciona uma posição no bytecode a uma localização no arquivo-fonte. Múltiplas instruções podem apontar para a mesma linha (e.g., compilação otimizada). Linhas e colunas são 1-based.

---

## Comportamento por Presença do `.belmap`

| Situação | Stack traces e breakpoints |
|----------|----------------------------------|
| **Sem `.belmap`** | Mostram offset de bytecode — `game.belc:0x00A4` |
| **Com `.belmap`** | Mostram localização no source — `game.bela:42:7` |

---

## Distribuição

- **Produção:** tipicamente omitido — evita expor localização do código-fonte.
- **Desenvolvimento:** essencial para stack traces e navegação de breakpoints.

---

## Itens em Aberto

> **TBD:** Campos adicionais do `.belmap` (nomes de variáveis para inline hints, scope boundaries). Será definido antes da implementação.

