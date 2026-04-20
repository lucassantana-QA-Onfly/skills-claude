---
name: criar-bug
description: Registra um bug de acordo com o time da tarefa. Time Travel (TRPO e similares): comenta na tarefa original e marca "Need fix". Time Onhappy (DLT, CHEER): cria ticket Fix no projeto TEST vinculado ao card. Bug avulso no backlog OnHappy Web: cria Bug direto no projeto CHEER, sem vínculo. Aceita chave da issue como argumento (ex: /criar-bug PROJ-123) ou detecta pelo contexto.
argument-hint: "[issue-key]"
---

# Skill: Registrador de Bug

## Como usar
Invocado com chave explícita (`/criar-bug PROJ-123`) ou sem argumento — neste
caso, identifique a issue pelo contexto da conversa. Se não conseguir identificar,
peça ao usuário a chave da issue.

## Identificar o fluxo pelo projeto

Antes de qualquer ação, verifique o prefixo da issue:

- Se o projeto for **DLT** ou **CHEER** → seguir o **Fluxo Onhappy** (abaixo)
- Se o projeto for **TRPO** ou outro projeto do time **Travel** → seguir o **Fluxo Travel (Padrão)** (abaixo)
- Se o usuário pedir um **bug avulso no backlog do OnHappy Web** (ex: "abrir bug no backlog onhappy web", "criar bug no CHEER sem vincular") sem referenciar uma issue existente → seguir o **Fluxo Backlog OnHappy Web (CHEER)** (abaixo)

---

## Fluxo Travel (Padrão) — projetos TRPO e demais do time Travel

### 1. Buscar a tarefa original
Use `mcp__Jira__getJiraIssue` para buscar a issue. Extraia e guarde:
- Título e descrição
- Projeto
- **Responsável (assignee)** — salve o accountId e o displayName para reuso

### 2. Coletar o relato do bug
Pergunte ao usuário:

> "Descreva o bug encontrado. Pode incluir:
> - O que aconteceu (comportamento atual)
> - O que era esperado (comportamento esperado)
> - Passos para reproduzir
> - Ambiente/contexto (ex: navegador, dispositivo, dados usados)
> - Qualquer outra informação relevante"

Aguarde o relato antes de continuar.

### 3. Gerar a descrição do bug
Com base no relato do usuário, gere uma descrição bem formatada contendo:

```
[Resumo claro do problema em 1-2 frases — sem subtítulo "Descrição"]

**Comportamento atual**
[O que está acontecendo de errado]

**Comportamento esperado**
[O que deveria acontecer]

**Passos para reproduzir**
1. ...
2. ...
3. ...

**Ambiente**
[Navegador, dispositivo, dados, configuração — se informado]

**Informações adicionais**
[Qualquer outro detalhe relevante mencionado]
```

Preencha apenas as seções que o usuário informou. Omita seções sem informação. **Não inclua o subtítulo "Descrição"** — comece diretamente com o resumo do problema.

### 4. Confirmar antes de registrar
Apresente a descrição gerada ao usuário e pergunte:

> "Deseja registrar esse bug como comentário na tarefa **[ISSUE-KEY]**?"

Aguarde a confirmação. Só prossiga se o usuário confirmar.

### 5. Registrar o bug como comentário na tarefa original
Use `mcp__Jira__addCommentToJiraIssue` na tarefa original com a descrição gerada no passo 3.

### 6. Marcar "Need fix" na tarefa original
Use `mcp__Jira__editJiraIssue` na tarefa original para marcar "Need fix"
no campo "Checkbox":
1. Localize o campo Checkbox (busque por `customfield_*` com label "Checkbox")
2. Adicione "Need fix" à lista de itens marcados, **preservando os já existentes**

Se o campo não existir ou "Need fix" não for encontrado, informe o usuário
em vez de falhar silenciosamente.

### 7. Comentar na tarefa original mencionando o responsável
Use `mcp__Jira__addCommentToJiraIssue` na tarefa original com um comentário ADF mencionando o assignee salvo no passo 1:

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {
      "type": "paragraph",
      "content": [
        {
          "type": "mention",
          "attrs": {
            "id": "<accountId do assignee>",
            "text": "@<displayName do assignee>"
          }
        }
      ]
    }
  ]
}
```

### 8. Confirmar ao usuário
Informe tudo que foi feito:
- Confirmação de que o bug foi registrado como comentário na tarefa original
- Confirmação de que "Need fix" foi marcado (ou aviso se não foi possível)
- Confirmação de que o responsável foi mencionado na tarefa original

---

## Fluxo Onhappy (projetos DLT e CHEER)

### 1. Buscar a tarefa original
Use `mcp__Jira__getJiraIssue` para buscar a issue. Extraia e guarde:
- Título e descrição
- Projeto
- **Responsável (assignee)** — salve o accountId e o displayName para reuso
- **Project de testes vinculado** — inspecione `issuelinks` e localize o issue com `issuetype.name === "Project"` (summary típico: "Sessão de Testes — <ISSUE-KEY>: <Título>"). Salve a chave (ex: `TEST-39`). Se não houver Project vinculado, avise o usuário — o Fix precisa ser vinculado ao Project, não à tarefa original. Pergunte se deseja rodar `/iniciar-testes` antes de prosseguir.

### 2. Coletar o relato do bug
Pergunte ao usuário:

> "Descreva o bug encontrado. Pode incluir:
> - O que aconteceu (comportamento atual)
> - O que era esperado (comportamento esperado)
> - Passos para reproduzir
> - Ambiente/contexto (ex: navegador, dispositivo, dados usados)
> - Qualquer outra informação relevante
> - Print/screenshot (informe o caminho do arquivo, se tiver)"

Aguarde o relato antes de continuar.

### 3. Gerar a descrição do bug
Com base no relato do usuário, gere uma descrição bem formatada contendo:

```
[Resumo claro do problema em 1-2 frases — sem subtítulo "Descrição"]

**Comportamento atual**
[O que está acontecendo de errado]

**Comportamento esperado**
[O que deveria acontecer]

**Passos para reproduzir**
1. ...
2. ...
3. ...

**Ambiente**
[Navegador, dispositivo, dados, configuração — se informado]

**Informações adicionais**
[Qualquer outro detalhe relevante mencionado]
```

Preencha apenas as seções que o usuário informou. Omita seções sem informação. **Não inclua o subtítulo "Descrição"** — comece diretamente com o resumo do problema.

### 4. Confirmar antes de registrar
Apresente a descrição gerada ao usuário e pergunte:

> "Deseja criar um ticket Fix no projeto TEST vinculado ao Project **[PROJECT-KEY]** (Sessão de Testes da tarefa **[ISSUE-KEY]**)?"

Aguarde a confirmação. Só prossiga se o usuário confirmar.

### 5. Criar o ticket Fix no projeto TEST
Use `mcp__Jira__createJiraIssue` para criar uma nova issue com:
- **Projeto**: TEST
- **Tipo de issue**: Fix (ou o tipo mais próximo disponível — use `mcp__Jira__getJiraProjectIssueTypesMetadata` para verificar os tipos disponíveis no projeto TEST)
- **Título (summary)**: resumo objetivo do defeito, derivado do relato do usuário. **NÃO** usar o título da tarefa original. **NÃO** incluir a chave da issue no título (ex: [DLT-170]) — o vínculo já registra essa relação.
- **Descrição**: apenas comportamento atual, comportamento esperado e ambiente (se informado). **NÃO incluir os passos de reprodução na descrição.**
- **Campo personalizado `customfield_11690`** ("Passo a passo de reprodução"): preencher com os passos em formato de bullet points. Este campo é **obrigatório** e separado da descrição.
- **Campo "Prioridade"**: **não preencher** a menos que o usuário indique explicitamente uma prioridade. Deixar o valor padrão do campo.

Salve a chave e o **summary (título)** da nova issue criada para uso nos próximos passos.

Quando o ticket Fix já existir (informado pelo usuário), use `mcp__Jira__getJiraIssue` para buscar o título real do ticket antes de enviar a notificação.

### 5.1 Vincular o Fix ao Project de testes
Use `mcp__Jira__createIssueLink` com tipo **"Relates"** entre o Fix recém-criado e o **Project** salvo no passo 1 (ex: `TEST-39`).

**Importante:** o link estrutural é entre Fix ↔ Project de testes, **não** entre Fix ↔ tarefa original. A tarefa original recebe apenas o smart link (inlineCard) como comentário, mas o vínculo de `issuelinks` fica com o Project.

```
inwardIssue: <chave do Fix criado>
outwardIssue: <chave do Project de testes>
type: "Relates"
```

### 5.5 Anexar arquivo ao ticket Fix (opcional)
Se o usuário informou um arquivo (imagem ou vídeo) no relato, anexe-o ao **ticket Fix criado** via Bash tool:

```bash
WINPATH=$(cygpath -w "/c/caminho/para/arquivo") && curl -s -X POST \
  "https://api.atlassian.com/ex/jira/24479377-75bf-4543-a6f6-0a189a0ec825/rest/api/3/issue/{TEST-XXX}/attachments" \
  -H "Authorization: Basic $(printf '%s' "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" | base64 -w 0)" \
  -H "X-Atlassian-Token: no-check" \
  -F "file=@\"$WINPATH\";type={MIME_TYPE}"
```

**Adaptações para Windows (Git Bash):**
- Use `cygpath -w` para converter o caminho Unix para Windows antes de passar ao curl
- As variáveis `$ATLASSIAN_EMAIL` e `$ATLASSIAN_TOKEN` já estão configuradas no ambiente — nunca interpole os valores diretamente

Detecte o tipo MIME pelo formato do arquivo:
- `.png` → `image/png`
- `.jpg` / `.jpeg` → `image/jpeg`
- `.gif` → `image/gif`
- `.webp` → `image/webp`
- `.mp4` → `video/mp4`
- `.mov` → `video/quicktime`

**Limitação importante — embedding inline na descrição:**
Não é possível incorporar o arquivo inline no corpo da descrição via API REST. O Jira exige um `mediaApiFileId` (ID interno do serviço de mídia da Atlassian) para usar o nó `mediaSingle` do ADF, e esse ID **não é retornado** pelos endpoints REST públicos. O arquivo ficará visível na aba **Attachments** do ticket. Para exibição inline, o usuário deve arrastar o arquivo para dentro da descrição diretamente na interface do Jira.

Se o curl retornar erro, informe o usuário e continue o fluxo normalmente.

### 6. Marcar "Need fix" na tarefa original (somente projetos CHEER)

Se a issue original for do projeto **CHEER**, use `mcp__Jira__editJiraIssue` para marcar "Need fix" no campo "Checkbox":
1. Localize o campo Checkbox (busque por `customfield_*` com label "Checkbox")
2. Adicione "Need fix" à lista de itens marcados, **preservando os já existentes**

Se o campo não existir ou "Need fix" não for encontrado, informe o usuário em vez de falhar silenciosamente.

> Para issues **DLT**, este passo deve ser ignorado.

### 7. Comentar no ticket Fix mencionando o responsável da tarefa original
Use `mcp__Jira__addCommentToJiraIssue` no **ticket Fix criado** (ex: TEST-456) com um comentário ADF mencionando o assignee salvo no passo 1:

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {
      "type": "paragraph",
      "content": [
        {
          "type": "mention",
          "attrs": {
            "id": "<accountId do assignee>",
            "text": "@<displayName do assignee>"
          }
        }
      ]
    }
  ]
}
```

### 8. Confirmar ao usuário
Informe tudo que foi feito:
- Confirmação de que o ticket Fix foi criado no projeto TEST (com a chave gerada)
- Confirmação de que o Fix foi vinculado ao Project de testes (ex: `TEST-39`)
- Confirmação de que o screenshot foi anexado ao ticket Fix (ou aviso se não foi possível/informado)
- Confirmação de que "Need fix" foi marcado na tarefa original (somente CHEER) ou aviso se não foi possível
- Confirmação de que o responsável foi mencionado no ticket Fix

---

## Fluxo Backlog OnHappy Web (CHEER) — bug avulso no backlog

Usar quando o usuário pedir para abrir um bug **direto no backlog** do OnHappy Web, sem vínculo com nenhum card existente.

### 1. Coletar o relato do bug
Pergunte ao usuário:

> "Descreva o bug encontrado. Pode incluir:
> - O que aconteceu (comportamento atual)
> - O que era esperado (comportamento esperado)
> - Passos para reproduzir
> - Ambiente/contexto (ex: navegador, dispositivo, dados usados)
> - Qualquer outra informação relevante
> - Print/screenshot (informe o caminho do arquivo, se tiver)"

Aguarde o relato antes de continuar.

### 2. Gerar a descrição do bug
Gere a descrição em **ADF** com as seções abaixo, na ordem:

1. Parágrafo inicial com resumo claro do problema em 1-2 frases — **sem subtítulo "Descrição"**
2. `Comportamento atual` (heading 3) + bullet list
3. `Comportamento esperado` (heading 3) + bullet list
4. `Passo a passo de reprodução` (heading 3) + bullet list
5. `Ambiente` (heading 3) + bullet list — **somente se o usuário informou**. Omita a seção inteira caso contrário.

Preencha apenas as seções que o usuário informou. Não inclua seções vazias.

### 3. Confirmar antes de registrar
Apresente a descrição gerada ao usuário e pergunte:

> "Deseja criar esse Bug no backlog do OnHappy Web (projeto CHEER)?"

Aguarde a confirmação. Só prossiga se o usuário confirmar.

### 4. Criar o Bug no projeto CHEER
Use `mcp__Jira__createJiraIssue` com:
- **Projeto**: CHEER
- **Tipo de issue**: Bug
- **Título (summary)**: resumo objetivo do defeito, derivado do relato do usuário. **NÃO** incluir chave de outra issue no título (ex: `[DLT-170]`).
- **Descrição**: o conteúdo ADF gerado no passo 2 (use `contentFormat: "adf"`)
- **Campo "Prioridade"**: **não preencher** a menos que o usuário indique explicitamente. Deixar o valor padrão.

Salve a chave da nova issue criada (ex: `CHEER-XXXX`).

### 4.5 Verificar e limpar template padrão na descrição
Após a criação, use `mcp__Jira__getJiraIssue` para buscar a issue criada e inspecione o campo `description`.

Se o conteúdo contiver **marcadores de template** — padrões como:
- Placeholders entre chaves duplas (`{{ ... }}`)
- Blocos de heading típicos do template de Bug no CHEER (ex: "Descrição do erro encontrado", "Comportamento esperado", "Pontos de atenção") acompanhados de texto placeholder

Então use `mcp__Jira__editJiraIssue` para **sobrescrever** o campo `description` com o ADF gerado no passo 2.

Se a descrição já contiver apenas o texto do bug (sem placeholders), pule este passo.

### 5. Anexar arquivo ao Bug (opcional)
Se o usuário informou um print/screenshot/vídeo, anexe-o ao Bug criado via Bash tool:

```bash
WINPATH=$(cygpath -w "/c/caminho/para/arquivo") && curl -s -X POST \
  "https://api.atlassian.com/ex/jira/24479377-75bf-4543-a6f6-0a189a0ec825/rest/api/3/issue/{CHEER-XXXX}/attachments" \
  -H "Authorization: Basic $(printf '%s' "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" | base64 -w 0)" \
  -H "X-Atlassian-Token: no-check" \
  -F "file=@\"$WINPATH\";type={MIME_TYPE}"
```

Use `cygpath -w` no Windows/Git Bash. Detecte o MIME pelo formato (`.png`→`image/png`, `.jpg`→`image/jpeg`, `.mp4`→`video/mp4`, etc.). Nunca interpole `$ATLASSIAN_EMAIL` ou `$ATLASSIAN_TOKEN` em texto visível.

Se o curl retornar erro, informe e continue o fluxo.

### 6. Confirmar ao usuário
Informe:
- Chave e link do Bug criado (ex: `https://onflylabs.atlassian.net/browse/CHEER-XXXX`)
- Se o template da descrição foi detectado e substituído (ou se já estava limpo)
- Se o arquivo foi anexado (ou aviso caso não tenha sido possível / não informado)
- Observação: criado no backlog sem responsável (o time define na priorização), a menos que o usuário tenha indicado um responsável específico
