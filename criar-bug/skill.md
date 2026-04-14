---
name: criar-bug
description: Registra um bug de acordo com o time da tarefa. Time Travel (TRPO e similares): comenta na tarefa original, marca "Need fix" e notifica. Time Onhappy (DLT, CHEER): cria ticket Fix no projeto TEST, vincula como "causes" na tarefa original e notifica. Aceita chave da issue como argumento (ex: /criar-bug PROJ-123) ou detecta pelo contexto.
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

### 7. Notificar o responsável via Google Chat
Consulte a memória `reference_google_chat_webhooks.md` para identificar o webhook
correto com base no projeto da tarefa. Envie uma mensagem no espaço correspondente
informando sobre o bug registrado, incluindo o link da tarefa no Jira.

Envie via curl usando `--data-binary @arquivo.json` para evitar problemas de encoding.
Evite acentos e caracteres especiais **somente na mensagem do Google Chat** para garantir legibilidade no webhook. Isso **não se aplica** ao comentário registrado no Jira — este deve preservar a acentuação correta do português.

### 8. Confirmar ao usuário
Informe tudo que foi feito:
- Confirmação de que o bug foi registrado como comentário na tarefa original
- Confirmação de que "Need fix" foi marcado (ou aviso se não foi possível)
- Confirmação de que o responsável foi notificado via Google Chat

---

## Fluxo Onhappy (projetos DLT e CHEER)

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

> "Deseja criar um ticket Fix no projeto TEST vinculado à tarefa **[ISSUE-KEY]**?"

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

### 5.5 Anexar screenshot ao ticket Fix (opcional)
Se o usuário informou um arquivo de imagem no relato, anexe-o ao **ticket Fix criado** via Bash tool:

```bash
curl -s -X POST \
  "https://api.atlassian.com/ex/jira/24479377-75bf-4543-a6f6-0a189a0ec825/rest/api/3/issue/{TEST-XXX}/attachments" \
  -H "Authorization: Basic $(printf '%s' "$JIRA_EMAIL:$JIRA_TOKEN" | base64 -w 0)" \
  -H "X-Atlassian-Token: no-check" \
  -F "file=@'{CAMINHO_DO_ARQUIVO}';type={MIME_TYPE}"
```

**Adaptações para Windows (Git Bash):**
- Converta `\` por `/` no caminho do arquivo (ex: `C:/Users/lucas/screenshot.png`)
- `base64 -w 0` e `printf '%s'` funcionam normalmente no Git Bash
- As variáveis `$JIRA_EMAIL` e `$JIRA_TOKEN` devem estar definidas no ambiente — nunca interpole os valores diretamente no comando

Detecte o tipo MIME pelo formato do arquivo:
- `.png` → `image/png`
- `.jpg` / `.jpeg` → `image/jpeg`
- `.gif` → `image/gif`
- `.webp` → `image/webp`

Se o curl retornar erro, informe o usuário e continue o fluxo normalmente.

### 6. Vincular o ticket Fix como "causes" na tarefa original
Use `mcp__Jira__createIssueLink` para vincular as issues:
- **Tipo de link**: "causes" (ou equivalente — use `mcp__Jira__getIssueLinkTypes` para verificar o tipo exato disponível)
- **Issue de origem (outward)**: o ticket Fix criado (ex: TEST-456)
- **Issue de destino (inward)**: a tarefa original (ex: DLT-170)

Se o tipo "causes" não existir, use o tipo de link mais semanticamente próximo disponível e informe o usuário qual foi usado.

### 7. Notificar via Google Chat
Consulte a memória `reference_google_chat_webhooks.md` para o webhook do projeto Onhappy (DLT → Onhappy, CHEER → Onhappy).

Envie a mensagem no seguinte formato de card (manter este layout exato):

```json
{
  "cards": [
    {
      "header": {
        "title": "Bug registrado - [ISSUE-KEY]",
        "subtitle": "[Nome do projeto]"
      },
      "sections": [
        {
          "widgets": [
            {
              "textParagraph": {
                "text": "Ola [displayName do assignee], um ticket Fix foi criado e vinculado a [ISSUE-KEY].\n\n*[Titulo do ticket Fix]*"
              }
            },
            {
              "buttons": [
                {
                  "textButton": {
                    "text": "Ver Fix: [TEST-XXX]",
                    "onClick": { "openLink": { "url": "https://onflylabs.atlassian.net/browse/TEST-XXX" } }
                  }
                },
                {
                  "textButton": {
                    "text": "Ver tarefa: [ISSUE-KEY]",
                    "onClick": { "openLink": { "url": "https://onflylabs.atlassian.net/browse/[ISSUE-KEY]" } }
                  }
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

Substitua:
- `[displayName do assignee]` pelo nome do responsável da tarefa original (salvo no passo 1)
- `[ISSUE-KEY]` pela chave da tarefa original
- `[TEST-XXX]` pela chave do ticket Fix criado
- `[Titulo do ticket Fix]` pelo summary do ticket Fix criado no passo 5
- `[Nome do projeto]` pelo nome do projeto Jira da tarefa original

Envie via curl usando `--data-binary @arquivo.json` para evitar problemas de encoding.
Evite acentos e caracteres especiais **somente na mensagem do Google Chat** para garantir legibilidade no webhook. Isso **não se aplica** ao título nem à descrição do ticket Fix no Jira — estes devem preservar a acentuação correta do português.

### 8. Confirmar ao usuário
Informe tudo que foi feito:
- Confirmação de que o ticket Fix foi criado no projeto TEST (com a chave gerada)
- Confirmação de que o screenshot foi anexado ao ticket Fix (ou aviso se não foi possível/informado)
- Confirmação de que o vínculo "causes" foi criado com a tarefa original
- Confirmação de que o responsável foi notificado via Google Chat
