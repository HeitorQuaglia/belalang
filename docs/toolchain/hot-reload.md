# Toolchain — Hot Reload

Hot reload permite substituir código Bela em tempo de execução sem reiniciar a VM. A responsabilidade é dividida: a VM expõe a API de reload, e o host detecta mudanças no sistema de arquivos e aciona o reload.

A VM **não faz file watching** — isso é deliberado. Monitoramento de arquivos depende de APIs do sistema operacional (`inotify` no Linux, `FSEvents` no macOS, `ReadDirectoryChanges` no Windows) e de loop de eventos do host. A VM não tem acesso a esse contexto. O host detecta a mudança e chama `bela_vm_reload()`.

Hot reload e bytecode são **mutuamente exclusivos por design**: produção usa `.belc` (sem código-fonte), desenvolvimento usa `.bela`. Não há suporte a hot reload em arquivos `.belc`.

---

## API

```c
BelaReloadStatus bela_vm_reload(BelaVM *vm, const char *filepath);
```

Recarrega o arquivo `.bela` indicado. A VM recompila o arquivo, valida compatibilidade com o código em execução e, se aprovado, substitui os corpos de função no estado da VM.

O arquivo deve ser o mesmo arquivo originalmente carregado via `bela_vm_load_file`. O filepath é usado como identificador — recarregar um arquivo diferente tem comportamento indefinido.

---

### BelaReloadStatus

> **TBD:** O enum `BelaReloadStatus` não está formalmente definido. Os valores abaixo são propostos e **não decididos**. Serão definidos antes da implementação.

```c
typedef enum {
    BELA_RELOAD_OK,             // reload aplicado com sucesso
    BELA_RELOAD_COMPILE_ERROR,  // sintaxe inválida ou erro de tipo no arquivo recarregado
    BELA_RELOAD_SCHEMA_CHANGE,  // assinaturas de função ou campos de struct/classe mudaram
    BELA_RELOAD_NOT_FOUND,      // arquivo não encontrado no caminho fornecido
} BelaReloadStatus;
```

| Valor                        | Quando ocorre (proposto)                                                              |
|------------------------------|---------------------------------------------------------------------------------------|
| `BELA_RELOAD_OK`             | Reload aplicado com sucesso. Novos corpos de função estão ativos.                     |
| `BELA_RELOAD_COMPILE_ERROR`  | O arquivo recarregado contém erros de sintaxe ou tipos inválidos.                     |
| `BELA_RELOAD_SCHEMA_CHANGE`  | Assinaturas de função ou campos de struct/classe mudaram — fora do escopo v1.         |
| `BELA_RELOAD_NOT_FOUND`      | Arquivo não encontrado no caminho fornecido.                                           |

> **Nota:** Em caso de `BELA_RELOAD_COMPILE_ERROR` ou `BELA_RELOAD_SCHEMA_CHANGE`, o código anterior permanece ativo. A VM não fica em estado inválido — o reload falhou de forma segura e o host pode continuar executando normalmente.

---

## Escopo do Reload (v1)

O reload em v1 é **function-body-only**: a VM aceita o novo código apenas se os contratos públicos não mudaram.

Condições para `BELA_RELOAD_OK`:
- Assinaturas de funções não mudaram (nome, número e ordem de parâmetros)
- Campos de structs e classes não mudaram (nome e ordem)

Se qualquer contrato mudou, `bela_vm_reload` retorna `BELA_RELOAD_SCHEMA_CHANGE`. O host decide o que fazer: reiniciar a VM, logar um aviso, ou ignorar. A VM permanece operacional com o código anterior.

Essa restrição cobre 90% do caso de uso em game dev: tweaking de lógica (comportamento de IA, cálculos de física, valores de balanceamento) sem alterar contratos entre módulos.

**Fora do escopo v1:**
- Method propagation — novos métodos chegando para objetos já instanciados
- Full migration ao estilo Erlang — módulo declara `on_hot_reload(old_state)` para migrar estado

---

## Aviso de Produção

> **Nota:** Hot reload **nunca deve ser usado em produção**. As funcionalidades são mutuamente exclusivas por design:
>
> - **Produção** → use `.belc` (bytecode pré-compilado, sem código-fonte, IP protegido)
> - **Desenvolvimento** → use `.bela` (código-fonte, hot reload disponível)
>
> Tentar chamar `bela_vm_reload` em uma VM que carregou um `.belc` tem comportamento indefinido.

---

## Itens Futuros

> **TBD:** Re-execução de top-level code — hot reload não deve re-executar side effects do init (ex: `db = Database.connect(...)`). Deve substituir definições de função e constantes literais sem re-executar código com efeitos colaterais. Como distinguir? Implicitamente (expressão computada vs. definição pura) ou explicitamente via anotação? Será definido antes da implementação.

> **TBD:** Method propagation — novos métodos chegam para objetos já instanciados. Versão futura.

> **TBD:** Full migration (modelo Erlang) — módulo declara `on_hot_reload(old_state)` para migrar estado existente para o novo schema. Versão futura.

---

## Exemplo de Integração com File Watching

O exemplo abaixo mostra um host Linux usando `inotify` para detectar mudanças e acionar `bela_vm_reload`. A mesma lógica se aplica com `FSEvents` no macOS ou `ReadDirectoryChanges` no Windows.

```c
#include "bela.h"
#include <sys/inotify.h>
#include <unistd.h>
#include <stdio.h>

#define SCRIPT_PATH "scripts/ai.bela"

int main(void) {
    // 1. Criar e inicializar a VM normalmente
    BelaVM *vm = bela_vm_new(NULL);

    BelaStatus status = bela_vm_load_file(vm, SCRIPT_PATH);
    if (status != BELA_OK) {
        fprintf(stderr, "falha ao carregar: %s\n", bela_vm_last_error(vm));
        bela_vm_free(vm);
        return 1;
    }

    // 2. Registrar watcher (host é responsável por isso — VM não faz watching)
    int inotify_fd = inotify_init1(IN_NONBLOCK);
    int watch_fd   = inotify_add_watch(inotify_fd, SCRIPT_PATH, IN_CLOSE_WRITE);

    // 3. Loop principal do host
    char buf[4096];
    while (1) {
        // ... executar frame do jogo/aplicação ...
        BelaValue delta = bela_value_float(0.016);
        bela_vm_call(vm, "onUpdate", 1, &delta);

        // 4. Verificar se há mudança no arquivo
        ssize_t len = read(inotify_fd, buf, sizeof(buf));
        if (len > 0) {
            // Mudança detectada — tentar reload
            BelaReloadStatus reload = bela_vm_reload(vm, SCRIPT_PATH);

            switch (reload) {
                case BELA_RELOAD_OK:
                    fprintf(stderr, "[hot-reload] scripts/ai.bela recarregado com sucesso\n");
                    break;

                case BELA_RELOAD_COMPILE_ERROR:
                    // Código anterior permanece ativo — logar e continuar
                    fprintf(stderr, "[hot-reload] erro de compilação: %s\n", bela_vm_last_error(vm));
                    break;

                case BELA_RELOAD_SCHEMA_CHANGE:
                    // Assinaturas ou campos mudaram — host decide o que fazer
                    fprintf(stderr, "[hot-reload] schema mudou — reiniciando VM\n");
                    bela_vm_free(vm);
                    vm = bela_vm_new(NULL);
                    bela_vm_load_file(vm, SCRIPT_PATH);
                    break;

                case BELA_RELOAD_NOT_FOUND:
                    fprintf(stderr, "[hot-reload] arquivo não encontrado: %s\n", SCRIPT_PATH);
                    break;
            }
        }
    }

    // 5. Limpeza
    inotify_rm_watch(inotify_fd, watch_fd);
    close(inotify_fd);
    bela_vm_free(vm);
    return 0;
}
```

> **Nota:** O exemplo usa `IN_CLOSE_WRITE` (não `IN_MODIFY`) para aguardar o editor fechar o arquivo antes de tentar recarregar. `IN_MODIFY` pode disparar enquanto o arquivo ainda está sendo escrito, causando erros de compilação transitórios.
