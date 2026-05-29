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

## Passo a passo

### 1. Identificar contexto
Obtenha do contexto da conversa:
- Chave da sessão de testes (TEST-XX)
- Chave da issue original (ex: INTR-296)
- Título da issue original
- Nome do documento: `[TEST-XX] [ISSUE-KEY] — [Título da issue]`

### 2. Localizar a pasta "Documento de testes" no Drive
Use `mcp__claude_ai_Google_Drive__search_files` para encontrar a pasta raiz:
```
query: name = 'Documento de testes' and mimeType = 'application/vnd.google-apps.folder'
```
Guarde o `id` da pasta encontrada.

### 3. Localizar a subpasta do projeto
Use `mcp__claude_ai_Google_Drive__search_files` para encontrar a subpasta do projeto
dentro de "Documento de testes":
```
query: name = '<nome-da-pasta>' and mimeType = 'application/vnd.google-apps.folder' and '<id-pai>' in parents
```
Guarde o `id` da subpasta.

Se a subpasta não existir, crie-a com `mcp__claude_ai_Google_Drive__create_file`
usando `mimeType: application/vnd.google-apps.folder` e `parentId` da pasta raiz.

### 4. Verificar se já existe um documento para a sessão

**IMPORTANTE:** `create_file` sempre cria um arquivo novo — nunca sobrescreve o existente.
Para evitar duplicatas, siga este fluxo:

Use `mcp__claude_ai_Google_Drive__search_files` buscando pelo título exato:
```
query: title = '<nome-do-doc>' and '<id-subpasta>' in parents
```

- **Se existir (um ou mais resultados):**
  1. Use o arquivo com `modifiedTime` mais recente.
  2. Leia o conteúdo atual com `mcp__claude_ai_Google_Drive__read_file_content`.
  3. Construa o conteúdo atualizado (conteúdo existente + nova evidência).
  4. Crie um novo arquivo com `mcp__claude_ai_Google_Drive__create_file` contendo
     TODO o conteúdo (antigo + novo).
  5. **Avise o usuário** que o arquivo antigo ficou duplicado no Drive e que ele
     deve deletar manualmente o(s) arquivo(s) mais antigo(s) com o mesmo nome.

- **Se não existir:** crie com `mcp__claude_ai_Google_Drive__create_file`.

### 5. Estrutura do documento

O documento deve seguir este formato em texto simples (plain text):

```
[TEST-XX] [ISSUE-KEY] — [Título da issue]
================================================

EVIDÊNCIA 1
[Resumo descritivo explicando o que a evidência mostra, o cenário testado
e o resultado observado. Deve ser objetivo e informativo para quem lê sem contexto.]

[Conteúdo da evidência — se for texto/log, cole aqui. Se for imagem, descreva
detalhadamente o que aparece: URL, valores no console, comportamento observado, etc.]

------------------------------------------------

EVIDÊNCIA 2
[Resumo descritivo...]

[Conteúdo...]

------------------------------------------------
```

### 6. Inserir a nova evidência
Ao adicionar uma evidência:
- Use o contexto da conversa para redigir o resumo descritivo.
- Se for um print/imagem: descreva o que aparece (URL, resultado no console,
  comportamento observado) com base no que o usuário compartilhou.
- Se for texto/log: cole o trecho relevante.
- Numere sequencialmente (`EVIDÊNCIA N`).

### 7. Salvar o documento
Use `mcp__claude_ai_Google_Drive__create_file` com:
- `title`: nome do documento
- `parentId`: id da subpasta do projeto
- `textContent`: conteúdo completo do documento
- `contentMimeType`: `text/plain` (será convertido para Google Doc automaticamente)

Confirme ao usuário o nome do arquivo e a pasta onde foi salvo.

---

## Observações
- Sempre que o usuário compartilhar um print ou resultado de teste durante a sessão,
  pergunte se quer adicionar ao documento de evidências.
- O documento é cumulativo: cada chamada à skill adiciona novas evidências sem apagar
  as anteriores.
- Se o usuário não tiver o Drive autenticado, oriente a autenticar antes de prosseguir.
