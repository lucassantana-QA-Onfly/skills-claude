---
name: jira-testes
description: Consulta uma demanda no Jira, gera plano de teste (QA) e anexa na descrição do Project (TEST-XX). Aceita chave da issue como argumento (ex: /jira-testes PROJ-123) ou detecta pelo contexto.
argument-hint: "[issue-key]"
---

# Skill: Gerador de Plano de Teste QA

## Como usar
Invocado com chave explícita (`/jira-testes PROJ-123`) ou sem argumento — neste
caso, identifique a issue pelo contexto da conversa. Se não conseguir identificar,
peça ao usuário a chave da issue.

## Passo a passo

### 1. Buscar a issue
Use `Atlassian:getJiraIssue` para buscar a issue. Extraia e guarde:
- Título e descrição
- Critérios de aceitação
- Fluxos e regras de negócio descritos
- Issues linkadas relevantes
- Subtarefas (para delimitar escopo da entrega)

### 1b. Baixar e analisar imagens (se houver)
Verifique se a issue possui imagens na descrição ou no campo `attachment`. Se sim:

Para cada anexo de imagem, baixe com um único comando e leia diretamente:
```bash
curl -s -L -u "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" "[attachment_content_url]" -o /tmp/jira_img_[id].png
```
Em seguida use `Read` no caminho `/tmp/jira_img_[id].png` para visualizar e extrair contexto visual.

Use o conteúdo das imagens para enriquecer a análise da issue antes de gerar o plano de teste.

### 1c. Acessar links do Figma (se houver)

Verifique se a issue contém links do Figma. Procure em:
1. **Descrição da issue**: busque por URLs com padrão `figma.com/design/`, `figma.com/board/` ou `figma.com/make/`
2. **Remote links**: use `Atlassian:getJiraIssueRemoteIssueLinks` para listar links externos da issue — filtre por URLs de Figma

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

Use essas informações para enriquecer a análise.

> Se o `mcp__claude_ai_Figma__get_design_context` retornar erro de acesso, capture a screenshot com `mcp__claude_ai_Figma__get_screenshot` e use o conteúdo visual para análise.

### 1d. Consultar Confluence OnHappy (se contexto OnHappy)
Para issues dos projetos OnHappy (CHEER, DLT, TRPO etc.), consulte o espaço OnHappy no Confluence quando houver dúvida sobre regra de negócio, fluxo ou padrão. Confluence é fonte de verdade para esse contexto.

### 1e. Consultar GitLab (se houver MR/branch vinculado)
Se a issue tiver MR/branch vinculado, considere invocar `/gitlab-impacto` ou consultar diretamente para entender a mudança real de código antes de descrever ambiente e pontos de atenção.

### 2. Gerar o plano de teste

O plano deve ser narrativo e estruturado por **headings**, no padrão abaixo. Não use formato CT-XX, não liste técnicas formais (Particionamento, Valor Limite etc.) explicitamente — incorpore o raciocínio no conteúdo.

#### Estrutura obrigatória (nesta ordem)

**Resumo do problema**
Parágrafo curto descrevendo o que a tarefa muda no produto: onde acontece, qual o comportamento atual (se for bug) ou o que está sendo entregue (se for feature), e — quando aplicável — a causa raiz/abordagem técnica. Mencione efeitos colaterais conhecidos (outros componentes/módulos afetados pela mesma alteração).

**Resultado esperado**
Parágrafo descrevendo o comportamento final esperado em produção após a entrega: quando algo deve aparecer/funcionar, quando deve permanecer oculto/inalterado, e o que **não** deve regredir.

**Motivação & Risco**
- **Motivação**: por que o teste é necessário (bug em produção, requisito de compliance, risco operacional, etc.).
- **Risco se não testar**: consequência concreta (impacto financeiro, SLA, segurança, experiência do usuário).
- **Probabilidade × Impacto**: classificar ambos (Baixa/Média/Alta) com justificativa curta.
- **Áreas adjacentes em risco**: componentes/módulos que podem ter sido afetados indiretamente pela mesma mudança.

**O que será testado**
Lista enxuta (bullets) das frentes de validação que serão cobertas. Inclua sempre uma linha final **"Fora do escopo:"** delimitando o que **não** será testado nessa sessão (com justificativa, ex.: backend, painel admin, módulo X).

**Valor entregue pelo teste**
Bullets curtos, em linguagem de negócio, descrevendo o que o teste devolve à entrega (mitigação de regressão, segurança, previsibilidade, confiança no release). Evite jargão técnico aqui.

**Como testar**
Bloco operacional com:
- **Ambiente**: URL/app/build específico (ex.: review do MR, TestFlight, APK, produção).
- **Dados/Pré-condições**: dados necessários (usuário, trip, registro X em estado Y, feature flag ativa, etc.).
- **Pontos de atenção**: combinações sensíveis, estados de transição (loading/erro), breakpoints, navegação, console limpo.
- **Critério de saída**: o que define "teste concluído com sucesso" (AC cumpridos, sem regressão em X, console limpo, etc.).

### 3. Apresentar o plano ao usuário
Exiba o plano completo no chat, seguindo a estrutura acima, antes de qualquer ação no Jira.

### 4. Perguntar antes de anexar
**SEMPRE** pergunte antes de qualquer alteração no Jira:

> "Deseja que eu anexe esse plano na **descrição do TEST-XX**?"

Se o usuário ainda não tem um Project vinculado, oriente-o a rodar `/iniciar-testes [ISSUE-KEY]` primeiro. **Nunca** anexe o plano na issue original (DLT/CHEER/TRPO etc.) — sempre no Project (TEST-XX).

Aguarde a confirmação. Só prossiga se o usuário confirmar.

### 5. Anexar o plano na descrição do Project (somente após confirmação)

Use `Atlassian:editJiraIssue` com `contentFormat: "adf"` para escrever no campo `description` do **TEST-XX** vinculado.

- Cada seção da estrutura deve virar um **heading** no ADF (`heading` level 2 ou 3).
- Liste itens com `bulletList`.
- Mantenha código/identificadores entre backticks (`code` mark).
- **Nunca** envie em markdown puro nem com `\n` literais — sempre ADF estruturado.

### 6. Confirmar ao usuário
Informe:
- Chave e link do TEST-XX atualizado.
- Que a descrição foi populada com o plano.
