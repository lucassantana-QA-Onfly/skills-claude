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

### 4. Perguntar antes de anexar
**SEMPRE** pergunte ao usuário antes de qualquer ação no Jira:

> "Deseja que eu anexe esses casos de teste na card **[ISSUE-KEY]**?"

Aguarde a confirmação. Só prossiga se o usuário confirmar.

### 5. Anexar casos de teste na card (somente após confirmação)

**5a. Verificar campos disponíveis**
Use `mcp__Jira__getJiraIssueTypeMetaWithFields` para verificar os campos da issue.
Procure por um campo chamado "Caso de testes (QA)" ou similar (`customfield_*`).

**5b. Se o campo "Caso de testes (QA)" existir:**
Use `mcp__Jira__editJiraIssue` para atualizar esse campo com os casos de teste.

**5c. Se o campo NÃO existir:**
Use `mcp__Jira__addCommentToJiraIssue` para adicionar os casos de teste como 
comentário na issue, com o cabeçalho:

> **📋 Casos de Teste (QA)**
> _Campo "Caso de testes (QA)" não encontrado. Casos adicionados via comentário._

### 6. Confirmar ao usuário
Informe o que foi feito:
- Se os casos de teste foram adicionados ao campo ou ao comentário
