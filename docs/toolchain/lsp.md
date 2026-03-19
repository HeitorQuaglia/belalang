# Toolchain — LSP

O LSP de Bela é co-designed com o package manager — resolução de módulos depende de `bela.toml`. O maior value add é o diagnóstico imediato de violações de `fixed<T>`, que é o principal mecanismo de segurança de tipos da linguagem.

---

## Formatting — bela fmt

`bela fmt` é opinionated: sem configuração (estilo gofmt). Qualquer arquivo `.bela` é formatado de forma idêntica, sem flags, sem preferências de projeto. Elimina discussões de estilo e garante que todo código Bela tenha aparência uniforme.

```
bela fmt arquivo.bela          # formata in-place
bela fmt --check arquivo.bela  # verifica sem modificar (útil em CI)
```

---

## Diagnósticos

### fixed<T> violations

> **TBD:** Assignment de tipo incompatível para variável `fixed<T>` deve aparecer sublinhado em vermelho imediatamente — a confirmar como feature prioritária v1. Será definido antes da implementação.

### Inlay Hints

> **TBD:** Tipos inferidos de expressões dynamic como hints inline — a confirmar como feature de alta prioridade. Será definido antes da implementação.

---

## Completions

### Membros de módulo importado

Sempre disponível: após `from io use`, o LSP sugere os símbolos exportados pelo módulo `io`. Não depende de inferência de tipo — é derivado diretamente da interface pública do módulo.

### Inferência para dynamic

> **TBD:** Profundidade de inferência de tipo para variáveis dynamic. Opções: apenas membros de imports (sempre possível) vs. inferência por contexto/uso (best-effort como TypeScript). Será definido antes da implementação.

---

## Itens Futuros

> **RESOLVIDO:** Parser tolerante a erros — decisão em `docs/superpowers/specs/2026-03-18-compilation-frontend-design.md`. A estratégia adotada é error nodes + pontos de sincronização no parser compartilhado (opção a), com tokens de trivia emitidos em modo LSP. Nenhum parser separado.

> **TBD:** Module resolution — protocolo para resolver `from stdlib`, `from dependência externa` (via `bela.toml`) e `from módulo local`. LSP e package manager precisam ser co-designed. Será definido antes da implementação.

---

## Sequenciamento

LSP v1 (Diagnostics + Completion básico) vem após:

1. Bytecode compilando corretamente
2. Hot reload estabilizado

O LSP v1 é o DX mínimo aceitável. Funcionalidades avançadas (inlay hints, diagnósticos de `fixed<T>`, module resolution completa) são v2+.
