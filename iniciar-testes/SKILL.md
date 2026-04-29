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
- **description** (ADF): apenas duas seções, nesta ordem:
  1. Heading **"Resumo do problema"** + parágrafo com o resumo extraído no passo 1.
  2. Heading **"Resultado esperado"** + parágrafo com o resultado esperado extraído no passo 1.

  Não incluir casos de teste, checklist de AC, passo a passo de reprodução, links extras nem outras seções na descrição. Casos de teste serão anexados separadamente no passo 7.
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

### 7. Criar pipeline de ambiente de review no GitLab

Após criar o Project e o vínculo, suba um ambiente de review automatizado disparando uma pipeline no repositório `onflylabs/onhappy/review`.

#### 7.1 — Calcular o `name` do ambiente
Pegue a chave da issue original e normalize:
- Lowercase
- Remover qualquer caractere que não seja `[a-z0-9]` (hífens, espaços, símbolos)
- Ex: `CHEER-3060` → `cheer3060`, `DLT-170` → `dlt170`

#### 7.2 — Identificar branches Frontend e Backend
Localize as branches de desenvolvimento vinculadas à issue original (aba **Desenvolvimento → Ramificações** no Jira, alimentada pelo GitLab dev panel).

1. Tente extrair as branches via dev panel da issue (campo `customfield_10000` / development summary) ou via `mcp__claude_ai_Gitlab_MCP__search` filtrando por chave da issue.
2. Mapeie por projeto GitLab:
   - **OnHappy Frontend** → valor da variável `frontend`
   - **OnHappy Backend** → valor da variável `backend`
3. Se a issue só tem branch em um dos lados (só Frontend ou só Backend), o lado ausente recebe o valor padrão `main`.
4. Se nenhum lado tiver branch, avise o usuário e pare — não criar pipeline sem nenhuma branch de feature, pois subiria um ambiente igual à main.

#### 7.3 — Perguntar estratégia de dados
Pergunte ao usuário:
> "Reutilizar o banco de dados da main neste ambiente de review? (sim/não)"

- **Sim** → adicionar variável `DATA_STRATEGY=reuse` à pipeline.
- **Não** → não enviar `DATA_STRATEGY` (deixar vazio / ausente).

#### 7.4 — Confirmar configuração antes de criar
Mostre ao usuário um resumo:
- **Repo:** `onflylabs/onhappy/review`
- **Branch da pipeline:** `main`
- **name:** `[name calculado]`
- **frontend:** `[branch ou main]`
- **backend:** `[branch ou main]`
- **DATA_STRATEGY:** `reuse` ou `(omitido)`

Aguarde confirmação. Só prossiga se confirmar.

#### 7.5 — Disparar a pipeline
Use `mcp__claude_ai_Gitlab_MCP__manage_pipeline` (action `create`) no projeto `onflylabs/onhappy/review`, branch `main`, passando as variáveis acima como inputs.

> URL do projeto: https://gitlab.com/onflylabs/onhappy/review
> URL da listagem de pipelines: https://gitlab.com/onflylabs/onhappy/review/-/pipelines

#### 7.6 — Aguardar a pipeline finalizar
Faça polling do status da pipeline (via `mcp__claude_ai_Gitlab_MCP__get_pipeline_jobs` ou refetch via `manage_pipeline`) até `status` ser `success`, `failed` ou `canceled`.

- Se **success** → seguir para 7.7.
- Se **failed/canceled** → informar o usuário com o link da pipeline e parar (não anexar URL nem prosseguir para o próximo passo).

#### 7.7 — Anexar URL do ambiente no Project
Monte a URL do ambiente deployado:
```
https://app.<name>.review.onhappy.com.br
```
Anexe como comentário ADF no Project (TEST-XX) via `mcp__Jira__addCommentToJiraIssue` com `contentFormat: "adf"`. Estrutura do body:
1. `heading` nível 2 — `Ambiente de review`
2. `paragraph` com link clicável: `https://app.<name>.review.onhappy.com.br`
3. `paragraph` opcional com link da pipeline GitLab usada para deploy

### 8. Anexar plano de teste via `/jira-testes`

Invoque `/jira-testes [ISSUE-KEY]` e delegue toda a responsabilidade de gerar e anexar o conteúdo de teste ao ticket Project. A própria skill `/jira-testes` cuida de:
- Gerar o plano (escopo, valor entregue, abordagem)
- Localizar o Project vinculado
- Anexar no Project (descrição)

Não duplicar a lógica aqui — apenas chamar a skill.

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
