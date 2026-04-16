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

### 6. Vincular o Project à issue original

Use `mcp__Jira__createIssueLink` com:
- Tipo: `"relates to"` (ou o tipo disponível mais próximo)
- `inwardIssue`: chave do ticket Project criado
- `outwardIssue`: chave da issue original

### 7. Adicionar plano de teste na descrição do ticket

Verifique se a issue original possui um plano de teste na descrição (seção como "Test Plan", "Plano de Teste" ou similar).

**Se houver:** copie o conteúdo do plano de teste e cole na descrição do ticket Project criado, usando `mcp__Jira__editJiraIssue` com `contentFormat: "adf"`.

**Se não houver:** invoque a skill `/jira-testes` passando a chave da issue original para gerar um plano de teste. Após a geração, cole o resultado na descrição do ticket Project criado da mesma forma.

### 8. Confirmar ao usuário

Informe de forma resumida:
- Chave e link do ticket Project criado
- Que o vínculo com a issue original foi registrado
- Que o campo Effort foi preenchido com o valor em horas
- Que a descrição foi preenchida com o plano de teste
