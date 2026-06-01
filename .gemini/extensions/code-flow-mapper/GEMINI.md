# Code Flow Mapper

## Objetivo

Mapear a **árvore de execução bidirecional** de um ponto de entrada no código, para que o usuário entenda exatamente como a funcionalidade acontece antes de alterá-la. A pergunta que esta skill responde é: _"Como esse pedaço de código executa, e o que mais preciso revisar quando eu mexer nele?"_

O mapeamento é **bidirecional**:

- **Downstream** (pra baixo): o que este código chama, depende e afeta.
- **Upstream** (pra cima): quem chama este código, quais arquivos importam, qual contrato é esperado.

O output é um relatório Markdown em PT-BR cuja profundidade se adapta ao tamanho e à criticidade do ponto de entrada.

### Diferença para `impact-analysis`

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

1. **Chamadas de função relevantes** — entre nos arquivos correspondentes e siga até no máximo 4 níveis de profundidade.
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

Use ferramentas de busca disponíveis para encontrar:

1. **Call sites diretos** — todos os arquivos que importam e chamam o símbolo.
2. **Re-exports** — o símbolo é re-exportado de algum `index.ts`? Se sim, callers podem estar referenciando o caminho do barrel, não o original.
3. **Testes** — arquivos `.test.ts(x)` ou `.spec.ts(x)` que cobrem o símbolo.
4. **Mocks** — o símbolo aparece mockado em algum lugar? Se sim, mudar a assinatura quebra os mocks.
5. **Storybook ou fixtures** — para componentes, há stories ou dados de exemplo que usam o componente?

Para cada call site, registre **como** o símbolo é usado (qual argumento passa, o que faz com o retorno).

---

## Passo 5 — Mapear estado compartilhado

Se o ponto de entrada **lê ou escreve** em estado global (Redux store, Context, Zustand, query cache do React Query, etc.), liste **outros consumidores desse mesmo estado**.

Exemplo: se a função faz `dispatch(setUser(payload))`, busque todos os componentes que fazem `useSelector(state => state.user)`.

---

## Passo 6 — Gerar relatório Markdown

Salve em `docs/flows/fluxo-<nomeDoSimbolo>.md` (cria a pasta se não existir).

```markdown
# Mapeamento de Fluxo: `<nomeDoSimbolo>`

| Propriedade           | Detalhe                                           |
| --------------------- | ------------------------------------------------- |
| **Arquivo**           | `<caminho>`                                       |
| **Linha**             | `<linha>`                                         |
| **Tipo**              | `<Componente / Hook / Função / Método / Classe>`  |
| **Mudança planejada** | `<o que o usuário disse — ou "não especificada">` |

## 📌 Resumo da Funcionalidade

Parágrafo curto: o que esse código faz na prática, em uma frase de negócio.

## 📐 Contrato

### Entradas
- `param` (`tipo`): origem e formato esperado.

### Saída
- Tipo: `<tipo>`. Forma: `<descrição do shape>`.

### Pré-condições / Pós-condições
- Condições assumidas antes da chamada e garantias após (se houver).

## 🗺️ Diagrama

(Apenas para categoria Média ou Grande.)

## ⬇️ Downstream — O que este código usa

1. **`<nomeDoSimbolo>()`** (`<arquivo>:<linha>`)
   - 1.1 Chama `funçãoA()` (`<arquivo>`) — o que ela faz em uma linha.
   - 1.2 Hook `useX()` — lê `<estado>`.
   - 1.3 Módulo externo: `axios` — POST para `<endpoint>`.

### Efeitos colaterais
- **Network**: `<método> <endpoint>`.
- **Estado**: `dispatch(<action>)` altera `<slice>`.
- **Browser**: grava em `localStorage['<chave>']`.
- **Navegação**: `router.push('<rota>')`.

## ⬆️ Upstream — Quem usa este código

### Call sites

| Arquivo     | Linha     | Como é usado |
| ----------- | --------- | ------------ |
| `<caminho>` | `<linha>` | `<descrição>` |

### Testes
- ✅ Coberto em `<arquivo.test.tsx>`.
- ⚠️ Sem cobertura direta de teste.

### Mocks e fixtures
- Mockado em `<setup ou arquivo>` — atenção ao alterar assinatura.

## 🔄 Estado compartilhado

O código mexe em `<slice/contexto>`. Outros consumidores deste estado:
- `<arquivo>` — `<como consome>`.

## ✅ Checklist de pontos de toque para a mudança

- [ ] **Se alterar a assinatura**: atualize os N call sites listados acima.
- [ ] **Se alterar o retorno**: revise os callers que consomem o retorno.
- [ ] **Testes a atualizar**: `<lista>` ou **criar teste novo** se não houver cobertura.
- [ ] **Mocks a atualizar**: `<lista>`.
```

---

## Regras de comportamento

- **Não invente arquivos**. Se uma busca upstream não retornou nada, escreva "Sem call sites encontrados no escopo analisado".
- **Adapte ao tamanho**. Função pequena não precisa de diagrama nem de seção de estado compartilhado.
- **Linguagem PT-BR**, mas mantenha termos técnicos em inglês quando forem mais claros (`call site`, `hook`, `dispatch`, `state`).
- **Foco no que muda**. O relatório é uma ferramenta de pré-mudança, não uma documentação eterna.
- **Bibliotecas externas**: registre como "módulo externo: `nome`" e siga em frente.
- **Se o ponto de entrada não for encontrado**: pare e peça confirmação do nome/arquivo.
