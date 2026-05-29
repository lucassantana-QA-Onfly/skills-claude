---
name: jira-testes
description: Consulta uma demanda no Jira, gera plano de teste (QA) e anexa na descrição do Project (TEST-XX). Aceita chave da issue como argumento (ex: /jira-testes PROJ-123) ou detecta pelo contexto.
argument-hint: "[issue-key]"
---

# Skill: Gerador de Plano de Teste QA

## Modelo de execução
- **Análise e geração do plano** (passos 1 a 3 — leitura da issue, Figma, GitLab diff, elaboração do plano): usar `Agent` com `model: "opus"` (`claude-opus-4-8`) para análise aprofundada.
- **Criação no Jira** (passos 4 a 6 — atualizar descrição do Project via API): executar no contexto principal (Sonnet).

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

### 1d. Consultar Confluence (OBRIGATÓRIO para projetos OnHappy | ignorar para INTR)

#### Projetos OnHappy (CHEER, DLT, TRPO etc.)

**Sempre** consulte o espaço OnHappy no Confluence antes de gerar o plano de teste. O Confluence é a fonte de verdade para regras de negócio, fluxos e padrões do produto.

**Como consultar:**
1. Acesse a visão geral do espaço: `https://onflylabs.atlassian.net/wiki/spaces/OnHappy/overview`
2. Use `Atlassian:searchConfluenceUsingCql` com `space.key = "OnHappy"` buscando pelos termos-chave da issue (ex.: nome da feature, módulo afetado, fluxo descrito).
3. Leia as páginas mais relevantes com `Atlassian:getConfluencePage`.
4. Extraia e use no plano:
   - Regras de negócio que a issue implementa ou modifica
   - Fluxos existentes que podem ser afetados (regressão)
   - Definições de termos do produto (ex.: o que é Wallet, como cupons funcionam)
   - Restrições ou comportamentos documentados que não estão na issue

> Não pule esse passo mesmo que a issue pareça simples — o Confluence frequentemente revela regras implícitas que enriquecem os cenários de teste e os pontos de atenção.

#### Projetos International (INTR)

**Pule a consulta ao Confluence.** Para issues INTR, use apenas o conteúdo da issue, subtarefas, Figma e diff do GitLab para compor o plano. Lembre-se de que o contexto de produto é a plataforma **Onfly** (gestão de viagens corporativas), não o OnHappy — os cenários devem refletir esse domínio (políticas de viagem, aprovações, emissão de bilhetes, relatórios de despesas, integrações com fornecedores, etc.).

### 1e. Buscar e analisar comentários da issue — OBRIGATÓRIO

Use `Atlassian:getJiraIssue` incluindo o campo `comment` para buscar os comentários da issue. Leia todos os comentários e filtre os que sejam pertinentes ao teste:

- Instruções ou requisitos adicionais de validação deixados pelo desenvolvedor ou PO
- Restrições de ambiente, dados ou pré-condições mencionadas
- Esclarecimentos sobre comportamento esperado que não constam na descrição
- Alertas sobre fluxos que devem ou não ser testados
- Pré-condições específicas de dados (ex.: "ter usuário X sem Y para validar Z")

Incorpore essas informações no plano — especialmente nas seções **Dados/Pré-condições** e **O que será testado**. Não ignore comentários mesmo que a descrição pareça completa.

### 1f. Consultar GitLab — OBRIGATÓRIO se houver MR/branch vinculado

**Sempre** invoque `/gitlab-impacto [ISSUE-KEY]` se a issue tiver MR ou branch vinculado no GitLab. Não pule esse passo mesmo que a descrição técnica do card pareça completa — o diff pode revelar arquivos afetados não mencionados, efeitos colaterais em outros módulos e contexto real da mudança que enriquece os pontos de atenção e os cenários de regressão do plano.

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
