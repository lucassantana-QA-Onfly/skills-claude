---
name: documento-evidencias
description: Cria ou atualiza um Google Doc de evidências de testes na pasta correta do projeto no Google Drive (dentro de "Documento de testes"). Cada evidência é inserida com um resumo descritivo acima do conteúdo. Aceita chave da sessão (TEST-XX) ou da issue original como argumento, ou detecta pelo contexto.
argument-hint: "[TEST-XX | PROJ-123]"
---

# Skill: Documento de Evidências no Google Drive

## Como usar
Invocado com chave explícita (`/documento-evidencias TEST-268`) ou sem argumento —
neste caso, identifique a sessão de testes (TEST-XX) e a issue original pelo contexto
da conversa.

> Esta skill **cria ou atualiza** um Google Doc no Drive. Não altera nada no Jira ou GitLab.

---

## Mapeamento de projeto → pasta no Drive

Com base na chave da issue original, identifique a pasta dentro de "Documento de testes":

| Prefixo da issue | Pasta no Drive |
|------------------|----------------|
| INTR-XXX         | Internacional  |
| CHEER-XXX        | OnHappy Web    |
| DLT-XXX          | OnHappy Web    |
| TRPO-XXX         | Travel Pós     |

Se o prefixo não estiver na tabela, pergunte ao usuário em qual pasta salvar.

---

## Funcionamento geral

**Responsabilidades divididas:**
- **Claude:** cria o documento vazio no Drive (apenas uma vez, na primeira invocação)
  e gera o conteúdo textual de cada evidência para o usuário colar manualmente.
- **Usuário:** cola o conteúdo gerado no documento e anexa os prints.

Claude **nunca tenta atualizar o documento no Drive** após a criação inicial.

---

## Passo a passo

### 1. Identificar contexto
Obtenha do contexto da conversa:
- Chave da sessão de testes (TEST-XX)
- Chave da issue original (ex: INTR-296)
- Título da issue original
- Nome do documento: `[TEST-XX] [ISSUE-KEY] — [Título da issue]`

### 2. Criar o documento no Drive (somente na primeira invocação)
Se o documento ainda não existir:

**2.1 — Localizar pasta raiz:**
```
query: title = 'Documento de testes' and mimeType = 'application/vnd.google-apps.folder'
```

**2.2 — Localizar subpasta do projeto** (criar se não existir):
```
query: title = '<nome-da-pasta>' and mimeType = 'application/vnd.google-apps.folder' and '<id-pai>' in parents
```

**2.3 — Criar o documento** com `mcp__claude_ai_Google_Drive__create_file`:
- `title`: nome do documento
- `parentId`: id da subpasta
- `textContent`: apenas o cabeçalho inicial
- `contentMimeType`: `text/plain`

Cabeçalho inicial:
```
[TEST-XX] [ISSUE-KEY] — [Título da issue original]
================================================
```

Informe ao usuário o link do documento criado e que ele deve colar as evidências manualmente.

Se o documento já existir na pasta, apenas informe o link e pule a criação.

### 3. Gerar conteúdo da evidência
A cada nova evidência durante a sessão, gere o bloco de texto abaixo e entregue
ao usuário para colar no documento:

```
EVIDÊNCIA N
O que foi testado: [descrição objetiva do cenário executado]

Resultado esperado: [comportamento correto conforme especificação]

Resultado obtido: [o que realmente aconteceu — valores, mensagens, comportamento observado]

------------------------------------------------
```

- Numere sequencialmente com base nas evidências já registradas na conversa.
- Se for print/imagem: descreva em "Resultado obtido" o que aparece (valores no console, URL, comportamento visível).
- **Não chame nenhuma ferramenta do Drive** — apenas gere o texto.

---

## Observações
- Claude não atualiza o Drive após a criação inicial. Todo conteúdo é gerado como
  texto para o usuário colar diretamente no documento.
- Prints são anexados manualmente pelo usuário no Google Drive.
- Se o usuário não tiver o Drive autenticado, oriente a autenticar antes de criar.
