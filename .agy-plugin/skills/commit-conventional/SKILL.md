---
name: commit-conventional
description: Use when the user wants to create a git commit — analyzes staged changes, determines type and scope following Conventional Commits, and writes the message in Brazilian Portuguese with format tipo(escopo): pequena_descrição. Triggers on "faz o commit", "commita", "gera o commit", "commit das alterações".
---

# Commit Conventional

## Objetivo

Analisar as alterações staged no git e gerar um commit seguindo o padrão **Conventional Commits** em **Português do Brasil**, no formato:

```text
tipo(escopo): pequena_descrição
```

---

## Passo 1 — Verificar o estado do repositório

Execute os dois comandos abaixo:

```bash
git status
git diff --staged
```

Se **não houver arquivos staged** (`git status` mostrar apenas unstaged ou untracked), pergunte ao usuário:

> "Nenhum arquivo está staged. Deseja que eu adicione todos os arquivos modificados (`git add -A`) ou prefere selecionar quais adicionar?"

Aguarde a resposta. Só prossiga após ter arquivos staged.

---

## Passo 2 — Determinar o tipo

Analise o diff staged e classifique com base no que foi alterado:

| Tipo | Quando usar |
| --- | --- |
| `feat` | Nova funcionalidade, novo componente, nova página, nova rota |
| `fix` | Correção de bug, comportamento incorreto, erro em lógica |
| `refactor` | Reorganização de código sem mudança de comportamento (ex: migração feature-based) |
| `style` | Formatação, espaçamento, ordem de imports — sem mudança de lógica |
| `docs` | Alterações em arquivos `.md`, comentários, documentação |
| `test` | Adição ou correção de testes |
| `chore` | Configurações, dependências, scripts, arquivos de build |
| `perf` | Melhorias de performance sem mudança de comportamento |
| `ci` | Alterações em pipelines CI/CD (`.gitlab-ci.yml`, `.github/workflows/`) |
| `revert` | Reversão de um commit anterior |

**Em caso de dúvida entre dois tipos**, prefira o de maior impacto (ex: `feat` > `refactor`).

---

## Passo 3 — Determinar o escopo

O escopo identifica **qual parte do sistema** foi alterada. Derive do caminho dos arquivos modificados:

| Arquivos modificados | Escopo sugerido |
| --- | --- |
| `src/features/auth/` | `auth` |
| `src/features/contagens/` | `contagens` |
| `src/features/equipamentos/` | `equipamentos` |
| `src/core/http/` | `http` |
| `src/core/offline/` | `offline` |
| `src/shared/` | `shared` |
| `src/app/` | `app` |
| Arquivos de config raiz (`.eslintrc`, `vite.config.ts`, `tsconfig.json`) | `config` |
| `package.json`, `package-lock.json` | `deps` |
| Arquivos `.md` de documentação | `docs` |
| Skills (`skills/`, `plugins/`, `.agy-plugin/`) | `skills` |
| Múltiplas features sem relação clara | omitir escopo |

Se os arquivos alterados cobrirem **mais de um domínio sem relação**, omita o escopo:

```text
refactor: separa lógica de apresentação nas páginas principais
```

---

## Passo 4 — Escrever a descrição

A descrição deve ser:

- Em **Português do Brasil**
- No **modo imperativo** (verbo no presente que indica ação): `adiciona`, `corrige`, `remove`, `extrai`, `move`, `atualiza`, `implementa`, `cria`
- **Curta**: máximo 72 caracteres na linha completa (tipo + escopo + descrição)
- **Sem ponto final**
- **Sem maiúsculas** no início (exceto siglas: `API`, `Redux`, `PWA`)

### Exemplos corretos

```text
feat(auth): adiciona tela de login com validação Zod
fix(contagens): corrige cálculo de total de itens contados
refactor(contagens): migra ContagemEmAndamento para feature-based
chore(deps): atualiza react-query para v5
docs(skills): adiciona skill de commit conventional
style(shared): aplica formatação prettier nos utilitários
test(auth): adiciona testes unitários para useLogin
```

### Exemplos incorretos

```text
feat(auth): Adiciona tela de login          ← maiúscula no início
fix(contagens): corrigido o cálculo         ← particípio, não imperativo
refactor: refatoração da feature contagens  ← substantivo, não verbo
feat(auth): adiciona tela de login com validação de formulário usando Zod e React Hook Form  ← muito longo
```

---

## Passo 5 — Confirmar antes de commitar

Apresente a mensagem gerada e peça confirmação:

> Mensagem do commit:
>
> ```text
> tipo(escopo): pequena_descrição
> ```
>
> Confirma? (s/n — ou sugira uma alteração)

Se o usuário pedir ajuste, refaça a mensagem e confirme novamente.

---

## Passo 6 — Executar o commit

Com a confirmação do usuário, execute:

```bash
git commit -m "tipo(escopo): pequena_descrição"
```

Após o commit, informe o hash gerado:

> Commit realizado: `abc1234 — tipo(escopo): pequena_descrição`

---

## Diretrizes gerais

- **Nunca use `git add -A` sem perguntar** — o usuário pode não querer commitar todos os arquivos.
- **Não faça push** — apenas commit, a menos que o usuário peça explicitamente.
- **Um commit por assunto** — se os arquivos staged cobrirem assuntos muito distintos, sugira dividir em commits separados.
- **Não invente escopo** — se não conseguir derivar do caminho dos arquivos, omita o escopo.
