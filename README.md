# commit-push-deliver

Skill para Claude Code que automatiza o ciclo completo de uma demanda no **Runrun.it** — do início ao deploy em produção.

## O que faz

Três modos de operação:

| Modo | Acionado por | O que acontece |
|------|-------------|----------------|
| **INICIAR** | "inicia a demanda 12345" | Busca detalhes do card, cria/troca para branch `task-{ID}`, move o card para "A fazer/fazendo" |
| **ENTREGAR** | "entrega a demanda" | Commit seletivo, push na branch, posta comentário de entrega no card, move para "Teste HM" |
| **SUBIR PARA PRODUÇÃO** | "sobe para produção" | Merge na master com `--no-ff`, push, posta comentário de deploy, pausa timer se ativo |

## Pré-requisitos

- **Claude Code** instalado (`npm install -g @anthropic-ai/claude-code`)
- **MCP Runrun.it** configurado no Claude Code (necessário para `tasks_get`, `tasks_comments_create`, etc.)
- Repositório Git com acesso ao Bitbucket configurado via SSH

## Instalação

### 1. Clonar o repositório

```bash
git clone git@github.com:seu-usuario/commit-push-deliver.git ~/.claude/skills/commit-push-deliver
```

Ou copiar manualmente a pasta para o diretório de skills do Claude Code:

```bash
cp -r /caminho/para/commit-push-deliver ~/.claude/skills/
```

### 2. Verificar a estrutura

```
~/.claude/skills/
  commit-push-deliver/
    SKILL.md
    README.md
```

### 3. Confirmar que o skill está disponível

Abra uma nova sessão do Claude Code. O skill aparece automaticamente — não é preciso nenhuma configuração extra. Você pode confirmar pedindo:

```
liste os skills disponíveis
```

## Como usar

Com o skill instalado, basta pedir naturalmente ao Claude Code:

```
# Iniciar uma demanda
inicia a demanda 68207

# Entregar após implementar
entrega a demanda

# Subir para produção
sobe para produção
```

O Claude seguirá o fluxo do skill automaticamente: busca os detalhes no Runrun.it, opera o Git, posta os comentários e move o card na esteira.

## Limitação conhecida

O `tasks_update_status` retorna `{}` sem efetivar a mudança de esteira. O Claude sempre avisa para mover o card manualmente no Runrun.it após cada etapa.
