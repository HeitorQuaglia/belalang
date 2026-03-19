# Toolchain — Debugger

> **Nota:** Esta documentação é um stub intencional. O debugger é v2 — a implementação
> depende de bytecode e source maps estabilizados (ver toolchain/bytecode.md).
> O conteúdo abaixo documenta o que está decidido; o restante será preenchido
> quando o debugger entrar em escopo.

---

## Protocolo

DAP (Debug Adapter Protocol) via TCP. Bela estende DAP com customRequests para
funcionalidades específicas da linguagem.

---

## Segurança do Debug Port (Decidido)

- Somente localhost (127.0.0.1)
- Sem autenticação
- ⚠ Somente para desenvolvimento. Nunca expor em produção. Nunca bind em 0.0.0.0.

---

## RC Inspection

> **TBD:** customRequest `bela/inspect` para inspeção de reference counts.
> Schema da resposta (address, rc, fields, possible_cycle) a definir.
> Será definido antes da implementação.

---

## Evaluate at Breakpoint

> **TBD:** Avaliação de expressões durante pausa sem encerrar a sessão de debug
> (sub-evaluator sandboxado). Será definido antes da implementação.

---

## Breakpoint Condicional

> **TBD:** Condições avaliadas com acesso ao escopo local completo.
> Depende de evaluate at breakpoint. Será definido antes da implementação.

---

## Event Loop no Breakpoint

> **TBD:** Comportamento quando pausado em breakpoint — timers param, promises
> não resolvem, conexões podem dar timeout. A documentar explicitamente.
> Será definido antes da implementação.

---

## Split Produção vs. Desenvolvimento

> **TBD:** Exclusão de código de debug em builds de produção.
> Mecanismo (`#ifdef BELA_DEBUG` ou binários separados `bela-vm.a` vs `bela-dev.a`) a definir.
> Será definido antes da implementação.

---

## Source Maps Compartilhados com Bytecode

> **TBD:** Formato `.belmap` definido em toolchain/bytecode.md e consumido aqui.
> Não duplicar schema. Ver toolchain/bytecode.md.
> Será definido antes da implementação.

---

## Sequenciamento

O debugger é v2:
1. Requer bytecode compilando corretamente (ver toolchain/bytecode.md)
2. Requer formato `.belmap` estabilizado
