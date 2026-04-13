---
name: atualizar-skills
description: Sincroniza as skills locais do Claude com o repositório remoto no GitHub (lucassantana-QA-Onfly/skills-claude). Deve ser chamada sempre que uma skill for criada ou atualizada.
argument-hint: "[mensagem de commit opcional]"
---

# Skill: Atualizar Repositório de Skills

## Objetivo
Fazer commit e push das alterações em `/c/Users/lucas.santana_onfly/.claude/skills/` para o repositório remoto `lucassantana-QA-Onfly/skills-claude` no GitHub.

O diretório de skills **já é** um repositório git clonado — não é necessário clonar nem copiar arquivos.

---

## Passo 1 — Verificar se há mudanças

```bash
cd /c/Users/lucas.santana_onfly/.claude/skills && git status --short
```

Se a saída estiver vazia, informe o usuário: **"Nenhuma alteração detectada. O repositório já está atualizado."** e encerre.

---

## Passo 2 — Definir mensagem de commit

- Se o usuário passou um argumento ao invocar a skill, use-o como mensagem.
- Caso contrário, liste os arquivos no `git status --short` e gere uma mensagem no formato:
  `update: <nome(s) da(s) skill(s) alterada(s)>`
  Exemplo: `update: criar-bug, jira-testes`

---

## Passo 3 — Commit e push

Configure o remote com autenticação via `$GITHUB_TOKEN`, faça o commit e push:

```bash
cd /c/Users/lucas.santana_onfly/.claude/skills
git remote set-url origin "https://x-access-token:$GITHUB_TOKEN@github.com/lucassantana-QA-Onfly/skills-claude.git"
git config user.email "lucas.santana@onfly.com.br"
git config user.name "Lucas Santana"
git add -A
git commit -m "<mensagem definida no passo 2>"
git push origin master
```

---

## Passo 4 — Confirmar ao usuário

Informe de forma resumida:
- Quais skills foram alteradas/adicionadas (baseado no `git status` do passo 1)
- Que o push foi realizado com sucesso para `lucassantana-QA-Onfly/skills-claude`
- A mensagem de commit utilizada
