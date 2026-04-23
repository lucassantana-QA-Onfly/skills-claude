---
name: jira-testes
description: Consulta uma demanda no Jira, gera casos de teste (QA) e anexa na card. Aceita chave da issue como argumento (ex: /jira-testes PROJ-123) ou detecta pelo contexto.
argument-hint: "[issue-key]"
---

# Skill: Gerador de Casos de Teste QA

## Como usar
Invocado com chave explícita (`/jira-testes PROJ-123`) ou sem argumento — neste 
caso, identifique a issue pelo contexto da conversa. Se não conseguir identificar, 
peça ao usuário a chave da issue.

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

Use o conteúdo das imagens para enriquecer a análise da issue antes de gerar os casos de teste.

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

**c) Extraia informações relevantes para o teste:**
- Fluxos de navegação e transições de tela
- Estados dos componentes (vazio, carregado, erro, loading, disabled, hover, etc.)
- Campos de formulário, validações visuais e mensagens de erro/sucesso
- Variações de layout (mobile x desktop, se houver)
- Anotações do designer (notas, restrições, regras descritas no Figma)
- Comportamentos esperados descritos nos protótipos

Use essas informações para enriquecer a análise e gerar casos de teste mais completos e precisos.

> Se o `mcp__claude_ai_Figma__get_design_context` retornar erro de acesso, capture a screenshot com `mcp__claude_ai_Figma__get_screenshot` e use o conteúdo visual para análise.

### 2. Escolher as técnicas de teste
Antes de gerar os casos, analise o conteúdo da issue e selecione as técnicas mais adequadas ao contexto. Informe ao usuário quais técnicas foram escolhidas e por quê.

**Técnicas disponíveis:**

- **Particionamento de Equivalência**: use quando há entradas ou condições que podem ser agrupadas em classes válidas e inválidas. Evita duplicar testes dentro da mesma classe. _Aplicar quando houver campos de entrada, tipos de usuário, status, categorias._

- **Análise de Valor Limite**: use em conjunto com particionamento quando os limites entre classes são críticos (valores mínimos, máximos, logo abaixo e logo acima). _Aplicar quando houver campos numéricos, datas, quantidades, limites de caracteres._

- **Tabela de Decisão**: use quando há combinações de condições que produzem resultados diferentes. _Aplicar quando houver regras de negócio com múltiplos critérios simultâneos, fluxos condicionais complexos._

- **Fluxo principal + alternativos**: sempre inclua o happy path e as variações válidas, independentemente das técnicas escolhidas.

- **Casos de erro/exceção**: sempre inclua cenários de falha, permissão negada, dados inválidos.

### 3. Gerar os casos de teste
Com base na análise e nas técnicas selecionadas, gere os casos de teste evitando redundâncias — não crie dois testes que cobrem exatamente a mesma classe de equivalência ou o mesmo ponto de decisão.

**Ordene os casos por criticidade**, do mais crítico ao menos crítico:
1. Crítica
2. Alta
3. Média
4. Baixa

**Formato de cada caso de teste:**
```
**CT-XX: [Nome descritivo]**
- **Prioridade**: Crítica | Alta | Média | Baixa
- **Pré-condições**: [o que precisa estar configurado/existir antes]
- **Passos**:
  1. ...
  2. ...
- **Resultado esperado**: [o que deve acontecer]
```

### 4. Apresentar os casos de teste ao usuário
Exiba:
1. As técnicas escolhidas e a justificativa para cada uma
2. Todos os casos de teste gerados, ordenados por criticidade (Crítica → Alta → Média → Baixa)

### 4. Identificar alvo do anexo (Project vs issue original)

**Regra obrigatória:** casos de teste devem ser anexados no ticket **Project** (Sessão de Testes — TEST-XX) vinculado à issue, e **NÃO** na issue original (DLT/CHEER/TRPO/etc).

Localizar o Project:
1. Inspecione `issuelinks` da issue original e procure por issue com `issuetype.name === "Project"` (summary típico: `Sessão de Testes — <ISSUE-KEY>: <Título>`).
2. Se houver Project vinculado → usar sua chave como alvo do anexo (ex: `TEST-62`).
3. Se **não** houver Project vinculado → avisar o usuário e sugerir rodar `/iniciar-testes [ISSUE-KEY]` antes. Não anexar na issue original.

### 5. Perguntar antes de anexar
**SEMPRE** pergunte ao usuário antes de qualquer ação no Jira:

> "Deseja que eu anexe esses casos de teste no Project **[PROJECT-KEY]** (Sessão de Testes da **[ISSUE-KEY]**)?"

Aguarde a confirmação. Só prossiga se o usuário confirmar.

### 6. Anexar casos de teste no Project (somente após confirmação)

**6a. Verificar campos disponíveis no Project**
Use `mcp__Jira__getJiraIssueTypeMetaWithFields` (projeto TEST, issuetype `Project`) para verificar campos. Procure por um campo chamado "Caso de testes (QA)" ou similar (`customfield_*`).

**6b. Se o campo "Caso de testes (QA)" existir no Project:**
Use `mcp__Jira__editJiraIssue` no **Project** para atualizar esse campo.

**6c. Se o campo NÃO existir:**
Use `mcp__Jira__editJiraIssue` para sobrescrever a `description` do Project com os casos (ADF), OU use `mcp__Jira__addCommentToJiraIssue` no **Project** adicionando os casos como comentário.

**Nunca** anexar os casos na issue original — a tarefa original permanece limpa, com apenas o smart link do Project.

### 7. Confirmar ao usuário
Informe:
- Em qual ticket Project (TEST-XX) os casos foram anexados
- Se foram adicionados em campo customizado, descrição ou comentário
