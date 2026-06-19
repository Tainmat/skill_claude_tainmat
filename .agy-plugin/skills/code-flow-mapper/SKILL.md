---
name: code-flow-mapper
description: "Use esta skill quando o usuário precisar entender como uma funcionalidade existente funciona ANTES de alterá-la, mapeando a árvore de chamadas (call tree) bidirecional. Triggers: 'mapear o fluxo de', 'engenharia reversa de', 'preciso entender como X funciona antes de mexer', 'rastreia a call tree de', 'quem chama essa função', 'o que essa função usa', 'mostra o fluxo de execução'. Recebe um ponto de entrada (função, componente, hook, método) com arquivo e linha, rastreia downstream (o que o código chama) E upstream (quem chama o código), e gera um relatório Markdown em PT-BR com diagrama Mermaid e checklist de pontos de toque. NÃO use para diagnóstico de bug em produção, code review estilístico, geração de documentação geral, ou análise de impacto de alto nível (essa última pertence à skill impact-analysis)."
---

# Code Flow Mapper

## Objetivo

Mapear a **árvore de execução bidirecional** de um ponto de entrada no código, para que o usuário entenda exatamente como a funcionalidade acontece antes de alterá-la. A pergunta que esta skill responde é: _"Como esse pedaço de código executa, e o que mais preciso revisar quando eu mexer nele?"_

O mapeamento é **bidirecional**:

- **Downstream** (pra baixo): o que este código chama, depende e afeta.
- **Upstream** (pra cima): quem chama este código, quais arquivos importam, qual contrato é esperado.

O output é um relatório Markdown em PT-BR cuja profundidade se adapta ao tamanho e à criticidade do ponto de entrada.

### Diferença para `impact-analysis`

Se já houver no marketplace uma skill `impact-analysis`, distinga-as assim:

- **`impact-analysis`** — análise de alto nível antes da implementação: lista de arquivos afetados, pontos críticos, checklist de testes.
- **`code-flow-mapper`** — rastreio detalhado da **execução**: call tree linha-a-linha, hooks consumidos, side effects específicos, callers reais.

Se a intenção do usuário ficar ambígua, pergunte qual ele quer.

---

## Passo 1 — Coletar o ponto de entrada

Receba do usuário:

- Nome do símbolo (função, componente, hook, método, classe).
- Arquivo (`src/...`) e, se possível, linha aproximada.
- **Intenção da mudança** (opcional mas valioso): "vou trocar o retorno", "vou remover o cache", "vou mudar a validação". Se o usuário não disser, pergunte uma vez de forma curta. Se não responder ou disser "ainda não sei", siga em frente sem essa informação.

Abra o arquivo e localize a definição. Se houver ambiguidade (ex: dois símbolos com o mesmo nome), confirme com o usuário antes de prosseguir.

---

## Passo 2 — Calibrar profundidade do relatório

Antes de mapear, classifique o ponto de entrada em uma das categorias abaixo. Isso define o **estilo do output**:

| Categoria   | Sinais                                                                      | Estilo do relatório                                                                          |
| ----------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Pequena** | Função utilitária pura, < 30 linhas, sem efeitos colaterais, poucos callers | Sucinto: sem diagrama, foco em contrato e callers                                            |
| **Média**   | Componente ou hook com lógica de negócio, 30–150 linhas, alguns efeitos     | Padrão: diagrama simples + call tree + checklist                                             |
| **Grande**  | Componente container, orquestrador, função com múltiplos efeitos e ramos    | Completo: diagrama + tree downstream e upstream + estado compartilhado + checklist detalhado |

Não anuncie a categoria escolhida ao usuário — apenas use-a internamente para decidir o quanto entregar.

---

## Passo 3 — Rastreio Downstream (o que o código usa)

A partir da definição, leia o corpo e identifique:

1. **Chamadas de função relevantes** — entre nos arquivos correspondentes e siga até no máximo 4 níveis de profundidade. Pode ir mais fundo se um ramo específico for diretamente relevante para a mudança planejada.
2. **Hooks consumidos** (`useContext`, `useSelector`, hooks customizados) — anote qual estado/contexto leem.
3. **Imports de tipos e interfaces** — importam algum tipo que será impactado pela mudança?
4. **Efeitos colaterais**:
   - Network: `fetch`, `axios`, mutations, queries.
   - Estado: `setState`, `dispatch`, `set` de Zustand, mutações em refs.
   - Browser: `localStorage`, `sessionStorage`, `cookies`, `document.*`.
   - Navegação: `router.push`, `navigate`, `redirect`.
   - Side effects de bibliotecas: notificações, analytics, logs.

**Regras de poda:**

- Ignore métodos nativos triviais (`.map`, `.filter`, `.toString`) a menos que façam transformações centrais para a mudança.
- Se uma chamada vai para uma biblioteca externa (ex: `react-router-dom`, `lodash`), não tente abrir o código — apenas registre como "módulo externo: `nome`".
- Se um ramo claramente não é tocado pela mudança planejada, marque como "fora do escopo" e não aprofunde.

---

## Passo 4 — Rastreio Upstream (quem usa este código)

Esta é a parte mais crítica para análise de impacto e é frequentemente esquecida. Use as ferramentas disponíveis no ambiente (grep, ripgrep, busca do editor, ferramentas de busca textual do agente) para encontrar:

1. **Call sites diretos** — todos os arquivos que importam e chamam o símbolo.
2. **Re-exports** — o símbolo é re-exportado de algum `index.ts`? Se sim, callers podem estar referenciando o caminho do barrel, não o original.
3. **Testes** — arquivos `.test.ts(x)` ou `.spec.ts(x)` que cobrem o símbolo. A presença ou ausência de testes muda a estratégia da mudança.
4. **Mocks** — o símbolo aparece mockado em algum lugar? Se sim, mudar a assinatura quebra os mocks.
5. **Storybook ou fixtures** — para componentes, há stories ou dados de exemplo que usam o componente?

Para cada call site, registre **como** o símbolo é usado (qual argumento passa, o que faz com o retorno). Isso é o que permite avaliar se a mudança planejada quebra o caller.

---

## Passo 5 — Mapear estado compartilhado

Se o ponto de entrada **lê ou escreve** em estado global (Redux store, Context, Zustand, query cache do React Query, etc.), liste **outros consumidores desse mesmo estado**. Eles não chamam a função diretamente, mas podem ser afetados se a forma do estado mudar.

Exemplo: se a função faz `dispatch(setUser(payload))`, busque todos os componentes que fazem `useSelector(state => state.user)` — eles precisam ser revisados se o shape de `user` mudar.

---

## Passo 6 — Gerar relatório Markdown

Salve em `docs/flows/fluxo-<nomeDoSimbolo>.md` (cria a pasta se não existir). Se o projeto já tiver convenção de pasta para documentação interna, prefira ela.

A estrutura abaixo é um **superconjunto** — adapte conforme a categoria definida no Passo 2:

```markdown
# Mapeamento de Fluxo: `<nomeDoSimbolo>`

| Propriedade           | Detalhe                                           |
| --------------------- | ------------------------------------------------- |
| **Arquivo**           | `<caminho>`                                       |
| **Linha**             | `<linha>`                                         |
| **Tipo**              | `<Componente / Hook / Função / Método / Classe>`  |
| **Mudança planejada** | `<o que o usuário disse — ou "não especificada">` |

## 📌 Resumo da Funcionalidade

Parágrafo curto: o que esse código faz na prática, em uma frase de negócio (não em uma frase técnica).

## 📐 Contrato

### Entradas

- `param` (`tipo`): origem e formato esperado.

### Saída

- Tipo: `<tipo>`. Forma: `<descrição do shape>`.

### Pré-condições / Pós-condições

- Condições assumidas antes da chamada e garantias após (se houver).

## 🗺️ Diagrama

(Apenas para categoria Média ou Grande. Escolha o tipo:)

- `flowchart TD` — para fluxo de dados e transformações.
- `sequenceDiagram` — para interações async (API, eventos).
- `graph TD` — para hierarquia de componentes.

## ⬇️ Downstream — O que este código usa

1. **`<nomeDoSimbolo>()`** (`<arquivo>:<linha>`)
   - 1.1 Chama `funçãoA()` (`<arquivo>`) — o que ela faz em uma linha.
     - 1.1.1 Chama `funçãoB()` (`<arquivo>`) — ...
   - 1.2 Hook `useX()` — lê `<estado>`.
   - 1.3 Módulo externo: `axios` — POST para `<endpoint>`.

### Efeitos colaterais

- **Network**: `<método> <endpoint>` (payload: `<descrição>`).
- **Estado**: `dispatch(<action>)` altera `<slice>`.
- **Browser**: grava em `localStorage['<chave>']`.
- **Navegação**: `router.push('<rota>')`.

## ⬆️ Upstream — Quem usa este código

### Call sites

| Arquivo     | Linha     | Como é usado                                                            |
| ----------- | --------- | ----------------------------------------------------------------------- |
| `<caminho>` | `<linha>` | `<descrição curta — ex: "passa formData direto, espera Promise<User>">` |

### Re-exports

- Exportado também via `<barrel.ts>` — buscar referências por esse caminho também.

### Testes

- ✅ Coberto em `<arquivo.test.tsx>` — cenários: `<lista curta>`.
- ⚠️ Sem cobertura direta de teste.

### Mocks e fixtures

- Mockado em `<setup ou arquivo>` — atenção ao alterar assinatura.

## 🔄 Estado compartilhado

(Apenas se o código lê/escreve estado global.)

O código mexe em `<slice/contexto>`. Outros consumidores deste estado:

- `<arquivo>` — `<como consome>`.

## ✅ Checklist de pontos de toque para a mudança

(Esta é a seção que o usuário vai usar de verdade ao codar.)

Considerando a mudança planejada (`<descrição>`), revise:

- [ ] **Se alterar a assinatura** (parâmetros): atualize os N call sites listados acima.
- [ ] **Se alterar o retorno** (tipo ou shape): revise os callers que consomem o retorno — em particular `<arquivo>` que faz `<uso específico>`.
- [ ] **Se remover/alterar o efeito X**: avise/ajuste os consumidores do estado compartilhado.
- [ ] **Se mudar o tipo importado**: rebuilde os tipos derivados em `<arquivo>`.
- [ ] **Testes a atualizar**: `<lista>` ou **criar teste novo** se não houver cobertura.
- [ ] **Mocks a atualizar**: `<lista>`.

## 📝 Observações

(Opcional. Inclua somente se houver algo realmente útil — acoplamentos suspeitos, dívida técnica relevante para a mudança, hooks com `deps` array problemático, etc. Se não houver nada notável, omita a seção inteira.)
```

---

## Regras de comportamento

- **Não invente arquivos**. Se uma busca upstream não retornou nada, escreva "Sem call sites encontrados no escopo analisado" — não preencha por preencher.
- **Adapte ao tamanho**. Função pequena não precisa de diagrama nem de seção de estado compartilhado se ela não toca estado. Não force as seções todas.
- **Linguagem PT-BR**, mas mantenha termos técnicos em inglês quando forem mais claros (`call site`, `hook`, `dispatch`, `state`).
- **Foco no que muda**. O relatório é uma ferramenta de pré-mudança, não uma documentação eterna. Detalhes irrelevantes para a mudança planejada devem ser cortados.
- **Confirme antes de aprofundar muito**. Em código grande, depois do mapeamento inicial, pergunte ao usuário se quer aprofundar algum ramo específico antes de gastar contexto rastreando 4 níveis em todas as branches.
- **Bibliotecas externas**: registre como "módulo externo: `nome`" e siga em frente — não tente abrir node_modules.
- **Se o ponto de entrada não for encontrado**: pare e peça confirmação do nome/arquivo. Não tente adivinhar.
