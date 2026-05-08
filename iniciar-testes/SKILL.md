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
- **Resumo do problema** (1–3 linhas extraídas da descrição/contexto)
- **Resultado esperado** (critério de aceite principal ou comportamento esperado)

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
- **Vínculo:** Project **tests** [ISSUE-KEY] (tipo "Tests")

Pergunte se pode prosseguir antes de executar qualquer ação no Jira.

### 5. Criar o ticket Project no projeto TEST

Use `mcp__Jira__createJiraIssue` com os seguintes campos:

- **project**: `TEST`
- **issuetype**: `Project` (id `11106`)
- **summary**: `Sessão de Testes — [ISSUE-KEY]: [título da issue original]`
- **assignee**: `accountId` obtido no passo 2
- **customfield_11756**: valor numérico do E total em horas (apenas o número, sem "h")
- **description**: **NÃO** preencher neste passo. A descrição será gerada e anexada exclusivamente pela skill `/jira-testes` no passo 7. Crie o ticket sem o campo `description` (ou com string vazia).
- **Categorias** (`customfield_*` correspondente): defina o valor conforme o projeto da issue original:
  - Issue do projeto **DLT** → valor `onhappy-mobile`
  - Issue do projeto **CHEER** → valor `onhappy-web`
  - Outros projetos → omitir o campo

  > Para descobrir o ID correto do campo e os IDs dos valores, use `mcp__Jira__getJiraIssueTypeMetaWithFields` no projeto TEST antes de criar o ticket.

### 6. Vincular o Project à issue original

Use `mcp__Jira__createIssueLink` com tipo **"Tests"** (direção: o Project **tests** a issue original):
- Tipo: `"Tests"` (se não existir, usar `mcp__Jira__getIssueLinkTypes` para descobrir o nome correto do link "tests / is tested by"; cair em `"Relates"` apenas se nenhum tipo de teste estiver disponível)
- `inwardIssue`: chave do ticket Project criado (lado que **testa**)
- `outwardIssue`: chave da issue original (lado que **é testado**)

> Se a relação no Jira for invertida (inward = "is tested by"), troque os campos para manter a semântica "Project tests Tarefa original".

### 7. Gerar e anexar a descrição via `/jira-testes` (obrigatório, sempre)

**Regra obrigatória:** a descrição do Project é gerada **sempre** pela skill `/jira-testes` — nunca pelo `/iniciar-testes` diretamente. Este passo acontece **logo após** criar o Project e o vínculo, antes de qualquer ação de pipeline.

Invoque `/jira-testes [ISSUE-KEY]` (chave da issue **original**, não a do Project). A própria skill cuida de:
- Localizar o Project vinculado (TEST-XX)
- Gerar o plano (Motivação & Risco, Escopo, Valor, Como testar)
- Anexar tudo na descrição do Project (sobrescrevendo, com Resumo do problema + Resultado esperado no topo)

Não duplicar a lógica aqui — apenas chamar a skill e aguardar a confirmação de anexação. Se o usuário recusar a anexação no fluxo do `/jira-testes`, **pare** e avise — o Project ficará sem descrição até que rode `/jira-testes` manualmente.

### 8. Criar pipeline de ambiente de review no GitLab

Após criar o Project e o vínculo, suba um ambiente de review automatizado disparando uma pipeline no repositório `onflylabs/onhappy/review`.

#### 8.1 — Calcular o `name` do ambiente
Pegue a chave da issue original e normalize:
- Lowercase
- Remover qualquer caractere que não seja `[a-z0-9]` (hífens, espaços, símbolos)
- Ex: `CHEER-3060` → `cheer3060`, `DLT-170` → `dlt170`

#### 8.2 — Identificar branches Frontend e Backend
Localize as branches de desenvolvimento vinculadas à issue original (aba **Desenvolvimento → Ramificações** no Jira, alimentada pelo GitLab dev panel).

1. Tente extrair as branches via dev panel da issue (campo `customfield_10000` / development summary) ou via `mcp__claude_ai_Gitlab_MCP__search` filtrando por chave da issue.
2. Mapeie por projeto GitLab:
   - **OnHappy Frontend** → valor da variável `frontend`
   - **OnHappy Backend** → valor da variável `backend`
3. Se a issue só tem branch em um dos lados (só Frontend ou só Backend), o lado ausente recebe o valor padrão `main`.
4. Se nenhum lado tiver branch, avise o usuário e pare — não criar pipeline sem nenhuma branch de feature, pois subiria um ambiente igual à main.

#### 8.3 — Perguntar estratégia de dados
Pergunte ao usuário:
> "Reutilizar o banco de dados da main neste ambiente de review? (sim/não)"

- **Sim** → adicionar variável `DATA_STRATEGY=reuse` à pipeline.
- **Não** → não enviar `DATA_STRATEGY` (deixar vazio / ausente).

#### 8.4 — Confirmar configuração antes de criar
Mostre ao usuário um resumo:
- **Repo:** `onflylabs/onhappy/review`
- **Branch da pipeline:** `main`
- **name:** `[name calculado]`
- **frontend:** `[branch ou main]`
- **backend:** `[branch ou main]`
- **DATA_STRATEGY:** `reuse` ou `(omitido)`

Aguarde confirmação. Só prossiga se confirmar.

#### 8.5 — Disparar a pipeline (via curl + variável de ambiente)

**Não usar o MCP `manage_pipeline`** — o parâmetro `variables` desse MCP está quebrado e rejeita qualquer formato, impedindo o envio de `DATA_STRATEGY`. Disparar sempre via `curl` direto na API do GitLab usando o PAT da variável de ambiente do usuário.

Variável de ambiente esperada (User scope no Windows):
- `GITLAB_PERSONAL_ACCESS_TOKEN` — PAT com escopo `api`

> **Nunca** interpolar o token em texto, logs ou URLs. Sempre passá-lo via header `PRIVATE-TOKEN` lendo direto de `$env:GITLAB_PERSONAL_ACCESS_TOKEN` (PowerShell) ou `[Environment]::GetEnvironmentVariable('GITLAB_PERSONAL_ACCESS_TOKEN','User')` quando a variável não estiver herdada na sessão.

Comando PowerShell (preferencial neste ambiente):

```powershell
$pat = [Environment]::GetEnvironmentVariable('GITLAB_PERSONAL_ACCESS_TOKEN','User')
$body = @{
  ref       = 'main'
  inputs    = @{ name = '<name>'; frontend = '<branch ou main>'; backend = '<branch ou main>' }
  variables = @(@{ key = 'DATA_STRATEGY'; value = 'reuse' })  # remova esta linha se DATA_STRATEGY foi descartado em 8.3
} | ConvertTo-Json -Depth 5 -Compress
Invoke-RestMethod -Method Post `
  -Uri 'https://gitlab.com/api/v4/projects/onflylabs%2Fonhappy%2Freview/pipeline' `
  -Headers @{ 'PRIVATE-TOKEN' = $pat; 'Content-Type' = 'application/json' } `
  -Body $body | ConvertTo-Json -Depth 5
```

Equivalente em bash/curl (caso a sessão estiver em shell POSIX e o PAT já exportado):

```bash
curl --silent --show-error --fail \
  --header "PRIVATE-TOKEN: $GITLAB_PERSONAL_ACCESS_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"ref":"main","inputs":{"name":"<name>","frontend":"<branch>","backend":"<branch>"},"variables":[{"key":"DATA_STRATEGY","value":"reuse"}]}' \
  "https://gitlab.com/api/v4/projects/onflylabs%2Fonhappy%2Freview/pipeline"
```

Da resposta JSON, guardar `id` (pipeline_id) e `web_url` para os próximos sub-passos.

Tratamento de erros comuns:
- **`insufficient_scope`** → o PAT não tem escopo `api` completo. Avisar o usuário e parar — não tentar workaround.
- **Variável de ambiente ausente** → avisar o usuário para configurar `GITLAB_PERSONAL_ACCESS_TOKEN` no escopo User do Windows e rodar de novo.

> URL do projeto: https://gitlab.com/onflylabs/onhappy/review
> URL da listagem de pipelines: https://gitlab.com/onflylabs/onhappy/review/-/pipelines

#### 8.6 — Aguardar a pipeline finalizar

Polling do status da pipeline pelo `pipeline_id` retornado em 8.5, via curl + PAT:

```powershell
$pat = [Environment]::GetEnvironmentVariable('GITLAB_PERSONAL_ACCESS_TOKEN','User')
Invoke-RestMethod -Method Get `
  -Uri "https://gitlab.com/api/v4/projects/onflylabs%2Fonhappy%2Freview/pipelines/<pipeline_id>" `
  -Headers @{ 'PRIVATE-TOKEN' = $pat }
```

Repetir até `status` ser `success`, `failed` ou `canceled`.

- Se **success** → seguir para 8.7.
- Se **failed/canceled** → informar o usuário com `web_url` da pipeline e parar (não anexar URL nem prosseguir para o próximo passo).

#### 8.7 — Anexar URL do ambiente no Project
Monte a URL do ambiente deployado:
```
https://app.<name>.review.onhappy.com.br
```
Anexe como comentário ADF no Project (TEST-XX) via `mcp__Jira__addCommentToJiraIssue` com `contentFormat: "adf"`. Estrutura do body:
1. `heading` nível 2 — `Ambiente de review`
2. `paragraph` com link clicável: `https://app.<name>.review.onhappy.com.br`
3. `paragraph` opcional com link da pipeline GitLab usada para deploy

### 9. Transicionar a issue original para validação de QA

Após todos os passos anteriores concluídos com sucesso, mova a **issue original** para a coluna de validação de QA.

1. Use `mcp__Jira__getTransitionsForJiraIssue` na issue original para listar as transições disponíveis.
2. Procure por uma transição cujo status de destino tenha nome compatível com validação de QA, considerando variações. Critério de match (case-insensitive, acentuação ignorada):
   - Contém `qa validation`, `in validation`, `em validação`, `validação qa`, `validação de qa`, `qa em andamento`, `qa in progress`, `testing`, `in testing`, `em teste`, `em testes`
3. Se houver exatamente uma transição candidata, aplique-a via `mcp__Jira__transitionJiraIssue`.
4. Se houver **múltiplas** candidatas, prefira nesta ordem: `QA VALIDATION` > `In Validation` > `Em Validação` > `In Testing` > `Em Teste`. Informe ao usuário qual foi aplicada.
5. Se **não** houver nenhuma transição compatível, informe o usuário (com a lista de transições disponíveis) em vez de falhar silenciosamente — não aplique nenhuma transição nesse caso.

### 10. Confirmar ao usuário

Informe de forma resumida:
- Chave e link do ticket Project criado
- Que o vínculo com a issue original foi registrado
- Que o campo Effort foi preenchido com o valor em horas
- URL do ambiente de review deployado (`https://app.<name>.review.onhappy.com.br`) ou aviso se a pipeline falhou
- Que a descrição foi preenchida com critérios de aceite / plano de teste
- Que a issue original foi movida para a coluna de validação de QA (com o nome do status aplicado) — ou aviso se não foi possível
