---
name: commit-push-deliver
description: Use when the user asks to start a Runrun.it task (fetch details, create branch, move card), deliver a task (commit to branch, push, post comments on card), or deploy to production (merge branch into master, push, post comment).
---

# Gerenciar Ciclo de Demanda (Runrun.it)

## Overview

Três modos para o ciclo completo de uma demanda:
- **INICIAR** — busca contexto, cria branch e move o card
- **ENTREGAR** — commit na branch, push e comenta no card
- **SUBIR PARA PRODUÇÃO** — merge na master, push e comenta no card

## Regras

- Só mova o card para "19 - Aviso Entrega" se for através do modo SUBIR PARA PRODUÇÃO.

---

## Modo: INICIAR demanda

**Acionado por:** "inicia a demanda X", "começa a demanda X", "dá uma olhada na demanda X"

### 1. Buscar detalhes da tarefa (paralelo)

```
tasks_get(id)  +  tasks_get_description(id)
```
Verificar via `tasks_get` se o campo `is_working_on` indica que o usuário atual está com o play ativo na demanda:

- **Timer ativo** → pular esta etapa
- **Timer inativo** → chamar `tasks_play(id)` para iniciar

### 2. Apresentar resumo

Exibir: título, cliente, projeto, estágio atual, responsável e descrição limpa (sem HTML).
Verificar, também, a parte de comentários para apurar se há mais informações relevantes sobre a demanda.

### 3. Verificar esteira e mover se necessário

Verificar o campo `board_stage_name` retornado pelo `tasks_get`:

- **Já está em "10 - A fazer/fazendo Front"** → pular esta etapa
- **Está em outra esteira** → mover com `tasks_update_status` usando `board_stage_id: 2195275`

> ⚠️ **Limitação conhecida:** `tasks_update_status` retorna `{}` consistentemente sem efetivar. Sempre avisar o usuário para mover manualmente para **"10 - A fazer/fazendo Front"**.

### 4. Verificar/criar branch

Convenção de nome: `task-{ID}` (ex: `task-68207`)

```bash
# Verifica se branch remota ou local existe
git fetch origin
git branch -a | grep "task-{ID}"
```

- **Existe localmente:** `git checkout task-{ID}`
- **Existe no remote:** `git checkout -b task-{ID} origin/task-{ID}`
- **Não existe:** Atualize a branch principal antes de criar a nova:
  ```bash
  # Substitua 'master' por 'main' se necessário
  git checkout master
  git pull origin master
  git checkout -b task-{ID}
  ```

### 5. Pausar timer

Verificar via `tasks_get` se o campo `is_working_on` indica que o usuário atual está com o play ativo na demanda:

- **Timer ativo** → chamar `tasks_pause(id)` para pausar
- **Timer inativo** → pular esta etapa

### 6. Confirmar ao usuário

Informar: branch ativa, esteira atual do card, e resumo do problema a resolver.

---

## Modo: ENTREGAR demanda

**Acionado por:** "entrega a demanda", "commita e entrega", "commita, push e entrega"

### 1. Garantir que está na branch correta

```bash
git branch --show-current
# Se não estiver na branch da demanda, fazer checkout
```

### 2. Identificar arquivos relevantes

```bash
git status
git diff
```

Commitar **apenas** os arquivos da correção — ignorar modificações pré-existentes não relacionadas.

### 3. Commit

```bash
git add <arquivo(s)>
git commit -m "$(cat <<'EOF'
fix(escopo): descrição objetiva do que foi corrigido e por quê

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

### 4. Push na branch e capturar hash

```bash
git push origin task-{ID} && git log -1 --format="%H"
```

### 5. Montar URL do commit

```bash
git remote get-url origin
# git@bitbucket.org:ORG/REPO.git → https://bitbucket.org/ORG/REPO/commits/HASH
```

### 6. Comentário de entrega no card

```
# Notas Técnicas da Implementação (DEV)

---

## 1. Resumo da Implementação

<Breve descrição do que foi implementado/corrigido e qual regra/comportamento foi alterado.>

---

## 2. Mudanças Realizadas

- <Arquivo/serviço/método alterado e o que foi feito>
- <Arquivo/serviço/método alterado e o que foi feito>
- <...>

---

## 3. Detalhamento Técnico

### 3.1 Arquivos/Classes Alterados

- `<caminho/Arquivo.ext>`
- `<caminho/Arquivo.ext>`

### 3.2 Fluxo Alterado

**Antes:**
1. <Passo 1 do fluxo anterior>
2. <Passo 2 do fluxo anterior>

**Agora:**
1. <Passo 1 do novo fluxo>
2. <Passo 2 do novo fluxo>

### 3.3 Justificativa Técnica

<Por que esta abordagem foi escolhida. Decisões de design, trade-offs, padrões seguidos.>

---

## 4. Impactos

- <Impacto 1 no sistema/usuário>
- <Impacto 2 em integrações ou outros módulos>

---

## 5. Riscos e Considerações

- <Risco ou ponto de atenção 1>
- <Risco ou ponto de atenção 2>

---

## 6. Instruções para QA

- Testar <cenário 1>: deve <comportamento esperado>.
- Testar <cenário 2>: deve <comportamento esperado>.
- <...>

---

## 7. Feature Flags / Configurações

- <Flag criada ou "Nenhuma flag criada.">
- <Dependência de ambiente ou "Não há dependências de ambiente.">

---

## 8. Referências

- **Branch:** task-{ID}
- **Commit:** https://bitbucket.org/ORG/REPO/commits/HASH
```

### 7. Verificar esteira e mover se necessário

Verificar o campo `board_stage_name` atual do card via `tasks_get` e mover conforme a esteira:

- **Está em "08 - A fazer/Fazendo Back"** → mover com `tasks_update_status` usando `board_stage_id: 2195274` ("09 - Validação Back")
- **Está em "10 - A fazer/fazendo Front"** → mover com `tasks_update_status` usando `board_stage_id: 2195276` ("11 - Validação Front")
- **Já está em "09 - Validação Back" ou "11 - Validação Front"** → pular esta etapa

> ⚠️ **Limitação conhecida:** `tasks_update_status` retorna `{}` consistentemente sem efetivar. Sempre avisar o usuário para mover manualmente para o estágio correto.

### 8. Pausar timer se estiver rodando

Verificar via `tasks_get` se o campo `is_working_on` indica que o usuário atual está com o play ativo na demanda:

- **Timer ativo** → chamar `tasks_pause(id)` para pausar
- **Timer inativo** → pular esta etapa

---

## Modo: SUBIR PARA HOMOLOGAÇÃO

**Acionado por:** "sobe para homologação", "mergeia na dev", "coloca em homologação", "sobe para hm", "coloca em hm"

> **Modelo:** delegar toda a execução deste modo a um sub-agente via ferramenta `Agent` com `model: "claude-haiku-4-5-20251001"`. Passar o número da demanda e os passos abaixo como prompt do agente.

### 1. Confirmar branch atual e atualizar dev

```bash
git checkout dev
git pull origin dev
```

### 2. Merge da branch da demanda

```bash
git merge task-{ID} --no-ff -m "$(cat <<'EOF'
Merge task-{ID}: <título resumido da demanda>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

### 3. Push na dev e capturar hash

```bash
git push origin dev && git log -1 --format="%H"
```

### 4. Verificar esteira e mover se necessário

Verificar o campo `board_stage_name` atual do card via `tasks_get`:

- **Já está em "14 - Teste HM"** → pular esta etapa
- **Está em "09 - Validação Back"** → mover com `tasks_update_status` usando `board_stage_id: 2195275` ("10 - A fazer/fazendo Front")
- **Está em outra esteira** → mover com `tasks_update_status` usando `board_stage_id: 2195279`

> ⚠️ **Limitação conhecida:** `tasks_update_status` retorna `{}` consistentemente sem efetivar. Sempre avisar o usuário para mover manualmente para o estágio correto (**"10 - A fazer/fazendo Front"** ou **"14 - Teste HM"**).

### 5. Comentário técnico no card

```
🚀 *Deploy em homologação*

**Branch mergeada:** task-{ID} → dev
**Commit:** https://bitbucket.org/ORG/REPO/commits/HASH

JSON TESTER:
{
  "card_id": "RR-4821",
  "titulo": "Corrigir cálculo de carência na contratação",
  "contexto_negocio": "...",
  "arquivos_impactados": [
    "app/Services/CarenciaService.php",
    "app/Http/Controllers/ContratacaoController.php"
  ],
  "endpoint_relacionado": "POST /contratacao/plano"
}
```

### 6. Pausar timer se estiver rodando

Verificar via `tasks_get` se o campo `is_working_on` indica que o usuário atual está com o play ativo na demanda:

- **Timer ativo** → chamar `tasks_pause(id)` para pausar
- **Timer inativo** → pular esta etapa

---

## Modo: SUBIR PARA PRODUÇÃO

**Acionado por:** "sobe para produção", "mergeia na master", "coloca em produção"

> **Modelo:** delegar toda a execução deste modo a um sub-agente via ferramenta `Agent` com `model: "claude-haiku-4-5-20251001"`. Passar o número da demanda e os passos abaixo como prompt do agente.

### 1. Confirmar branch atual e atualizar master

```bash
git checkout master
git pull origin master
```

### 2. Merge da branch da demanda

```bash
git merge task-{ID} --no-ff -m "$(cat <<'EOF'
Merge task-{ID}: <título resumido da demanda>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

### 3. Push na master e capturar hash

```bash
git push origin master && git log -1 --format="%H"
```

### 4. Verificar esteira e mover se necessário

Verificar o campo `board_stage_name` atual do card via `tasks_get`:

- **Já está em "19 - Aviso Entrega"** → pular esta etapa
- **Está em outra esteira** → mover com `tasks_update_status` usando `board_stage_id: 2195286`

> ⚠️ **Limitação conhecida:** `tasks_update_status` retorna `{}` consistentemente sem efetivar. Sempre avisar o usuário para mover manualmente para **"19 - Aviso Entrega"**.

### 5. Comentário técnico no card

```
🚀 *Deploy em produção*

**Branch mergeada:** task-{ID} → master
**Commit:** https://bitbucket.org/ORG/REPO/commits/HASH
```

### 6. Pausar timer se estiver rodando

Verificar via `tasks_get` se o campo `is_working_on` indica que o usuário atual está com o play ativo na demanda:

- **Timer ativo** → chamar `tasks_pause(id)` para pausar
- **Timer inativo** → pular esta etapa

---

## Quick Reference

| Etapa | Ferramenta |
|-------|-----------|
| Buscar tarefa | `tasks_get` + `tasks_get_description` (paralelo) |
| Verificar branch | `git fetch origin` + `git branch -a` |
| Criar/trocar branch | `git checkout -b task-{ID}` |
| Ver mudanças | `git status` + `git diff` |
| Commit | `git add <arquivo>` + `git commit` |
| Push branch | `git push origin task-{ID}` |
| Merge master | `git checkout master` + `git merge task-{ID} --no-ff` |
| Push master | `git push origin master` |
| URL commit | `git remote get-url origin` → HTTPS |
| Comentários | `tasks_comments_create` (técnico + cliente) |
| Mover para "08 - A fazer/Fazendo Back" | `tasks_update_status` com `board_stage_id: 2195273` |
| Mover para "09 - Validação Back" | `tasks_update_status` com `board_stage_id: 2195274` |
| Mover para "10 - A fazer/fazendo Front" | `tasks_update_status` com `board_stage_id: 2195275` |
| Mover para "11 - Validação Front" | `tasks_update_status` com `board_stage_id: 2195276` |
| Mover para "14 - Teste HM" | `tasks_update_status` com `board_stage_id: 2195279` |
| Mover para "19 - Aviso Entrega" | `tasks_update_status` com `board_stage_id: 2195286` |
| Pausar timer da demanda | `tasks_pause(id)` (apenas se `is_working_on` estiver ativo) |

## Common Mistakes

- **Commitar arquivos não relacionados** — sempre verificar `git diff` antes de `git add`
- **Push na master direto sem branch** — sempre trabalhar na branch `task-{ID}` até subir para produção
- **Merge sem `--no-ff`** — usar `--no-ff` para preservar histórico da branch no merge
- **Esquecer de capturar hash** — `git push && git log -1 --format="%H"` em um comando só
- **URL errada do commit** — converter `git@bitbucket.org:ORG/REPO.git` → `https://bitbucket.org/ORG/REPO/commits/HASH`
- **Comentário cliente técnico demais** — linguagem simples, focar no benefício e em como usar
- **Mover esteira sem verificar ID** — sempre usar `tasks_list` + grep para confirmar o `board_stage_id`
