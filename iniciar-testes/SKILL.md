---
name: iniciar-testes
description: Cria um ticket do tipo Project no projeto TEST vinculado à tarefa original, popula o campo Effort com a estimativa de teste e atribui ao usuário que invocou a skill. Se não houver estimativa disponível no contexto, invoca /jira-estimativa automaticamente. Aceita chave da issue como argumento (ex: /iniciar-testes PROJ-123) ou detecta pelo contexto.
argument-hint: "[issue-key]"
---

# Skill: Iniciar Testes

## Como usar
Invocado com chave explícita (`/iniciar-testes PROJ-123`) ou sem argumento — neste caso, identifique a issue pelo contexto da conversa. Se não conseguir identificar, peça ao usuário a chave da issue.

## Passo a passo

### 1. Buscar a issue original
Use `mcp__Jira__getJiraIssue` para buscar a issue. Extraia e guarde:
- Chave da issue (ex: `DLT-123`)
- Título (summary)

### 2. Identificar o usuário que está invocando a skill
Use `mcp__Jira__atlassianUserInfo` para obter os dados do usuário autenticado. Guarde o `accountId` para usar como responsável no ticket Project.

### 3. Verificar se há estimativa de esforço disponível

Verifique se no contexto da conversa já existe um valor de **E total** gerado pela skill `/jira-estimativa` (soma das estimativas PERT das atividades de teste, em horas).

**Se houver:** use esse valor numérico para o campo `customfield_11756` (Effort).

**Se não houver:** invoque a skill `/jira-estimativa` passando a chave da issue. Aguarde a estimativa ser gerada e exibida. Em seguida, extraia o valor total de E da tabela PERT (campo "Total" da coluna `E = (O+4M+P)/6`) e use-o como valor do Effort.

### 4. Confirmar com o usuário antes de criar

Exiba um resumo do que será criado:
- **Projeto:** TEST
- **Tipo:** Project
- **Título:** `Sessão de Testes — [ISSUE-KEY]: [título da issue]`
- **Responsável:** nome do usuário identificado no passo 2
- **Effort:** valor em horas obtido no passo 3
- **Vínculo:** "relates to" com [ISSUE-KEY]

Pergunte se pode prosseguir antes de executar qualquer ação no Jira.

### 5. Criar o ticket Project no projeto TEST

Use `mcp__Jira__createJiraIssue` com os seguintes campos:

- **project**: `TEST`
- **issuetype**: `Project` (id `11106`)
- **summary**: `Sessão de Testes — [ISSUE-KEY]: [título da issue original]`
- **assignee**: `accountId` obtido no passo 2
- **customfield_11756**: valor numérico do E total em horas (apenas o número, sem "h")
- **Categorias** (`customfield_*` correspondente): defina o valor conforme o projeto da issue original:
  - Issue do projeto **DLT** → valor `onhappy-mobile`
  - Issue do projeto **CHEER** → valor `onhappy-web`
  - Outros projetos → omitir o campo

  > Para descobrir o ID correto do campo e os IDs dos valores, use `mcp__Jira__getJiraIssueTypeMetaWithFields` no projeto TEST antes de criar o ticket.

### 6. Vincular o Project à issue original

Use `mcp__Jira__createIssueLink` com:
- Tipo: `"relates to"` (ou o tipo disponível mais próximo)
- `inwardIssue`: chave do ticket Project criado
- `outwardIssue`: chave da issue original

### 7. Gerar e anexar casos de teste no Project

**Sempre** invocar `/jira-testes [ISSUE-KEY]` para gerar os casos de teste estruturados (técnicas: equivalência, valor limite, erro, regressão, exploratórios, etc.), **independentemente** da issue original ter AC ou não.

**Regra obrigatória:** os casos gerados devem ser anexados **no ticket Project (TEST-XX) criado neste fluxo**, e **NÃO** na issue original. A tarefa original permanece limpa (apenas com o smart link do Project gerado pelo vínculo do passo 6).

Não incluir checklist de critérios na descrição do Project — apenas os casos de teste gerados via `/jira-testes`.

Fluxo: criar Project vazio no passo 5 → vincular no passo 6 → invocar `/jira-testes` → anexar retorno via `mcp__Jira__editJiraIssue` na descrição do Project (ADF) ou via `mcp__Jira__addCommentToJiraIssue` no Project.

### 8. Transicionar a issue original para validação de QA

Após todos os passos anteriores concluídos com sucesso, mova a **issue original** para a coluna de validação de QA.

1. Use `mcp__Jira__getTransitionsForJiraIssue` na issue original para listar as transições disponíveis.
2. Procure por uma transição cujo status de destino tenha nome compatível com validação de QA, considerando variações. Critério de match (case-insensitive, acentuação ignorada):
   - Contém `qa validation`, `in validation`, `em validação`, `validação qa`, `validação de qa`, `qa em andamento`, `qa in progress`, `testing`, `in testing`, `em teste`, `em testes`
3. Se houver exatamente uma transição candidata, aplique-a via `mcp__Jira__transitionJiraIssue`.
4. Se houver **múltiplas** candidatas, prefira nesta ordem: `QA VALIDATION` > `In Validation` > `Em Validação` > `In Testing` > `Em Teste`. Informe ao usuário qual foi aplicada.
5. Se **não** houver nenhuma transição compatível, informe o usuário (com a lista de transições disponíveis) em vez de falhar silenciosamente — não aplique nenhuma transição nesse caso.

### 9. Confirmar ao usuário

Informe de forma resumida:
- Chave e link do ticket Project criado
- Que o vínculo com a issue original foi registrado
- Que o campo Effort foi preenchido com o valor em horas
- Que a descrição foi preenchida com critérios de aceite / plano de teste
- Que a issue original foi movida para a coluna de validação de QA (com o nome do status aplicado) — ou aviso se não foi possível
