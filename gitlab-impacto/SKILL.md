---
name: gitlab-impacto
description: Consulta o card do Jira, localiza o link do GitLab, acessa as mudanças de código e gera uma análise de impacto da tarefa. Somente leitura — nenhuma alteração é feita. Aceita chave da issue como argumento (ex: /gitlab-impacto PROJ-123) ou detecta pelo contexto.
argument-hint: "[issue-key]"
---

# Skill: Análise de Impacto via GitLab

## Como usar
Invocado com chave explícita (`/gitlab-impacto PROJ-123`) ou sem argumento — neste
caso, identifique a issue pelo contexto da conversa. Se não conseguir identificar,
peça ao usuário a chave da issue.

> **Esta skill é somente leitura.** Não realiza nenhuma alteração no Jira, GitLab ou qualquer outro sistema.

---

## Passo a passo

### 1. Buscar a issue no Jira
Use `mcp__Jira__getJiraIssue` para buscar a issue. Extraia e guarde:
- Título e descrição
- Tipo da issue (bug, story, task, etc.)
- Status atual e responsável
- Qualquer link ou URL do GitLab mencionado na descrição ou campos customizados

### 2. Buscar links remotos da issue
Use `mcp__Jira__getJiraIssueRemoteIssueLinks` para listar todos os links externos
associados à issue. Procure por URLs do GitLab (gitlab.com ou instância interna),
que podem apontar para:
- **Merge Request (MR)**: `.../merge_requests/NNN`
- **Commit**: `.../commit/HASH`
- **Branch**: `.../tree/NOME-BRANCH`
- **Compare**: `.../compare/...`

Se não encontrar nenhum link, informe o usuário e encerre.
Se encontrar múltiplos links, liste-os e pergunte qual analisar.

### 3. Identificar o tipo de link e extrair informações
A partir da URL do GitLab identificada, determine o tipo e extraia os parâmetros:
- **Projeto**: `namespace/repo` (ex: `lucas.santana7/meu-projeto`)
- **Tipo**: MR, commit ou branch

Codifique o projeto para uso na API: substitua `/` por `%2F`.

Use sempre a variável de ambiente `$GITLAB_PERSONAL_ACCESS_TOKEN` nas chamadas —
**nunca exiba o valor do token** no texto da resposta ou em URLs mostradas ao usuário.

### 4. Buscar as mudanças de código no GitLab

**Se for Merge Request:**
```
GET /api/v4/projects/{project}/merge_requests/{iid}
GET /api/v4/projects/{project}/merge_requests/{iid}/changes
GET /api/v4/projects/{project}/merge_requests/{iid}/commits
```
Execute via Bash com curl usando o header `-H "PRIVATE-TOKEN: $GITLAB_PERSONAL_ACCESS_TOKEN"`.

**Se for Commit:**
```
GET /api/v4/projects/{project}/repository/commits/{sha}
GET /api/v4/projects/{project}/repository/commits/{sha}/diff
```

**Se for Branch:**
```
GET /api/v4/projects/{project}/repository/branches/{branch}
GET /api/v4/projects/{project}/repository/compare?from=main&to={branch}
```

Colete:
- Lista de arquivos alterados (com tipo: added, modified, deleted, renamed)
- Diff de cada arquivo
- Mensagens de commits envolvidos

### 5. Gerar a análise de impacto

Com base nas mudanças coletadas, produza um relatório estruturado:

---

**Análise de Impacto — [ISSUE-KEY]: [Título da issue]**

**Resumo da mudança**
[1-3 frases descrevendo o que foi feito na tarefa em linguagem de negócio]

**Arquivos alterados** (`N arquivos`)
| Arquivo | Tipo | Impacto estimado |
|---|---|---|
| `caminho/do/arquivo.ext` | modificado | Médio |
| `caminho/outro.ext` | adicionado | Baixo |

> Tipos: adicionado / modificado / removido / renomeado
> Impacto estimado: Alto (lógica crítica/core), Médio (feature/fluxo), Baixo (estilo/config/teste)

**O que foi alterado**
Para cada arquivo relevante, explique em linguagem clara:
- O que mudou (sem transcrever o diff inteiro)
- Por que essa mudança existe (correlacionando com o contexto da issue)

**Pontos de atenção**
- Liste riscos potenciais, dependências afetadas, integrações impactadas
- Indique se há alterações em banco de dados, contratos de API, permissões ou fluxos críticos
- Mencione se há testes cobrindo as mudanças ou se parece não ter cobertura

**Contexto dos commits**
- Liste os commits incluídos com hash curto e mensagem

---

### 6. Apresentar o resultado ao usuário
Exiba o relatório completo. Não faça nenhuma alteração em nenhum sistema.

Ao final, pergunte se o usuário quer aprofundar algum arquivo ou ponto específico.
