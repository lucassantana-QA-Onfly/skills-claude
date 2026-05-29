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

**IMPORTANTE — limitação do MCP:** `create_file` sempre cria um arquivo novo, nunca
atualiza o existente. Por isso, o documento é mantido em memória de contexto durante
toda a sessão e gravado no Drive apenas quando o usuário pedir explicitamente
("salva no drive", "grava as evidências") ou ao finalizar os testes.
Isso garante um único arquivo por sessão, sem duplicatas.

---

## Passo a passo

### 1. Identificar contexto
Obtenha do contexto da conversa:
- Chave da sessão de testes (TEST-XX)
- Chave da issue original (ex: INTR-296)
- Título da issue original
- Nome do documento: `[TEST-XX] [ISSUE-KEY] — [Título da issue]`

### 2. Inicializar o documento em memória
Na primeira invocação da sessão, crie o conteúdo do documento em memória:

```
[TEST-XX] [ISSUE-KEY] — [Título da issue]
================================================
```

Mantenha este conteúdo acumulado no contexto da conversa ao longo de toda a sessão.
**Não grave no Drive ainda.**

### 3. Acumular evidências em memória
A cada nova evidência recebida durante a sessão (print, resultado de console, log):

1. Acrescente ao conteúdo em memória seguindo a estrutura abaixo.
2. Confirme ao usuário: "Evidência N registrada em memória. [resumo de 1 linha]"
3. **Não chame nenhuma ferramenta do Drive** — apenas atualize o conteúdo em contexto.

Estrutura de cada evidência:
```
EVIDÊNCIA N
[Resumo descritivo: cenário testado, resultado observado, conclusão.]

[Detalhes: valores de console, URL, comportamento. Se for imagem, descreva o que aparece.]
[Anexar manualmente: NomeDoArquivo.png]

------------------------------------------------
```

### 4. Gravar no Drive (somente quando solicitado)
Quando o usuário pedir para salvar ("salva no drive", "grava as evidências",
"finaliza o doc") ou ao encerrar a sessão de testes:

**4.1 — Localizar pasta raiz:**
```
query: title = 'Documento de testes' and mimeType = 'application/vnd.google-apps.folder'
```

**4.2 — Localizar subpasta do projeto:**
```
query: title = '<nome-da-pasta>' and mimeType = 'application/vnd.google-apps.folder' and '<id-pai>' in parents
```
Se não existir, crie com `mimeType: application/vnd.google-apps.folder`.

**4.3 — Verificar se já existe um doc desta sessão:**
```
query: title = '<nome-do-doc>' and '<id-subpasta>' in parents
```
- Se existir: avise o usuário que uma versão anterior existe e será substituída.
  O usuário deve deletar a versão antiga manualmente após confirmar o conteúdo.
- Se não existir: prossiga.

**4.4 — Criar o arquivo** com `mcp__claude_ai_Google_Drive__create_file`:
- `title`: nome do documento
- `parentId`: id da subpasta do projeto
- `textContent`: conteúdo completo acumulado em memória
- `contentMimeType`: `text/plain`

Confirme ao usuário: nome do arquivo, pasta e número de evidências gravadas.

---

## Observações
- Durante a sessão, responda "Evidência N adicionada." sem chamar ferramentas do Drive.
- Imagens/prints são mencionados no texto com "[Anexar manualmente: arquivo.png]"
  e o usuário os adiciona diretamente no Google Drive após a gravação.
- Se o usuário não tiver o Drive autenticado, oriente a autenticar antes de gravar.
