---
name: jira-estimativa
description: Consulta uma demanda no Jira e gera uma estimativa de tempo e esforço de teste (QA) usando a técnica de três pontos, exibindo o resultado sem salvar. Aceita chave da issue como argumento (ex: /jira-estimativa PROJ-123) ou detecta pelo contexto.
argument-hint: "[issue-key]"
---

# Skill: Estimativa de Tempo e Esforço de Teste

## Como usar
Invocado com chave explícita (`/jira-estimativa PROJ-123`) ou sem argumento — neste
caso, identifique a issue pelo contexto da conversa. Se não conseguir identificar,
peça ao usuário a chave da issue.

## Passo a passo

### 1. Buscar a issue
Use `mcp__Jira__getJiraIssue` para buscar a issue. Extraia e guarde:
- Título e descrição
- Critérios de aceitação
- Tipo da issue (Story, Bug, Task, Sub-task etc.)
- Fluxos e regras de negócio descritos
- Issues linkadas relevantes

### 1b. Baixar e analisar imagens (se houver)
Verifique se a issue possui imagens na descrição ou no campo `attachment`. Se sim:

Para cada anexo de imagem, baixe com um único comando e leia diretamente:
```bash
curl -s -L -u "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" "[attachment_content_url]" -o /tmp/jira_img_[id].png
```
Em seguida use `Read` no caminho `/tmp/jira_img_[id].png` para visualizar e extrair contexto visual.

Use o conteúdo das imagens para enriquecer a análise antes de gerar a estimativa.

### 2. Analisar a complexidade da demanda
Com base no conteúdo da issue, avalie os seguintes fatores de complexidade:

**Fatores que aumentam o esforço:**
- Múltiplos fluxos alternativos ou regras de negócio condicionais
- Integrações com sistemas externos (APIs, gateways de pagamento, etc.)
- Dados sensíveis ou restrições de permissão/acesso
- Campos com validações complexas (limites, formatos, combinações)
- Ausência de critérios de aceitação claros
- Dependência de estados anteriores ou dados específicos para reprodução

**Fatores que reduzem o esforço:**
- Escopo bem definido com critérios de aceitação claros
- Fluxo único sem variações relevantes
- Sem integrações externas
- Alteração cosmética ou de texto apenas

### 3. Selecionar as técnicas de teste aplicáveis
Identifique quais técnicas serão necessárias para cobrir a demanda:

- **Particionamento de Equivalência**: entradas agrupáveis em classes válidas/inválidas
- **Análise de Valor Limite**: campos com limites numéricos, datas, quantidades
- **Tabela de Decisão**: múltiplas condições com resultados diferentes
- **Fluxo principal + alternativos**: happy path e variações válidas
- **Casos de erro/exceção**: falhas, permissões negadas, dados inválidos
- **Testes de regressão**: áreas que podem ser impactadas pela mudança
- **Testes exploratórios**: cenários não documentados que merecem investigação

### 4. Aplicar a técnica de Estimativa de Três Pontos (PERT)
Para cada atividade de teste, estime três cenários:

- **O (Otimista)**: tudo ocorre sem impedimentos, escopo claro, sem bugs bloqueantes
- **M (Mais Provável)**: cenário realista considerando complexidade normal e eventuais dúvidas
- **P (Pessimista)**: surgem obstáculos (ambiguidades, regressões, ambiente instável, muitos bugs)

Calcule a estimativa ponderada com a fórmula PERT:

> **E = (O + 4M + P) / 6**

Calcule também o desvio padrão para indicar a incerteza:

> **σ = (P - O) / 6**

Aplique essa fórmula para cada atividade listada no passo 5.

### 5. Gerar a estimativa de tempo e esforço

Com base na análise dos passos anteriores, gere a estimativa no seguinte formato:

---

## Estimativa de Teste — [ISSUE-KEY]: [Título da issue]

### Resumo da análise
[2-3 frases explicando o que foi avaliado e os principais fatores que influenciam o esforço]

### Complexidade
**Nível:** Baixa | Média | Alta | Muito Alta

**Justificativa:** [Explique os principais fatores que determinaram o nível]

### Técnicas de teste aplicáveis
| Técnica | Aplicável | Motivo |
|---|---|---|
| Particionamento de Equivalência | Sim/Não | [motivo] |
| Análise de Valor Limite | Sim/Não | [motivo] |
| Tabela de Decisão | Sim/Não | [motivo] |
| Fluxo principal + alternativos | Sim/Não | [motivo] |
| Casos de erro/exceção | Sim/Não | [motivo] |
| Testes de regressão | Sim/Não | [motivo] |
| Testes exploratórios | Sim/Não | [motivo] |

### Estimativa de esforço por atividade (Três Pontos — PERT)

| Atividade | O (Otimista) | M (Mais Provável) | P (Pessimista) | E = (O+4M+P)/6 | σ = (P-O)/6 |
|---|---|---|---|---|---|
| Análise e planejamento dos testes | Xh | Xh | Xh | Xh | Xh |
| Escrita dos casos de teste | Xh | Xh | Xh | Xh | Xh |
| Execução dos testes (ciclo 1) | Xh | Xh | Xh | Xh | Xh |
| Reteste / validação de correções | Xh | Xh | Xh | Xh | Xh |
| Documentação de evidências | Xh | Xh | Xh | Xh | Xh |
| **Total** | | | | **Xh** | **Xh** |

> **Interpretação:** O total estimado (E) representa o esforço esperado. O desvio padrão (σ) indica a margem de incerteza — quanto maior, mais variável é a atividade.

### Riscos e premissas
- [Risco ou premissa relevante para a estimativa, ex: "assume ambiente estável"]
- [Risco ou premissa relevante, ex: "reteste pode aumentar se houver muitos bugs"]

### Observações
[Qualquer informação adicional relevante, como áreas de regressão, dependências, ou sugestões]

---

### 6. Apresentar a estimativa ao usuário
Exiba a estimativa completa conforme o formato acima. Não salve nada no Jira — apenas exiba o resultado na conversa.

Ao final, informe:
> "Estimativa gerada com base na análise da issue usando a técnica de três pontos (PERT). Caso queira gerar os casos de teste detalhados, use `/jira-testes [ISSUE-KEY]`."
