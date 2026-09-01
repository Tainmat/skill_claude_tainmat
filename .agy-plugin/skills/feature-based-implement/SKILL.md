---
name: feature-based-implement
description: Use when the user wants to refactor or migrate React/Next.js code to feature-based architecture — symptoms include code scattered across /pages, /components, /hooks, /api, /helpers by file type instead of by domain, or when they describe a feature to be reorganized following the View + Hook Controller pattern (Component.tsx + useComponent.ts + index.ts).
---

# Feature-Based Implement

## Objetivo

Analisar o projeto React/Next.js atual e gerar o plano de refatoração de uma feature para a arquitetura **Feature-Based / Domain-Driven**, seguindo o padrão **View + Hook Controller** (`Component.tsx` + `useComponent.ts` + `index.ts`).

A resposta é gerada em **Português do Brasil** e o plano é salvo como arquivo `.md` na raiz do projeto.

---

## Passo 1 — Fazer as perguntas obrigatórias

Sempre inicie com **estas duas perguntas juntas**, em uma única mensagem:

> **1. Qual feature ou tela vamos migrar?**
> Descreva a funcionalidade ou informe o nome da página/componente atual.
>
> **2. Onde ela está hoje no projeto?**
> Informe o caminho (ex: `src/pages/ContagemEmAndamento/`) ou diga se não sabe — nesse caso, informe palavras-chave para localizar.

Aguarde a resposta antes de prosseguir.

---

## Passo 2 — Mapear a estrutura atual do projeto

Execute os comandos abaixo para entender a estrutura do projeto:

```bash
# Estrutura raiz do src/
ls src/

# Se existirem, listar os diretórios principais
ls src/pages/ 2>/dev/null
ls src/components/ 2>/dev/null
ls src/hooks/ 2>/dev/null
ls src/api/ 2>/dev/null
ls src/features/ 2>/dev/null
```

Identifique qual tipo de arquitetura o projeto usa atualmente:

| Padrão detectado | Indicadores |
| --- | --- |
| **Por tipo** (legado) | Pastas `pages/`, `components/`, `hooks/`, `api/`, `helpers/` na raiz do `src/` |
| **Feature-based** (já migrado) | Pasta `features/` com subdomínios dentro do `src/` |
| **Misto** | Ambas as estruturas coexistindo |

Se o projeto **já usar feature-based**, identifique em qual feature a página se encaixa e pule para o Passo 4.

---

## Passo 3 — Localizar os arquivos da feature (se o usuário não souber)

Se o usuário não informou o caminho, solicite palavras-chave e busque nos seguintes locais **nesta ordem**:

1. `src/pages/` — telas principais
2. `src/components/` — componentes relacionados
3. `src/hooks/` — hooks vinculados
4. `src/api/` — chamadas de API
5. `src/redux/modules/` ou `src/store/` — slices Redux
6. `src/helpers/` ou `src/utils/` — utilitários relacionados

Para cada arquivo encontrado, identifique:

- O que ele faz (view, lógica, chamada HTTP, estado global, utilitário)
- Se é exclusivo da feature ou compartilhado com outras

**Confirme com o usuário** a lista de arquivos antes de prosseguir para a análise completa.

---

## Passo 4 — Identificar o domínio de negócio

Com base no propósito da feature, classifique em qual domínio ela pertence:

| Feature / Tela | Domínio sugerido |
| --- | --- |
| Login, seleção de acesso, autenticação | `auth` |
| Listagem, criação, operação principal do negócio | nome do conceito (ex: `contagens`, `pedidos`) |
| Consulta de detalhe, histórico de item | nome do conceito (ex: `equipamentos`, `produtos`) |
| Tutorial, onboarding, guias | `onboarding` |
| Delegação, atribuição entre usuários | `atribuicao` ou nome do conceito |

Se não se encaixar em nenhum, pergunte ao usuário qual nome de domínio faz mais sentido para o negócio.

---

## Passo 5 — Gerar a estrutura feature-based

Monte a nova estrutura de diretórios para a feature, seguindo este modelo:

```text
src/
└── features/
    └── {dominio}/
        ├── api/
        │   └── {dominio}.api.ts          # Funções de chamada HTTP (get, post, patch, etc.)
        ├── components/                    # Componentes exclusivos deste domínio
        │   └── {NomeComponente}/
        │       ├── {NomeComponente}.tsx
        │       ├── use{NomeComponente}.ts
        │       └── index.ts
        ├── pages/
        │   └── {NomePagina}/
        │       ├── {NomePagina}.tsx       # View pura — apenas JSX, zero lógica
        │       ├── use{NomePagina}.ts     # Hook controller — toda a lógica
        │       ├── {NomePagina}.styles.ts # Styled components (se usar)
        │       └── index.ts              # Barrel: export { NomePagina } from './NomePagina'
        ├── hooks/                         # Hooks de domínio reutilizáveis entre páginas
        ├── schemas/                       # Schemas de validação (Yup / Zod)
        ├── types/                         # Interfaces e tipos TypeScript do domínio
        └── store/                         # Redux slices específicos do domínio
```

### Regras de Separação

**`{NomePagina}.tsx` — View Pura:**

- Contém apenas JSX/HTML
- Importa e desestrutura o retorno do hook (`use{NomePagina}.ts`)
- Zero chamadas de API, zero lógica de negócio, zero Redux direto
- Props tipadas com interface exportada

**`use{NomePagina}.ts` — Hook Controller:**

- Toda a lógica: formulários, queries, mutations, Redux, efeitos
- Retorna apenas o necessário para a View (estados e handlers)
- Importa de `@core`, `@shared` e do próprio domínio

**`index.ts` — Barrel Export:**

```ts
export { NomePagina } from './NomePagina';
export type { NomePaginaProps } from './NomePagina'; // se houver props públicas
```

---

## Passo 6 — Mapear os arquivos atuais para a nova estrutura

Gere uma tabela De-Para com **todos** os arquivos envolvidos na feature:

```markdown
| Arquivo Atual | Nova Localização | Camada | Observação |
| --- | --- | --- | --- |
| `src/pages/NomeTela/index.tsx` | `src/features/{dominio}/pages/NomeTela/NomeTela.tsx` | View | Separar lógica para o hook |
| `src/pages/NomeTela/index.tsx` | `src/features/{dominio}/pages/NomeTela/useNomeTela.ts` | Hook Controller | Extrair toda lógica |
| `src/api/getAlgo.ts` | `src/features/{dominio}/api/{dominio}.api.ts` | API | Mover função para o api.ts do domínio |
| `src/hooks/useAlgoEspecifico.ts` | `src/features/{dominio}/hooks/useAlgoEspecifico.ts` | Hook de Domínio | Exclusivo desta feature |
| `src/hooks/useAlgoGenerico.ts` | `src/shared/hooks/useAlgoGenerico.ts` | Shared | Usado por outras features |
| `src/redux/modules/algoSlice.ts` | `src/features/{dominio}/store/algo.slice.ts` | Store | Redux slice do domínio |
| `src/helpers/algoHelper.ts` | `src/features/{dominio}/utils/algo.helper.ts` | Utilitário | Exclusivo do domínio |
```

**Critério para Shared vs Feature:**

- Move para `shared/` se for usado por **2 ou mais domínios diferentes**
- Move para a feature se for **exclusivo** do domínio sendo migrado

---

## Passo 7 — Gerar o código refatorado

Para cada arquivo que precisa de **separação de View + Hook Controller**, gere o código completo dos três arquivos:

### Template: `useNomePagina.ts`

```ts
import { useCallback, useMemo, useEffect } from 'react';
// imports de @core, @shared, domínio

export function useNomePagina() {
  // Toda lógica aqui: useQuery, useMutation, useDispatch, useForm, etc.

  return {
    // Apenas o necessário para a View
    dado,
    isLoading,
    handleAcao,
  };
}
```

### Template: `NomePagina.tsx`

```tsx
import { useNomePagina } from './useNomePagina';
// imports de componentes de UI

export interface NomePaginaProps {
  // props externas (se houver)
}

export function NomePagina({ ...props }: NomePaginaProps) {
  const { dado, isLoading, handleAcao } = useNomePagina();

  return (
    // JSX puro — sem lógica, sem chamadas, sem useState aqui
  );
}
```

### Template: `index.ts`

```ts
export { NomePagina } from './NomePagina';
export type { NomePaginaProps } from './NomePagina';
```

---

## Passo 8 — Gerar o plano de migração

Salve o plano como `feature-migration-{nome-dominio}.md` na raiz do projeto com esta estrutura:

```markdown
# Plano de Migração: {Nome da Feature}

## Domínio
`src/features/{dominio}/`

## Arquivos Envolvidos

### Origem → Destino
[Tabela De-Para gerada no Passo 6]

## Nova Estrutura de Diretórios

[Árvore de diretórios gerada no Passo 5]

## Código Refatorado

### `use{NomePagina}.ts`
[código completo]

### `{NomePagina}.tsx`
[código completo]

### `index.ts`
[código completo]

[Repetir para cada componente/hook que precisar de refatoração]

## Checklist de Migração

### Criação de arquivos
- [ ] Criar estrutura de diretórios `src/features/{dominio}/`
- [ ] Criar `{NomePagina}.tsx` (View pura)
- [ ] Criar `use{NomePagina}.ts` (Hook Controller)
- [ ] Criar `index.ts` (Barrel export)
- [ ] Criar `{dominio}.api.ts` com funções HTTP

### Movimentação
- [ ] Mover hooks exclusivos para `src/features/{dominio}/hooks/`
- [ ] Mover hooks genéricos para `src/shared/hooks/`
- [ ] Mover tipos/interfaces para `src/features/{dominio}/types/`
- [ ] Mover Redux slices para `src/features/{dominio}/store/`
- [ ] Mover helpers exclusivos para `src/features/{dominio}/utils/`

### Atualização de imports
- [ ] Atualizar imports nas rotas (`src/app/routes.tsx` ou equivalente)
- [ ] Atualizar imports em outros componentes que referenciam os arquivos movidos
- [ ] Verificar path aliases (`@features`, `@core`, `@shared`)

### Validação
- [ ] Verificar tipagem (`tsc --noEmit`)
- [ ] Verificar linting (`eslint src/features/{dominio}/`)
- [ ] Testar fluxo da feature manualmente
- [ ] Remover arquivos legados após validação

## Observações
[Pontos de atenção específicos da migração, como dependências circulares, slices compartilhados, etc.]
```

---

## Diretrizes gerais

- **Não invente código** — baseie a refatoração nos arquivos reais do projeto.
- **Preserve o comportamento** — a refatoração não deve alterar o que a feature faz, apenas reorganizar onde o código vive.
- **Componentes compartilhados vão para `shared/`** — nunca copie para dentro de uma feature código que é usado globalmente.
- **Não migre tudo de uma vez** — o plano deve poder ser executado em fases sem quebrar o que está em produção.
- **Confirme com o usuário** antes de gerar o código completo quando os arquivos foram identificados por busca de palavras-chave.
- **Se o projeto usa Next.js App Router**, também indique o arquivo `src/app/{rota}/page.tsx` correspondente que deve importar a página da feature.
