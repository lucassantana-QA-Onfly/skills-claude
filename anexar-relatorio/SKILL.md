---
name: anexar-relatorio
description: Anexa um relatório de execução de testes como comentário no ticket Project (TEST-XX) do Jira. Recebe a chave por argumento (ex: /anexar-relatorio TEST-77) ou detecta pelo contexto, e usa o resumo enviado pelo usuário como corpo do relatório. Não envia notificação e não faz transição de status.
argument-hint: "[test-key]"
---

# Skill: Anexar Relatório de Teste

## Como usar
Invocada com chave explícita (`/anexar-relatorio TEST-77`) ou sem argumento — neste caso, identifique a chave do Project (TEST-XX) pelo contexto da conversa. Se não conseguir identificar, peça ao usuário.

O usuário envia, junto com a invocação ou em mensagem subsequente, um **resumo livre** do que foi feito na execução do teste. A skill anexa esse resumo como comentário no Project, formatado em ADF.

## Passo a passo

### 1. Identificar e validar a issue alvo
1. Se o usuário passou uma chave, use-a. Caso contrário, identifique pelo contexto (ex: TEST-XX já mencionado).
2. Use `mcp__Jira__getJiraIssue` para buscar a issue.
3. **Validações obrigatórias** — abortar com mensagem clara se qualquer falhar:
   - `project.key` deve ser `TEST`
   - `issuetype.name` deve ser `Project`
4. Guarde `summary` da issue para confirmar com o usuário antes de comentar.

### 2. Obter o resumo do relatório
1. Verifique se o usuário já enviou o resumo no mesmo turno da invocação (texto livre após o comando ou mensagem imediatamente anterior).
2. Caso não tenha, peça: "Envie o resumo da execução do teste para que eu anexe em [TEST-XX]".
3. Aguarde o resumo. Não invente conteúdo — usar exatamente o texto que o usuário fornecer, apenas formatando.

### 3. Identificar o executor e data
1. Use `mcp__Jira__atlassianUserInfo` para obter `displayName` do usuário autenticado.
2. Use a data atual (currentDate do contexto) no formato `YYYY-MM-DD`.

### 4. Confirmar com o usuário antes de anexar
Mostre um resumo curto:
- **Alvo:** [TEST-XX] — [summary do Project]
- **Executor:** [displayName]
- **Data:** [YYYY-MM-DD]
- **Resumo:** primeiras ~200 caracteres do texto enviado, com reticências se truncado

Pergunte: "Anexar relatório como comentário em [TEST-XX]?" e aguarde confirmação.

### 5. Anexar comentário em ADF

Use `mcp__Jira__addCommentToJiraIssue` com `contentFormat: "adf"` e o seguinte body (ADF):

1. `heading` nível 2 — `Relatório de execução`
2. `paragraph` com texto em formato `Executor: [displayName] · Data: [YYYY-MM-DD]` (use `·` ou `—` como separador)
3. `rule` (separador)
4. Conteúdo do resumo enviado pelo usuário, preservado:
   - Quebras de linha duplas → parágrafos separados
   - Listas com `-` ou `*` no início da linha → `bulletList` em ADF
   - Listas com `1.` `2.` no início da linha → `orderedList` em ADF
   - Texto comum → `paragraph`

Não inserir headings adicionais que o usuário não tenha pedido. Não reescrever o conteúdo — preservar o texto original.

### 6. Confirmar ao usuário
Informe:
- Chave do Project + link (`https://onflylabs.atlassian.net/browse/[TEST-XX]`)
- Que o relatório foi anexado como comentário

### 7. O que esta skill NÃO faz
- Não envia notificação (Google Chat ou outro)
- Não transiciona o status do Project nem da tarefa original vinculada
- Não altera a descrição do ticket
- Não cria Fix tickets nem Bugs — para isso, usar `/criar-bug`
- Não reescreve nem interpreta o conteúdo do resumo — apenas formata em ADF
