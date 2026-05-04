---
name: pr-code-review
description: "Code review, PR review, revisar código, checar bugs — analisa a branch atual vs develop (merge-base) em arquivos .ts/.tsx/.js/.jsx e gera relatório Markdown em PT-BR com problemas classificados por severidade: CRITICO, AVISO, SUGESTAO."
---

# PR Code Review

## Objetivo
Revisar todas as mudancas da branch atual em relacao a `develop`, identificando bugs, problemas de performance e violacoes de boas praticas. O relatorio e gerado em Portugues do Brasil e salvo como arquivo `.md` na raiz do projeto.

## Passo 1 — Encontrar o merge-base

Execute para encontrar o commit exato onde a branch atual foi criada a partir de `develop`:

```bash
git merge-base HEAD develop
```

Guarde o hash retornado (ex: `a3f9c12`). Este e o ponto de comparacao correto — garante que voce analisa apenas o que a branch atual adicionou, ignorando commits que chegaram na `develop` depois.

## Passo 2 — Obter o diff filtrado

Liste os arquivos alterados (apenas `.ts`, `.tsx`, `.js`, `.jsx`):

```bash
git diff <merge-base> HEAD --name-only -- '*.ts' '*.tsx' '*.js' '*.jsx'
```

Obtenha o diff completo com contexto:

```bash
git diff <merge-base> HEAD --unified=5 -- '*.ts' '*.tsx' '*.js' '*.jsx'
```

## Passo 3 — Extrair ranges de linhas

Para cada arquivo, extraia os hunks do diff. O formato do diff mostra hunks assim:

```
@@ -115,10 +115,12 @@ function exemplo() {
```

A parte `+a,b` indica: a partir da linha `a`, foram modificadas/adicionadas `b` linhas. Entao o range e `a` ate `a+b-1`.

Se `b` e 0 ou 1, a linha unica e `a`.

Consolide hunks proximos (distancia <= 3 linhas) em um unico range para evitar fragmentacao excessiva.

**Formato de saida dos ranges:**
- Multiplos ranges: `115-125 | 150-180`
- Linha unica: `151`
- Range de uma linha so: `200-200` → exibir como `200`

## Passo 4 — Ler o conteudo para analise

Para cada arquivo alterado, leia o arquivo completo ou as secoes relevantes para entender o contexto ao redor das mudancas. Voce precisa de contexto suficiente para avaliar:
- O que a funcao/componente faz no geral
- Como as linhas alteradas se encaixam no fluxo
- Se dependencias externas sao usadas corretamente

## Passo 5 — Analisar as mudancas

Para cada arquivo, analise as linhas alteradas (e seu contexto) buscando:

### Bugs (CRITICO ou AVISO)
- Erros de logica que causam comportamento incorreto
- Acesso a propriedades de valores potencialmente `null`/`undefined` sem guard
- Condicoes de corrida (race conditions) em operacoes assincronas
- Mutacao direta de estado (em React/Redux)
- Dependencias faltando em hooks (`useEffect`, `useCallback`, `useMemo`)
- Memoria nao liberada (listeners, timers, subscricoes sem cleanup)
- Tratamento de erro ausente em operacoes criticas (chamadas API, parse de JSON)
- Comparacoes com `==` onde `===` seria correto
- Variaveis nao inicializadas usadas antes de atribuicao

### Performance (AVISO ou SUGESTAO)
- Objetos/arrays/funcoes criados dentro do render sem memoizacao
- Renderizacoes desnecessarias por falta de `React.memo`, `useMemo`, `useCallback`
- Loops aninhados com complexidade O(n²) ou pior onde existe alternativa simples
- Chamadas de API duplicadas ou dentro de loops
- Imports de bibliotecas inteiras quando so um submodulo e necessario
- Re-criacao de regex dentro de loops ou renders

### Boas Praticas (AVISO ou SUGESTAO)
- Tipos TypeScript ausentes, uso excessivo de `any` ou `as unknown as X`
- Funcoes/componentes muito longos (>100 linhas) sem divisao logica clara
- Codigo duplicado que poderia ser extraido
- Nomes de variaveis/funcoes pouco descritivos
- Constantes magicas sem nomeacao
- `console.log` deixados no codigo
- Comentarios desatualizados ou enganosos
- Props nao tipadas em componentes React
- `useEffect` com logica complexa que poderia ser um custom hook

## Passo 6 — Gerar o relatorio

Monte o conteudo do relatorio seguindo a estrutura abaixo e salve como arquivo Markdown na raiz do projeto com o nome `code-review-<nome-da-branch>.md`. Apos salvar, informe ao usuario o caminho do arquivo gerado.

O arquivo deve ter o seguinte conteudo:

---

```markdown
# Relatório de Code Review

| Campo       | Valor                          |
|-------------|--------------------------------|
| Branch      | <nome da branch>               |
| Branch base | develop                        |
| Merge-base  | <hash do merge-base>           |
| Data        | <data atual dd/mm/aaaa>        |

---

## Resumo das Alterações

<Paragrafo descrevendo o que a PR faz em linhas gerais: qual
funcionalidade adiciona, qual bug corrige, qual refatoracao
realiza. Base isso no contexto dos arquivos alterados.>

---

## Arquivos Alterados

| # | Arquivo | Linhas alteradas |
|---|---------|-----------------|
| 1 | `src/modules/Auth/Login.tsx` | 45-67 \| 120 |
| 2 | `src/shared/hooks/useAuth.ts` | 10-25 \| 88-102 |

**Total:** X arquivo(s) | Y hunk(s) de alteração

---

## Problemas Encontrados

### 🔴 CRÍTICO

#### #1 — <Titulo curto e descritivo>

**Arquivo:** `src/modules/Auth/Login.tsx` — Linhas: 45-52

**Problema:**
<Descricao clara do problema observado no codigo>

**Por que é um problema:**
<Explicacao do impacto: o que pode dar errado, em qual cenario,
qual o risco para o usuario ou sistema>

**Sugestão de correção:**
<Instrucao de como corrigir. Se for codigo, use bloco de codigo formatado>

---

### 🟡 AVISO

#### #2 — <Titulo>

**Arquivo:** `...` — Linhas: ...

...

---

### 🔵 SUGESTÃO

#### #3 — <Titulo>

**Arquivo:** `...` — Linhas: ...

...

---

## Resumo Final

| Severidade | Quantidade |
|-----------|------------|
| 🔴 Crítico   | X |
| 🟡 Aviso     | Y |
| 🔵 Sugestão  | Z |
| **Total**    | W |

> <Se nao houver problemas criticos:>
> ✅ A PR pode ser aprovada com atenção aos avisos listados.
>
> <Se houver problemas criticos:>
> ❌ A PR requer correções antes de ser aprovada.

---

## Sugestões de Testes

### O que deve ser testado (funcionalidades implementadas)

<Lista dos cenarios de teste feliz (happy path) relacionados ao que a PR
implementou. Para cada funcionalidade nova ou modificada, descreva o que
o testador deve verificar manualmente ou via testes automatizados.>

Exemplo de formato:
- [ ] <Cenario de teste 1: descreva a acao e o resultado esperado>
- [ ] <Cenario de teste 2>

### O que pode ter quebrado (regressões)

<Lista de funcionalidades preexistentes que podem ter sido afetadas pelas
mudancas desta PR — mesmo que indiretamente. Pense em: componentes que
importam os arquivos alterados, fluxos que dependem dos comportamentos
modificados, edge cases que a mudanca pode ter alterado.>

Exemplo de formato:
- [ ] <Risco de regressao 1: descreva o que pode ter quebrado e como verificar>
- [ ] <Risco de regressao 2>
```

---

## Notas importantes

- **Arquivo de saida**: salve o relatorio como `code-review-<nome-da-branch>.md` na raiz do projeto (mesmo nivel do `package.json`). Apos salvar, informe o caminho completo ao usuario.
- **Foco no diff**: analise apenas o codigo que foi adicionado ou modificado (linhas com `+` no diff). Nao reporte problemas em codigo pre-existente que nao foi tocado pela PR, a menos que a mudanca da PR introduza uma interacao problematica com ele.
- **Contexto e rei**: antes de reportar um problema, considere se o padrao pode ser intencional dado o contexto do projeto (ex: certos padroes de performance podem ser aceitaveis em componentes nao criticos).
- **Sem falsos positivos**: e melhor reportar menos problemas reais do que muitos problemas duvidosos. Se nao tiver certeza, omita ou classifique como SUGESTAO com ressalva.
- **Codigo de correcao**: quando a correcao for codigo, mostra o trecho relevante corrigido, nao o arquivo inteiro.
- **Testes**: a secao de sugestoes de teste deve ser pratica e objetiva — cenarios reais que um QA ou o proprio desenvolvedor consiga executar manualmente em poucos minutos.
- **Severidades**:
  - `CRITICO` — pode causar bug, crash, falha de seguranca ou perda de dados
  - `AVISO` — degradacao de performance, comportamento inesperado em edge cases, violacao de padrao do projeto
  - `SUGESTAO` — melhoria de legibilidade, manutencao ou organizacao sem impacto funcional imediato
