---
name: create-mr
description: "Gera localmente o título e a descrição de um Merge Request em Markdown, salvando o arquivo MR-<branch>.md na raiz do projeto. Não faz push nem abre MR no GitLab."
---

# Create MR — Gerador de Descrição de Merge Request

## Objetivo

Analisar os commits e o diff da branch atual em relação a uma branch de destino e gerar um arquivo `MR-<nome-da-branch>.md` na raiz do projeto com título e descrição prontos para copiar ao abrir o MR.

Nenhum push é feito. Nenhuma chamada à API do GitLab é feita. O resultado é apenas o arquivo `.md` local.

---

## Passo 1 — Identificar a branch atual e a branch de destino

```bash
git rev-parse --abbrev-ref HEAD
```

Guarde o nome da branch atual (ex: `feature/minha-feature`).

Se o usuário não informou a branch de destino, use `develop`. Se `develop` não existir no repositório, use `main`.

Para verificar se a branch existe:

```bash
git rev-parse --verify origin/develop 2>/dev/null && echo "existe" || echo "nao existe"
```

---

## Passo 2 — Calcular o merge-base

```bash
git merge-base origin/<branch-destino> HEAD
```

Se `origin/<branch-destino>` não existir, tente sem o prefixo `origin/`:

```bash
git merge-base <branch-destino> HEAD
```

Se nenhum funcionar, use `HEAD~1` como fallback.

Guarde o hash retornado.

---

## Passo 3 — Coletar os dados do diff

Com o hash do merge-base, execute:

```bash
# Commits incluídos
git log <merge-base>..HEAD --oneline

# Arquivos alterados
git diff <merge-base>...HEAD --name-only | sort | uniq

# Estatísticas resumidas
git diff <merge-base>...HEAD --stat | head -50
```

Para extrair os ranges de linhas modificadas por arquivo, execute:

```bash
git diff <merge-base>...HEAD 2>/dev/null | awk '
/^diff --git/ { file=$3; sub(/^a\//, "", file) }
/^@@ / {
  match($0, /\+([0-9]+)(,([0-9]+))?/, arr)
  start = arr[1]+0
  count = (arr[3] != "" ? arr[3]+0 : 1)
  if (count == 0) next
  end = start + count - 1
  range = (count == 1 ? start : start"-"end)
  ranges[file] = (ranges[file] == "" ? range : ranges[file] " | " range)
}
END { for (f in ranges) printf "%s\t%s\n", f, ranges[f] }
' | sort
```

---

## Passo 4 — Gerar o título do MR

Com base nos commits e nos arquivos alterados, crie um título descritivo para o MR seguindo estas regras:

- Máximo de 72 caracteres
- Comece com um verbo no infinitivo em português (ex: `Adiciona`, `Corrige`, `Refatora`, `Remove`, `Atualiza`)
- Seja específico sobre o que foi feito, não genérico
- Não use prefixos de conventional commits (`feat:`, `fix:` etc.) — o título do MR é para humanos

Exemplos de bons títulos:
- `Adiciona autenticação por magic link no fluxo de login`
- `Corrige cálculo de desconto quando cupom é aplicado com frete grátis`
- `Refatora hook useCart para usar TanStack Query`

---

## Passo 5 — Gerar a descrição do MR

Com base nos commits, arquivos alterados, ranges e diff, monte a descrição seguindo **exatamente** esta estrutura Markdown:

```markdown
## O que foi feito

- <bullet point descrevendo cada mudança principal>
- <bullet point>
- ...

---

## Arquivos alterados

| Arquivo | Linhas modificadas |
|---------|-------------------|
| `caminho/do/arquivo.ts` | 45-67 \| 120 |
| `caminho/outro/arquivo.tsx` | 10-25 |

---

## Motivação

<Parágrafo explicando o porquê da mudança: qual problema resolve, qual melhoria traz, ou qual requisito atende. Base isso nos commits e no contexto do diff.>

---

## Como testar

- [ ] <passo de teste 1: ação concreta e resultado esperado>
- [ ] <passo de teste 2>
- [ ] <passo de teste 3>
```

**Regras para cada seção:**

- **O que foi feito**: bullet points concisos, um por mudança lógica. Não repita o título.
- **Arquivos alterados**: use os ranges extraídos no Passo 3. Formato da coluna: `45-67 | 120` (pipe separando ranges).
- **Motivação**: 2–4 frases. Foque no "porquê", não no "o quê" (isso já está acima).
- **Como testar**: passos práticos que um desenvolvedor ou QA consegue executar. Seja específico.

---

## Passo 6 — Salvar o arquivo

Monte o arquivo final com este formato e salve como `MR-<nome-da-branch>.md` na raiz do projeto:

```markdown
# <TÍTULO DO MR>

> Branch: `<branch-atual>` → `<branch-destino>`
> Merge-base: `<hash>`

---

## O que foi feito
...

---

## Arquivos alterados
...

---

## Motivação
...

---

## Como testar
...
```

Após salvar, informe ao usuário:
- O caminho do arquivo gerado
- O título escolhido
- Quantos arquivos e commits foram incluídos na análise

---

## Notas importantes

- **Sem push, sem GitLab**: este skill apenas gera o arquivo local. Nunca tente fazer `git push` ou chamar a API do GitLab.
- **Branch de destino padrão**: use `develop` se existir, caso contrário `main`. Aceite override do usuário.
- **Fallback de título**: se não conseguir gerar um título descritivo, use o nome da branch formatado.
- **Arquivo de saída**: sempre na raiz do projeto (mesmo nível do `.git`), nunca em subpastas.
- **Se não houver commits**: informe ao usuário que não há diferença entre a branch atual e a de destino e encerre sem criar o arquivo.
