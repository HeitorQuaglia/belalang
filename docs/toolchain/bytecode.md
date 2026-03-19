# Toolchain — Bytecode

O compilador Bela transforma arquivos `.bela` em bytecode `.belc`, opcionalmente acompanhado de um source map `.belmap`. O bytecode protege a propriedade intelectual do desenvolvedor — sem ele, o código-fonte precisa ser distribuído diretamente. A função `bela_vm_load_file()` aceita ambos os formatos e detecta automaticamente pelo header binário do arquivo (referência autoritativa: [capi/embedding.md](../capi/embedding.md#bela_vm_load_file)).

---

## Arquivos de Saída

| Arquivo    | Descrição                                              |
|------------|--------------------------------------------------------|
| `.belc`    | Bytecode compilado — bundle único (v1)                 |
| `.belmap`  | Source map — separado, opcional, mesmo nome base       |

Convenção: `game.bela` → `game.belc` + `game.belmap` (opcional)

### .belc — Bundle Único (v1)

O formato v1 adota o modelo de bundle único: todo o código do projeto é compilado em um único arquivo `.belc`. Esse modelo simplifica a distribuição — um único arquivo a empacotar, sem dependências entre arquivos de módulo.

Compilação por módulo com etapa de linking (como `.o` + linker em C) fica para versão futura.

### .belmap — Source Map Separado

O source map é distribuído independentemente do `.belc`. O host decide se inclui o `.belmap` na distribuição ou não — em produção, é comum omiti-lo para não expor localização de código; em desenvolvimento, ele é essencial para mensagens de erro úteis.

- **Sem `.belmap`:** erros e stack traces mostram localização no bytecode (offset de instrução).
- **Com `.belmap`:** erros e stack traces mostram linha e coluna do arquivo `.bela` original.

> **TBD:** Formato exato do `.belmap`. O schema será compartilhado com o debugger — não duplicar. Ver `toolchain/debugger.md`. Será definido antes da implementação.

---

## Versionamento

A estratégia de versionamento adotada é **strict rejection**.

Ao carregar um `.belc`, a VM verifica a versão no header:

- **Versão compatível** → executa normalmente.
- **Versão incompatível** → falha com `BELA_COMPILE_ERROR` e exibe a mensagem:

  ```
  bytecode compilado para VM vX.Y, esta VM é vZ.W. Por favor recompile.
  ```

Não há backward-compatibility intencional. Essa decisão é viável porque o host embarca a VM junto com o jogo ou aplicação — ele controla qual versão da VM está em uso e pode recompilar os `.belc` ao atualizar a VM.

---

## Itens Futuros

> **TBD:** Bytecode encryption via C API — suporte a chave de decriptação fornecida pelo host para proteção adicional de IP. Fora do escopo v1, mas um slot está reservado no header para não quebrar o formato futuramente. Será definido antes da implementação.

> **TBD:** Constant pool — string interning e escopo do pool (por módulo vs. por bundle). Será definido antes da implementação.

> **TBD:** Source maps compartilhados entre bytecode e debugger — o formato `.belmap` deve ser definido uma vez e consumido pelos dois. Não duplicar schema. Ver `toolchain/debugger.md`. Será definido antes da implementação.
