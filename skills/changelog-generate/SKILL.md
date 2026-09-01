---
name: changelog-generate
description: Use when the user wants to generate a CHANGELOG file — compares current branch against master, collects all commits, determines the next SemVer version (major/minor/patch) based on Conventional Commits, and saves a CHANGELOG-{version}.md file. Triggers on "gera o changelog", "cria o changelog", "gerar changelog", "versão do changelog".
---

# Changelog Generate

## Objetivo

Comparar a branch atual com `master`, analisar todos os commits incluídos, determinar o próximo número de versão pelo **SemVer** e gerar o arquivo `CHANGELOG-{versão}.md` em **Português do Brasil**.

---

## Passo 1 — Identificar a versão atual

Tente obter a versão atual pela seguinte ordem de prioridade:

```bash
# 1. Última tag git de versão
git tag --sort=-v:refname | head -1

# 2. Versão no package.json (se existir)
cat package.json | grep '"version"' | head -1

# 3. Buscar CHANGELOGs existentes para inferir a última versão
ls CHANGELOG-*.md 2>/dev/null | sort -V | tail -1
```

Se não encontrar nenhuma versão, pergunte ao usuário:

> "Não encontrei a versão atual do projeto. Qual é a versão atual? (ex: 1.25.0)"

Aguarde resposta antes de prosseguir.

---

## Passo 2 — Coletar os commits entre a branch e master

Execute os comandos abaixo para encontrar o ponto de divergência e listar todos os commits:

```bash
# Encontrar o commit base (onde a branch divergiu de master)
git merge-base HEAD master

# Listar todos os commits desde o merge-base até HEAD
git log <merge-base>..HEAD --pretty=format:"%H|%s|%an|%ad" --date=short

# Listar as branches que já foram mergeadas nesta branch (se for uma branch de release)
git branch --merged HEAD --no-column | grep -v "^\*"
```

Se estiver em uma **branch de release** que agrega várias feature branches, colete também os títulos das PRs/MRs mergeadas:

```bash
git log <merge-base>..HEAD --pretty=format:"%s" | grep -E "^Merge (branch|pull request)"
```

---

## Passo 3 — Classificar cada commit

Para cada commit coletado, classifique com base no prefixo da mensagem (Conventional Commits):

| Prefixo do commit | Categoria no changelog | Impacto SemVer |
| --- | --- | --- |
| `feat:` / `feat(escopo):` | ✨ Novas Funcionalidades | **MINOR** |
| `fix:` / `fix(escopo):` | 🐛 Correções de Bug | **PATCH** |
| `perf:` / `perf(escopo):` | ⚡ Melhorias de Performance | **PATCH** |
| `refactor:` / `refactor(escopo):` | ♻️ Refatorações | **PATCH** |
| `docs:` / `docs(escopo):` | 📝 Documentação | **PATCH** |
| `style:` / `style(escopo):` | 💄 Estilo | **PATCH** |
| `test:` / `test(escopo):` | ✅ Testes | **PATCH** |
| `chore:` / `chore(escopo):` | 🔧 Manutenção | **PATCH** |
| `ci:` / `ci(escopo):` | 👷 CI/CD | **PATCH** |
| `build:` / `build(escopo):` | 📦 Build | **PATCH** |
| `BREAKING CHANGE` no corpo | 💥 Quebras de Compatibilidade | **MAJOR** |
| `feat!:` / `fix!:` (com `!`) | 💥 Quebras de Compatibilidade | **MAJOR** |
| Commits sem prefixo convencional | 🔀 Outros | **PATCH** |

---

## Passo 4 — Determinar a próxima versão (SemVer)

Com base nos commits coletados, aplique a regra mais alta encontrada:

```
MAJOR.MINOR.PATCH
```

**Regras (em ordem de prioridade):**

1. **MAJOR** — se qualquer commit contiver `BREAKING CHANGE` no corpo ou `!` após o tipo (ex: `feat!:`, `fix!:`)
   - Incrementa o primeiro número, zera os outros
   - Ex: `1.25.3` → `2.0.0`

2. **MINOR** — se não houver MAJOR e existir ao menos um commit `feat:`
   - Incrementa o segundo número, zera o terceiro
   - Ex: `1.25.3` → `1.26.0`

3. **PATCH** — se não houver MAJOR nem MINOR (apenas `fix:`, `refactor:`, `chore:`, etc.)
   - Incrementa apenas o terceiro número
   - Ex: `1.25.3` → `1.25.4`

**Apresente a decisão ao usuário antes de gerar o arquivo:**

> **Versão atual:** `1.25.0`
> **Próxima versão:** `1.26.0` — **MINOR**
>
> **Motivo:** Foram encontrados X commits do tipo `feat:`, o que indica adição de novas funcionalidades sem quebra de compatibilidade.
>
> Confirma? (s/n — ou informe outra versão)

Aguarde confirmação antes de gerar o arquivo.

---

## Passo 5 — Gerar o arquivo CHANGELOG

Salve o changelog como `CHANGELOG-{versão}.md` na raiz do projeto.

Use **exatamente** esta estrutura:

```markdown
# CHANGELOG — v{versão}

> **Tipo de versão:** MINOR / PATCH / MAJOR
> **Motivo:** {explicação da decisão SemVer em 1–2 frases}
> **Data:** {data atual dd/mm/aaaa}
> **Branch base:** master
> **Branch de origem:** {nome da branch atual}

---

## 💥 Quebras de Compatibilidade

{listar apenas se houver BREAKING CHANGEs}

- **{escopo}:** {descrição da quebra} (`{hash curto}`)

---

## ✨ Novas Funcionalidades

- **{escopo}:** {descrição} (`{hash curto}`)

---

## 🐛 Correções de Bug

- **{escopo}:** {descrição} (`{hash curto}`)

---

## ⚡ Melhorias de Performance

- **{escopo}:** {descrição} (`{hash curto}`)

---

## ♻️ Refatorações

- **{escopo}:** {descrição} (`{hash curto}`)

---

## 🔧 Manutenção e Infraestrutura

{agrupa chore, ci, build, docs, style, test}

- **{escopo}:** {descrição} (`{hash curto}`)

---

## 🔀 Outros

{commits sem prefixo convencional}

- {descrição} (`{hash curto}`)

---

## Resumo

| Categoria | Quantidade |
| --- | --- |
| 💥 Quebras de Compatibilidade | X |
| ✨ Novas Funcionalidades | X |
| 🐛 Correções de Bug | X |
| ⚡ Performance | X |
| ♻️ Refatorações | X |
| 🔧 Manutenção | X |
| **Total de commits** | X |
```

**Regras de escrita das entradas:**

- Se o commit tem escopo: `**auth:** adiciona validação de token expirado (`abc1234`)`
- Se não tem escopo: `adiciona validação de token expirado (`abc1234`)`
- Omita seções vazias (sem commits naquela categoria)
- Hash curto = primeiros 7 caracteres do SHA

---

## Passo 6 — Informar o resultado

Após salvar o arquivo, informe ao usuário:

> Changelog gerado: `CHANGELOG-{versão}.md`
>
> **Versão:** `{versão anterior}` → `{nova versão}` ({MAJOR/MINOR/PATCH})
> **Commits incluídos:** {total}
> **Arquivo salvo em:** `./CHANGELOG-{versão}.md`

---

## Diretrizes gerais

- **Não altere o código** — esta skill apenas lê o histórico git e gera documentação.
- **Omita seções vazias** — não inclua categorias sem commits no arquivo.
- **Commits de merge** (ex: `Merge branch 'feat/...'`) são ignorados no changelog — use apenas o histórico real de commits.
- **Se não houver commits convencionais**, trate todos como PATCH e liste em "🔀 Outros".
- **Se a branch atual for `master` ou `main`**, avise o usuário: o changelog compara com o histórico da própria branch — certifique-se de estar na branch de release correta.
- **Não faça push, tag ou merge** — apenas gera o arquivo. O usuário decide o que fazer com ele.
