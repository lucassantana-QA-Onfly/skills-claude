# Skills Claude — Onfly QA

Skills customizadas do [Claude Code](https://claude.ai/code) usadas pelo time de QA da Onfly para automatizar tarefas recorrentes no fluxo de qualidade.

---

## Sumário

- [/criar-bug](#criar-bug)
- [/jira-testes](#jira-testes)
- [/jira-estimativa](#jira-estimativa)
- [/gitlab-impacto](#gitlab-impacto)
- [/atualizar-skills](#atualizar-skills)

---

## /criar-bug

Registra um bug no Jira de acordo com o time responsável pela tarefa.

**Uso:**
```
/criar-bug PROJ-123
```
> A chave da issue pode ser omitida — a skill detecta pelo contexto da conversa.

**Fluxo por time:**

| Time | Ação |
|------|------|
| **Travel (TRPO)** | Comenta o bug na tarefa original, marca "Need fix" e notifica via Google Chat |
| **Onhappy (DLT, CHEER)** | Cria ticket Fix no projeto TEST, vincula como "causes" na tarefa original e notifica via Google Chat |

**Destaques:**
- Solicita confirmação antes de cada ação destrutiva
- Passo a passo de reprodução incluído automaticamente na descrição
- Notificação no Google Chat com card, responsável e link (sem expor o conteúdo do bug)

---

## /jira-testes

Gera casos de teste a partir de uma demanda no Jira e os anexa na card correspondente.

**Uso:**
```
/jira-testes PROJ-123
```

**Fluxo:**
1. Busca a issue no Jira (título, descrição, critérios de aceitação)
2. Analisa imagens anexadas, se houver
3. Seleciona técnicas de teste adequadas (particionamento de equivalência, análise de limites, tabela de decisão, etc.)
4. Gera casos ordenados por criticidade (Crítica → Baixa), sem redundâncias
5. Apresenta ao usuário e solicita confirmação antes de anexar na card

---

## /jira-estimativa

Estima o esforço de teste de uma demanda usando a técnica PERT (três pontos). Somente leitura — não salva nada no Jira.

**Uso:**
```
/jira-estimativa PROJ-123
```

**Atividades estimadas:**
- Análise e planejamento
- Escrita de casos de teste
- Execução
- Reteste
- Documentação

A fórmula `E = (O + 4M + P) / 6` é aplicada por atividade, com cálculo de desvio padrão (σ) para indicar margem de incerteza.

---

## /gitlab-impacto

Consulta o card no Jira, localiza o link do GitLab associado e gera uma análise de impacto das mudanças de código. Somente leitura.

**Uso:**
```
/gitlab-impacto PROJ-123
```

**O relatório inclui:**
- Resumo das mudanças em linguagem de negócio
- Tabela de arquivos alterados com nível de impacto (Alto / Médio / Baixo)
- Explicação detalhada por arquivo
- Avaliação de riscos e dependências
- Metadados dos commits

> Utiliza a variável de ambiente `$GITLAB_PERSONAL_ACCESS_TOKEN` — o token nunca é exposto na saída.

---

## /atualizar-skills

Sincroniza as skills locais com este repositório no GitHub. Deve ser executada sempre que uma skill for criada ou atualizada.

**Uso:**
```
/atualizar-skills
/atualizar-skills "mensagem de commit customizada"
```

**Fluxo:**
1. Verifica se há mudanças locais (`git status`)
2. Gera mensagem de commit automática listando as skills modificadas (ex: `update: criar-bug, jira-testes`)
3. Faz commit e push para o branch `master` usando `$GITHUB_TOKEN`

---

## Estrutura do repositório

```
skills-claude/
├── atualizar-skills/
│   └── skill.md
├── criar-bug/
│   └── skill.md
├── gitlab-impacto/
│   └── SKILL.md
├── jira-estimativa/
│   └── SKILL.md
└── jira-testes/
    └── SKILL.md
```

---

## Requisitos

- [Claude Code](https://claude.ai/code) instalado e configurado
- Variáveis de ambiente:
  - `$GITHUB_TOKEN` — para a skill `/atualizar-skills`
  - `$GITLAB_PERSONAL_ACCESS_TOKEN` — para a skill `/gitlab-impacto`
- Acesso ao Jira via MCP Atlassian configurado no Claude Code
