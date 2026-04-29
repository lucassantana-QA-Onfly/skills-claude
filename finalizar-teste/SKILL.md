---
name: finalizar-teste
description: Encerra uma sessão de testes — valida bugs em aberto, garante relatório anexado, transiciona o Project e a issue original, e destrói o ambiente de review. Aceita chave do Project (TEST-XX) ou da issue original (CHEER/DLT/TRPO) como argumento, ou detecta pelo contexto.
argument-hint: "[test-key | issue-key]"
---

# Skill: Finalizar Teste

## Como usar
Invocada com chave explícita (`/finalizar-teste TEST-77` ou `/finalizar-teste CHEER-3080`) ou sem argumento — neste caso, identifique a chave pelo contexto da conversa. Se não conseguir identificar, peça ao usuário.

Se a chave passada for uma issue original (DLT/CHEER/TRPO/etc), localizar o Project (TEST-XX) vinculado via `issuelinks`.

## Passo a passo

### 1. Identificar Project e issue original
1. Use `mcp__Jira__getJiraIssue` para buscar a issue passada.
2. Se for tipo `Project` no projeto `TEST` → essa é a chave do Project.
3. Caso contrário → inspecionar `issuelinks` e localizar issue com `issuetype.name === "Project"` no projeto TEST. Salvar como **Project**. A issue passada vira **issue original**.
4. Se for um Project, descobrir a issue original olhando `issuelinks` com tipo `Test` (outwardIssue).
5. Se nenhum Project for encontrado → abortar e avisar o usuário (sugerir `/iniciar-testes`).

### 2. Validar bugs em aberto (Fix tickets)
1. Inspecione `issuelinks` do Project — colete todas as issues com tipo `Blocks` em que o Project está como bloqueado (`inwardIssue` é o Fix, `outwardIssue` é o Project). Esses são os Fix tickets vinculados.
2. Para cada Fix, verificar status (`statusCategory.key`):
   - `done` → considerado fechado
   - qualquer outro → **em aberto**
3. Se houver Fix em aberto, listar ao usuário (chave, título, status) e perguntar:
   > "Há N Fix tickets em aberto. Deseja prosseguir mesmo assim com a finalização?"
   Aguardar confirmação. Se não confirmar → abortar.

### 3. Validar presença de relatório de execução
1. Buscar comentários do Project via `mcp__Jira__getJiraIssue` (campo `comment.comments`).
2. Procurar comentário cujo body contenha um heading `Relatório de execução`.
3. Se não houver:
   - Avisar o usuário e perguntar se quer anexar agora via `/anexar-relatorio [TEST-XX]`.
   - Se sim → invocar `/anexar-relatorio` e aguardar a anexação. Após sucesso, seguir para próximo passo.
   - Se não → abortar (não permitir finalizar sem relatório).
4. Se houver, perguntar se quer adicionar um relatório complementar ou pular.

### 4. Resumo de bugs encontrados
Apresentar ao usuário um sumário com:
- Total de Fix tickets vinculados
- Quantidade fechada (status `done`) x em aberto
- Lista detalhada (chave, título, status, link)

### 5. Confirmar antes de finalizar
Mostrar ao usuário o que vai acontecer:
- Project **[TEST-XX]** será transicionado para `Concluído`
- Issue original **[ISSUE-KEY]** será transicionada para `[status alvo]` (perguntado no passo 7)
- Ambiente de review **`https://app.<name>.review.onhappy.com.br`** será destruído via job `destroy` na pipeline original

Aguardar confirmação. Só prosseguir se confirmar.

### 6. Transicionar Project para Concluído
1. Use `mcp__Jira__getTransitionsForJiraIssue` no Project.
2. Procurar transição cujo `to.name` ou `name` corresponda a `Concluído` / `Concluida` / `Completed` / `Done` (case/acento insensível).
3. Aplicar via `mcp__Jira__transitionJiraIssue`. Se falhar, avisar e continuar (não bloquear restante).

### 7. Transicionar issue original
1. Use `mcp__Jira__getTransitionsForJiraIssue` na issue original.
2. Listar transições disponíveis ao usuário e perguntar para qual status mover, sugerindo `Ready for Deploy` como padrão (se existir).
3. Aplicar a transição escolhida via `mcp__Jira__transitionJiraIssue`.
4. Se o usuário não quiser transicionar a issue original, pular.

### 8. Destruir ambiente de review
1. Buscar nos comentários do Project o último com heading `Ambiente de review` e extrair:
   - URL: `https://app.<name>.review.onhappy.com.br` → extrair `<name>`
   - Link da pipeline (se presente): `https://gitlab.com/onflylabs/onhappy/review/-/pipelines/<pipeline_id>`
2. Se não conseguir extrair `name` ou pipeline → avisar usuário e pular.
3. Localizar o job `destroy` da pipeline:
   - Use `mcp__claude_ai_Gitlab_MCP__get_pipeline_jobs` no projeto `onflylabs/onhappy/review` com o `pipeline_id` extraído.
   - Filtrar `name === "destroy"`.
4. Disparar o job manual:
   - O job `destroy` é manual (`status: "manual"`) e roda no stage `deploy`.
   - Para executá-lo, usar a API GitLab "play job" — via `mcp__claude_ai_Gitlab_MCP__manage_pipeline` com `retry: true` na pipeline **não** funciona para jobs manuais; o caminho correto é POST `jobs/:job_id/play`.
   - Se o tooling MCP não expuser `play job` diretamente, montar curl autenticado:
     ```bash
     curl -s -X POST \
       "https://gitlab.com/api/v4/projects/onflylabs%2Fonhappy%2Freview/jobs/<job_id>/play" \
       -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
       -H "Content-Type: application/json"
     ```
     Usar variável de ambiente `$GITLAB_TOKEN` se disponível; **não interpolar token em texto visível**.
5. Aguardar a conclusão do job `destroy` (poll com `get_pipeline_jobs` até status `success` / `failed`).
6. Reportar resultado ao usuário.

### 9. Resumo final ao usuário
Informar:
- Chave e link do Project (status final)
- Chave e link da issue original (status final, se transicionada)
- Status do ambiente de review (destruído / falhou / pulado)
- Resumo de Fix tickets (totais e abertos remanescentes, se algum)

### 10. O que esta skill NÃO faz
- Não envia notificação no Google Chat
- Não cria Fix tickets nem Bugs novos
- Não altera descrição do Project
- Não remove Fix tickets em aberto — apenas avisa
