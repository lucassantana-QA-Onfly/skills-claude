---
name: jira-testes
description: Consulta uma demanda no Jira e gera um plano de teste (escopo, valor entregue, abordagem). Não escreve casos de teste — descreve o que será testado, o valor do teste e como testar. Aceita chave da issue como argumento (ex: /jira-testes PROJ-123) ou detecta pelo contexto.
argument-hint: "[issue-key]"
---

# Skill: Plano de Teste QA

## Como usar
Invocado com chave explícita (`/jira-testes PROJ-123`) ou sem argumento — neste
caso, identifique a issue pelo contexto da conversa. Se não conseguir identificar,
peça ao usuário a chave da issue.

> **Importante:** esta skill **não escreve casos de teste detalhados** (passo a passo, pré-condições, resultado esperado de cada cenário). Ela produz um **plano de teste** com três blocos: escopo (o que será testado), valor entregue (por que vale a pena testar) e abordagem (como testar).

## Passo a passo

### 1. Buscar a issue
Use `mcp__Jira__getJiraIssue` para buscar a issue. Extraia e guarde:
- Título e descrição
- Critérios de aceitação
- Fluxos e regras de negócio descritos
- Issues linkadas relevantes

### 1b. Baixar e analisar imagens (se houver)
Verifique se a issue possui imagens na descrição ou no campo `attachment`. Se sim:

Para cada anexo de imagem, baixe com um único comando e leia diretamente:
```bash
curl -s -L -u "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" "[attachment_content_url]" -o /tmp/jira_img_[id].png
```
Em seguida use `Read` no caminho `/tmp/jira_img_[id].png` para visualizar e extrair contexto visual.

Use o conteúdo das imagens para enriquecer o plano de teste.

### 1c. Acessar links do Figma (se houver)

Verifique se a issue contém links do Figma. Procure em:
1. **Descrição da issue**: busque por URLs com padrão `figma.com/design/`, `figma.com/board/` ou `figma.com/make/`
2. **Remote links**: use `mcp__Jira__getJiraIssueRemoteIssueLinks` para listar links externos da issue — filtre por URLs de Figma

Se encontrar um ou mais links do Figma, para cada link:

**a) Extraia o `fileKey` e o `nodeId` da URL:**
- `figma.com/design/:fileKey/:fileName?node-id=:nodeId` → converta `-` em `:` no nodeId
- `figma.com/board/:fileKey/...` → use `get_figjam`
- `figma.com/make/:fileKey/...` → use o fileKey normalmente

**b) Acesse o design com `mcp__claude_ai_Figma__get_design_context`** passando `fileKey` e `nodeId`.
Se não houver nodeId, use apenas o fileKey.

**c) Extraia informações relevantes para o plano:**
- Telas e fluxos principais
- Estados dos componentes (vazio, carregado, erro, loading, disabled, etc.)
- Pontos de validação visual (mensagens, breakpoints, variações mobile/desktop)
- Anotações do designer com regras ou restrições

> Se o `mcp__claude_ai_Figma__get_design_context` retornar erro de acesso, capture a screenshot com `mcp__claude_ai_Figma__get_screenshot` e use o conteúdo visual para análise.

### 1d. Consultar Confluence OnHappy (obrigatório)

**Regra obrigatória:** sempre consultar o espaço Confluence OnHappy antes de gerar o plano, para enriquecer entendimento de fluxos, regras de negócio e contexto do que será validado.

- **Espaço:** OnHappy — https://onflylabs.atlassian.net/wiki/spaces/OnHappy/overview
- **cloudId:** `24479377-75bf-4543-a6f6-0a189a0ec825`

Passos:
1. Use `mcp__Jira__searchConfluenceUsingCql` com `space = "OnHappy"` filtrando por termos extraídos do título/descrição/AC da issue.
2. Para resultados promissores, leia com `mcp__Jira__getConfluencePage` (`contentFormat: "markdown"`) e extraia regras de negócio, integrações envolvidas, dados de exemplo.
3. Páginas-chave já mapeadas:
   - **Fluxos & Regras de negócio** — `640680814`
   - **Jornadas chaves do colaborador** — `640680327`
   - **Padrões de Projeto** — `640679944`
   - **Frontend Onhappy** — `640680037`
   - **Backend Onhappy** — `640680787`
   - **Architecture Decision Records** — `640681401`
   - **Luna V3** — `640680737`
   - **Fluxo de BUG** — `640680563`
   - **Fluxo de Incidentes** — `640680530`
4. Se a informação não estiver no Confluence, declare "não encontrei no espaço OnHappy" e siga com base no card.

### 2. Gerar o plano de teste

O plano deve ter exatamente **três blocos**, nesta ordem:

#### 2.1 — O que será testado (Escopo)
Descreva, de forma objetiva, **quais áreas, fluxos, regras ou comportamentos** entram no escopo desta validação. Inclua:
- Funcionalidades/telas/endpoints alvo
- Fluxos principais e variações relevantes
- Regras de negócio impactadas
- Plataformas/dispositivos/navegadores cobertos (quando aplicável)
- O que **fica fora** do escopo (explicitar exclusões evita ambiguidade)

> Não detalhe casos passo a passo. Foco em delimitar superfície de teste.

#### 2.2 — Valor entregue pelo teste
Explique **por que testar isso importa**. Conecte o teste a risco, qualidade ou objetivo de produto. Inclua:
- Risco mitigado caso a validação seja feita (ex: regressão em fluxo crítico, bug em produção, vazamento de dado, queda de receita)
- Impacto no usuário final ou no negócio se o defeito passar
- Confiança ganha para release/decisão (ex: "valida correção do logout antes de promover para prod")
- Áreas de regressão protegidas

> Foco em "por que vale a pena gastar X horas testando isso".

#### 2.3 — Como testar (Abordagem)
Descreva **como a validação será conduzida**. Inclua:
- Estratégia geral (manual, automatizado, exploratório, regressão dirigida)
- Técnicas aplicáveis (particionamento, valor limite, tabela de decisão, fluxo + alternativos, erro/exceção, exploratório)
- Ambiente(s) (homol, prod, staging) e dados de teste necessários
- Ferramentas (Postman/Bruno, ADB, BrowserStack, Charles, dispositivos físicos, etc.)
- Pré-condições gerais (acessos, contas, configs) — sem virar receita passo a passo
- Critério de saída (quando consideramos "testado")

> Mantenha em nível de plano. Não escreva casos de teste estruturados (CT-XX).

### 3. Formato de saída

```markdown
# Plano de Teste — [ISSUE-KEY]: [Título da issue]

## O que será testado
[parágrafo + bullets do escopo]

## Valor entregue pelo teste
[parágrafo + bullets do valor / risco mitigado]

## Como testar
[parágrafo descrevendo abordagem + bullets com técnicas, ambiente, ferramentas, dados, critério de saída]
```

### 4. Apresentar ao usuário
Exiba o plano completo no formato acima. **Não inclua casos de teste estruturados (CT-XX) nem cenários Gherkin.** Se o usuário pedir explicitamente casos detalhados, redirecione: "esta skill produz plano de teste; para cenários estruturados, peça explicitamente."

### 5. Identificar alvo do anexo (Project vs issue original)

**Regra obrigatória:** o plano deve ser anexado no ticket **Project** (Sessão de Testes — TEST-XX) vinculado à issue, e **NÃO** na issue original (DLT/CHEER/TRPO/etc).

Localizar o Project:
1. Inspecione `issuelinks` da issue original e procure por issue com `issuetype.name === "Project"` (summary típico: `Sessão de Testes — <ISSUE-KEY>: <Título>`).
2. Se houver Project vinculado → usar sua chave como alvo do anexo (ex: `TEST-62`).
3. Se **não** houver Project vinculado → avisar o usuário e sugerir rodar `/iniciar-testes [ISSUE-KEY]` antes. Não anexar na issue original.

### 6. Perguntar antes de anexar
**SEMPRE** pergunte ao usuário antes de qualquer ação no Jira:

> "Deseja que eu anexe esse plano de teste no Project **[PROJECT-KEY]** (Sessão de Testes da **[ISSUE-KEY]**)?"

Aguarde a confirmação. Só prossiga se o usuário confirmar.

### 7. Anexar plano no Project (somente após confirmação)

Use `mcp__Jira__editJiraIssue` para sobrescrever a `description` do Project com o plano (ADF), OU use `mcp__Jira__addCommentToJiraIssue` no **Project** adicionando o plano como comentário.

**Nunca** anexar na issue original — a tarefa original permanece limpa, apenas com o smart link do Project.

### 8. Confirmar ao usuário
Informe:
- Em qual ticket Project (TEST-XX) o plano foi anexado
- Se foi adicionado em descrição ou comentário
