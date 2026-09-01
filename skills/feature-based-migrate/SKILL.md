---
name: feature-based-migrate
description: Use when the user points to a specific page or component and wants to physically migrate it to feature-based architecture — moving scattered files (hooks, api, redux, helpers) into a features/ domain folder and applying the View + Hook Controller pattern (Component.tsx + useComponent.ts + index.ts). Triggers when user says things like "aplica feature-based no Contagens", "migra essa page pra feature", "quero aplicar a arquitetura nessa tela".
---

# Feature-Based Migrate

## Objetivo

Dado um arquivo de página ou componente, migrar **tudo que pertence àquela feature** para a arquitetura feature-based, aplicando o padrão **View + Hook Controller** (`Component.tsx` + `useComponent.ts` + `index.ts`) e reorganizando os arquivos espalhados em `pages/`, `hooks/`, `api/`, `redux/`, `helpers/` para dentro de `src/features/{dominio}/`.

> Esta skill **executa** as mudanças. Não gera plano — gera código e move arquivos.

---

## Passo 1 — Confirmar o alvo

Se o usuário não informou o caminho exato, pergunte:

> "Qual o caminho da page que vamos migrar? (ex: `src/pages/Contagens/`)"

Com o caminho em mãos, leia o arquivo principal da page:

```bash
cat src/pages/{NomePage}/index.tsx   # ou o arquivo principal
```

---

## Passo 2 — Rastrear todas as dependências da feature

A partir dos imports do arquivo principal, trace **recursivamente** todos os arquivos pertencentes a esta feature.

### 2.1 Ler o arquivo da page e extrair imports

Leia o conteúdo completo da page. Para cada import identificado, classifique:

| Tipo de import | Onde costuma estar | Pertence à feature? |
| --- | --- | --- |
| Hook customizado específico | `src/hooks/useAlgo.ts` | Sim, se exclusivo |
| Chamada de API específica | `src/api/getAlgo.ts` | Sim, se exclusivo |
| Redux slice específico | `src/redux/modules/algoSlice.ts` | Sim, se exclusivo |
| Helper específico | `src/helpers/algoHelper.ts` | Sim, se exclusivo |
| Componente específico da tela | `src/components/CardAlgo/` | Sim, se exclusivo |
| Hook genérico | `src/hooks/useDebounce.ts` | Não — vai para `shared/` |
| Componente de UI genérico | `src/components/Button/` | Não — fica em `shared/` |
| Infraestrutura | `src/infraestructure/api.ts` | Não — fica em `core/` |

### 2.2 Para cada hook e componente encontrado, leia seus imports também

Rastreie 1 nível de profundidade adicional para garantir que nenhum arquivo relacionado fique para trás.

```bash
# Exemplo: encontrar todos os arquivos que a page importa
grep -r "from.*src/hooks/useContagens" src/ --include="*.ts" --include="*.tsx" -l
grep -r "from.*src/api/getContagens" src/ --include="*.ts" --include="*.tsx" -l
```

### 2.3 Montar a lista de arquivos para migrar

Antes de executar qualquer mudança, liste os arquivos identificados e confirme com o usuário:

```
Encontrei os seguintes arquivos relacionados à feature "{NomePage}":

📄 Page principal:
  - src/pages/Contagens/index.tsx

🪝 Hooks:
  - src/hooks/useContagens.ts
  - src/hooks/useCarregaContagem.ts

🌐 API:
  - src/api/getContagens.ts
  - src/api/postContagem.ts

🗄️ Redux:
  - src/redux/modules/contagensSlice.ts
  - src/redux/modules/contagemAtivaSlice.ts

🧩 Componentes exclusivos:
  - src/components/CardContagem/

🔧 Helpers:
  - src/helpers/mesclaContagem.ts

Prossigo com a migração?
```

Aguarde confirmação antes de continuar.

---

## Passo 3 — Determinar o domínio e a estrutura alvo

Com base no nome e propósito da page, defina o nome do domínio (em minúsculas, sem acento):

- `Contagens` → `contagens`
- `LoginFuncionario` → `auth`
- `DetalhesEquipamento` → `equipamentos`
- Em dúvida: use o nome da própria page em minúsculas

**Estrutura alvo a criar:**

```text
src/features/{dominio}/
├── api/
│   └── {dominio}.api.ts
├── components/
│   └── {NomeComponente}/
│       ├── {NomeComponente}.tsx
│       ├── use{NomeComponente}.ts
│       └── index.ts
├── hooks/
│   └── use{AlgoEspecifico}.ts
├── pages/
│   └── {NomePage}/
│       ├── {NomePage}.tsx        ← View pura
│       ├── use{NomePage}.ts      ← Hook Controller
│       └── index.ts              ← Barrel export
├── schemas/
├── types/
└── store/
    └── {algo}.slice.ts
```

---

## Passo 4 — Criar a estrutura de diretórios

Verifique se `src/features/` existe. Se não existir, crie:

```bash
mkdir -p src/features/{dominio}/api
mkdir -p src/features/{dominio}/components
mkdir -p src/features/{dominio}/hooks
mkdir -p src/features/{dominio}/pages/{NomePage}
mkdir -p src/features/{dominio}/schemas
mkdir -p src/features/{dominio}/types
mkdir -p src/features/{dominio}/store
```

---

## Passo 5 — Separar View + Hook Controller

### 5.1 Ler o arquivo original da page

Leia o conteúdo completo. Identifique:

- **Bloco de lógica**: `useState`, `useEffect`, `useSelector`, `useDispatch`, `useMutation`, `useQuery`, `useForm`, handlers, cálculos derivados
- **Bloco de apresentação**: o `return (...)` com JSX

### 5.2 Criar `use{NomePage}.ts` — Hook Controller

Extraia **toda a lógica** para o hook. Regras:

- Todo `useState`, `useEffect`, `useSelector`, `useDispatch`, `useMutation`, `useQuery`, `useForm` vai para o hook
- Handlers (`handleSubmit`, `handleIncrementa`, etc.) vão para o hook
- O hook retorna apenas o necessário para a View
- Atualize os imports para os novos caminhos dentro de `@features/{dominio}/`

```ts
// src/features/{dominio}/pages/{NomePage}/use{NomePage}.ts

import { ... } from '@features/{dominio}/api/{dominio}.api';
import { ... } from '@features/{dominio}/hooks/use{AlgoEspecifico}';

export function use{NomePage}() {
  // toda a lógica aqui

  return {
    // apenas o necessário para a View
  };
}
```

### 5.3 Criar `{NomePage}.tsx` — View Pura

A View recebe tudo do hook. Regras:

- Zero `useState`, `useEffect`, `useSelector`, `useDispatch` direto aqui
- Apenas JSX consumindo o retorno do hook
- Imports apenas de componentes de UI e do próprio hook

```tsx
// src/features/{dominio}/pages/{NomePage}/{NomePage}.tsx

import { use{NomePage} } from './use{NomePage}';

export function {NomePage}() {
  const { ... } = use{NomePage}();

  return (
    // JSX puro
  );
}
```

### 5.4 Criar `index.ts` — Barrel Export

```ts
// src/features/{dominio}/pages/{NomePage}/index.ts
export { {NomePage} } from './{NomePage}';
```

---

## Passo 6 — Mover os arquivos de suporte

Para cada arquivo da lista do Passo 2, mova para o destino correto:

| Arquivo original | Destino |
| --- | --- |
| `src/hooks/useAlgoEspecifico.ts` | `src/features/{dominio}/hooks/useAlgoEspecifico.ts` |
| `src/api/getAlgo.ts`, `postAlgo.ts` | consolidar em `src/features/{dominio}/api/{dominio}.api.ts` |
| `src/redux/modules/algoSlice.ts` | `src/features/{dominio}/store/algo.slice.ts` |
| `src/helpers/algoHelper.ts` | `src/features/{dominio}/utils/algo.helper.ts` |
| `src/components/CardAlgo/` | `src/features/{dominio}/components/CardAlgo/` |

**Ao mover cada arquivo**, atualize seus imports internos para refletir o novo caminho.

**Arquivos genéricos** (usados por outras features) **não são movidos** — apenas atualize os imports da feature para apontar para `@shared/`.

---

## Passo 7 — Atualizar imports no restante do projeto

Após mover os arquivos, localize todos os outros arquivos do projeto que importavam dos caminhos antigos e atualize para os novos:

```bash
# Exemplo: encontrar quem importava do caminho antigo
grep -r "from.*pages/Contagens" src/ --include="*.ts" --include="*.tsx" -l
grep -r "from.*hooks/useContagens" src/ --include="*.ts" --include="*.tsx" -l
grep -r "from.*api/getContagens" src/ --include="*.ts" --include="*.tsx" -l
```

Para cada arquivo encontrado, atualize o import para o novo caminho:

```ts
// Antes
import { Contagens } from '../pages/Contagens';

// Depois
import { Contagens } from '@features/contagens/pages/Contagens';
```

**Não esqueça de atualizar o arquivo de rotas** (`src/app/routes.tsx`, `src/app/Rotas/index.tsx` ou equivalente).

---

## Passo 8 — Remover arquivos originais

Somente após confirmar que os novos arquivos estão corretos e os imports atualizados, remova os arquivos originais:

```bash
rm src/pages/{NomePage}/index.tsx   # ou o arquivo original
# Remova a pasta se estiver vazia
rmdir src/pages/{NomePage}/ 2>/dev/null
```

Pergunte ao usuário antes de deletar:

> "Os arquivos novos estão criados e os imports atualizados. Posso remover os arquivos originais em `src/pages/{NomePage}/` e os hooks/api movidos?"

---

## Passo 9 — Validar

Execute a verificação de tipos para confirmar que nenhum import ficou quebrado:

```bash
npx tsc --noEmit
```

Se houver erros, corrija antes de encerrar. Os erros mais comuns são:

- Import apontando para caminho antigo que não existe mais → atualize o import
- Tipo não exportado corretamente → adicione `export type` no `index.ts`
- Path alias não configurado → verifique se `@features` está no `tsconfig.json` e `vite.config.ts`

---

## Diretrizes gerais

- **Confirme antes de deletar** — nunca remova arquivos originais sem aprovação do usuário.
- **Migre uma feature por vez** — não tente migrar múltiplas pages em paralelo.
- **Não quebre o que já funciona** — se um arquivo é compartilhado por outras features, não mova, apenas atualize o import.
- **Preserve o comportamento** — a refatoração reorganiza, não reescreve lógica de negócio.
- **Se o projeto não tiver path aliases** (`@features`, `@shared`), use caminhos relativos e informe o usuário sobre como configurar os aliases.
