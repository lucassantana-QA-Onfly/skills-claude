---
name: iniciar-testes
description: Cria um ticket do tipo Project no projeto TEST vinculado à tarefa original, popula o campo Story point estimate (em pontos, via escala Fibonacci a partir da estimativa PERT) e atribui ao usuário que invocou a skill. Se não houver estimativa disponível no contexto, invoca /jira-estimativa automaticamente. Aceita chave da issue como argumento (ex: /iniciar-testes PROJ-123) ou detecta pelo contexto.
argument-hint: "[issue-key]"
---

# Skill: Iniciar Testes

## Como usar
Invocado com chave explícita (`/iniciar-testes PROJ-123`) ou sem argumento — neste caso, identifique a issue pelo contexto da conversa. Se não conseguir identificar, peça ao usuário a chave da issue.

## Passo a passo

### 1. Buscar a issue original
Use `Atlassian:getJiraIssue` para buscar a issue. Extraia e guarde:
- Chave da issue (ex: `DLT-123`)
- Título (summary)

### 2. Identificar o usuário que está invocando a skill
Use `Atlassian:atlassianUserInfo` para obter os dados do usuário autenticado. Guarde o `accountId` para usar como responsável no ticket Project.

### 3. Verificar se há estimativa de esforço disponível

Verifique se no contexto da conversa já existe um valor de **E total** gerado pela skill `/jira-estimativa` (soma das estimativas PERT das atividades de teste, em horas).

**Se houver:** use esse valor de E total (em horas) como base para calcular o valor em Story Points.

**Se não houver:** invoque a skill `/jira-estimativa` passando a chave da issue. Aguarde a estimativa ser gerada e exibida. Em seguida, extraia o valor total de E da tabela PERT (campo "Total" da coluna `E = (O+4M+P)/6`).

### 3b. Converter horas em Story Points (escala Fibonacci)

Use a tabela abaixo para converter o E total (em horas) em pontos:

| E total (horas) | Story Points |
|---|---|
| ≤ 2h | 1 |
| ≤ 4h | 2 |
| ≤ 8h | 3 |
| ≤ 16h | 5 |
| ≤ 32h | 8 |
| ≤ 56h | 13 |
| > 56h | 21 |

### 3c. Descobrir o ID do campo Story point estimate

Use `Atlassian:getJiraIssueTypeMetaWithFields` no projeto **TEST** para o tipo **Project** (id `11106`) e localize o ID do custom field cujo nome é **"Story point estimate"** (ou variação como "Story Points"). Guarde o `customfield_*` correspondente.

### 4. Confirmar com o usuário antes de criar

Exiba um resumo do que será criado:
- **Projeto:** TEST
- **Tipo:** Project
- **Título:** `Sessão de Testes — [ISSUE-KEY]: [título da issue]`
- **Responsável:** nome do usuário identificado no passo 2
- **Story Points:** valor em pontos obtido no passo 3b (indique também o E total em horas que originou o ponto)
- **Vínculo:** "relates to" com [ISSUE-KEY]

Pergunte se pode prosseguir antes de executar qualquer ação no Jira.

### 5. Criar o ticket Project no projeto TEST

Use `Atlassian:createJiraIssue` com os seguintes campos:

- **project**: `TEST`
- **issuetype**: `Project` (id `11106`)
- **summary**: `Sessão de Testes — [ISSUE-KEY]: [título da issue original]`
- **assignee**: `accountId` obtido no passo 2
- **Story point estimate** (`customfield_*` descoberto no passo 3c): valor numérico em pontos obtido no passo 3b (apenas o número, sem "pts")
- **Categorias** (`customfield_*` correspondente): defina o valor conforme o projeto da issue original:
  - Issue do projeto **DLT** → valor `onhappy-mobile`
  - Issue do projeto **CHEER** → valor `onhappy-web`
  - Outros projetos → omitir o campo

  > Para descobrir o ID correto do campo e os IDs dos valores, use `Atlassian:getJiraIssueTypeMetaWithFields` no projeto TEST antes de criar o ticket.

### 6. Vincular o Project à issue original

Use `Atlassian:createIssueLink` com:
- Tipo: `"relates to"` (ou o tipo disponível mais próximo)
- `inwardIssue`: chave do ticket Project criado
- `outwardIssue`: chave da issue original

### 7. Adicionar plano de teste na descrição do ticket

Verifique se a issue original possui um plano de teste na descrição (seção como "Test Plan", "Plano de Teste" ou similar).

**Se houver:** copie o conteúdo do plano de teste e cole na descrição do ticket Project criado, usando `Atlassian:editJiraIssue` com `contentFormat: "adf"`.

**Se não houver:** invoque a skill `/jira-testes` passando a chave da issue original para gerar um plano de teste. Após a geração, cole o resultado na descrição do ticket Project criado da mesma forma.

### 8. Confirmar ao usuário

Informe de forma resumida:
- Chave e link do ticket Project criado
- Que o vínculo com a issue original foi registrado
- Que o campo Story point estimate foi preenchido com o valor em pontos (Fibonacci, derivado do E total em horas)
- Que a descrição foi preenchida com o plano de teste
