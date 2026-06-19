---
name: impact-analysis
description: Analisa o impacto de uma alteração antes de implementá-la. Identifica arquivos afetados, mapeia o fluxo da funcionalidade, aponta pontos críticos e gera um checklist de testes. Use esta skill sempre que o usuário quiser entender o impacto de uma mudança, correção de bug ou nova feature antes de codar — mesmo que diga "o que pode quebrar?", "onde fica isso?", "que arquivos são afetados?", "me ajuda a planejar essa alteração" ou simplesmente descreva uma mudança que quer fazer.
---

# Impact Analysis

## Objetivo

Antes de implementar qualquer alteração, entender o escopo completo do impacto: quais arquivos estão envolvidos, como a funcionalidade funciona, o que pode quebrar e o que precisa ser testado. A resposta é gerada em **Português do Brasil** e salva como arquivo `.md`.

---

## Passo 1 — Fazer as duas perguntas obrigatórias

Sempre comece perguntando **as duas perguntas abaixo juntas**, numa única mensagem:

> **1. O que vamos fazer?**
> Descreva a alteração, correção de bug ou nova funcionalidade.
>
> **2. Onde vamos fazer?**
> Informe o caminho do arquivo, pasta ou módulo. Se não souber, tudo bem — te ajudo a localizar.

Aguarde a resposta antes de prosseguir.

---

## Passo 2 — Localizar os arquivos (se o usuário não souber)

Se o usuário **não souber** onde a alteração deve ser feita:

**Não inicie a busca ainda.** Primeiro solicite palavras-chave:

> "Para localizar a feature sem analisar o projeto inteiro, me informe algumas palavras-chave. Pode ser: nome da tela, texto exibido na interface, nome de botão, nome de componente, nome de função, nome de endpoint ou nome de variável relacionada."

Somente após receber as palavras-chave, execute a busca seguindo **esta ordem obrigatória**:

1. `src/views` — procurar nos arquivos de view
2. `src/components` — procurar nos componentes

Em cada diretório, analise:
- nomes de arquivos
- nomes de componentes
- textos renderizados (strings, labels, títulos)
- imports e exports
- hooks utilizados
- funções e variáveis relevantes
- referências entre arquivos

Ao final, **sugira os arquivos mais prováveis** e confirme com o usuário antes de prosseguir para a análise completa.

---

## Passo 3 — Executar as análises

Com os arquivos confirmados, execute as quatro análises abaixo:

### 3.1 Localizar a Feature

Identifique todos os arquivos relacionados à funcionalidade:
- Views envolvidas
- Componentes utilizados
- Hooks customizados
- Serviços e chamadas de API
- Funções utilitárias
- Estados globais (ex: Redux, Zustand, Context)
- Rotas relacionadas

### 3.2 Mapear o Fluxo da Funcionalidade

Explique como a funcionalidade funciona no sistema:
- Origem dos dados (API, estado local, contexto global)
- Sequência de componentes envolvidos
- Fluxo entre arquivos (quem chama quem)
- Dependências relevantes (libs externas, configs)

### 3.3 Identificar Pontos Críticos

Destaque locais que **podem quebrar** se a alteração for feita incorretamente:
- Funções reutilizadas por outros módulos
- Contratos de API (tipos de request/response)
- Tipos e interfaces TypeScript compartilhados
- Lógica de validação
- Estados compartilhados
- Dependências indiretas que podem ser afetadas

### 3.4 Listar Impactos no Sistema

Identifique o que pode ser afetado **direta ou indiretamente**:
- Outras telas ou fluxos que usam os mesmos componentes/hooks
- Outros módulos que dependem dos arquivos alterados
- Efeitos colaterais em estados globais
- Mudanças que afetam a experiência em outras partes do sistema

---

## Passo 4 — Gerar o relatório

Salve a análise como arquivo `.md` na raiz do projeto com o nome `impact-analysis.md`.

Use **exatamente** esta estrutura:

```markdown
# Análise de Alteração

## Tarefa
[Descrição da alteração solicitada pelo usuário]

## Local da Alteração
[Arquivos ou módulos identificados como ponto central da mudança]

## Arquivos Envolvidos
[Lista de todos os arquivos relacionados à feature, com caminho relativo]

## Fluxo da Funcionalidade
[Explicação de como a funcionalidade funciona no sistema — origem dos dados, componentes, sequência de chamadas]

## Pontos Críticos
[Locais onde alterações mal feitas podem causar problemas — com explicação do risco de cada um]

## Impactos no Sistema
[Dependências e partes do sistema que podem ser afetadas direta ou indiretamente]

## Checklist de Testes
[Lista do que deve ser validado após a alteração]

### Fluxos principais
- [ ] ...

### Cenários alternativos
- [ ] ...

### Possíveis regressões
- [ ] ...

### Telas afetadas
- [ ] ...

### Endpoints relacionados
- [ ] ...

### Estados de erro
- [ ] ...

### Casos limite
- [ ] ...
```

---

## Diretrizes gerais

- **Seja objetivo e técnico.** O foco é evitar regressões, não descrever o óbvio.
- **Priorize pontos críticos.** Se algo tem alto risco de quebrar, diga claramente.
- **Use caminhos relativos** nos nomes de arquivos para facilitar a navegação.
- **Não invente** arquivos ou comportamentos — se não encontrou, diga que não localizou e sugira onde o usuário pode confirmar.
- **Confirme com o usuário** antes de iniciar a análise completa quando os arquivos foram localizados por busca de palavras-chave.
