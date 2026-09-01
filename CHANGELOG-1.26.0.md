# Changelog — v1.26.0

> Data: 2026-08-26

---

## Novas funcionalidades

- **Cadastro de clientes — Seleção de loja**
  Adicionado campo de seleção de loja no fluxo de inclusão de cliente. O sistema pré-preenche automaticamente com a loja do usuário logado e restringe as opções ao escopo de permissão do usuário, impedindo a escolha de lojas fora do seu acesso.

- **Motivos de cancelamento — Remoção do grupo OPERACIONAIS**
  O grupo de motivos `OPERACIONAIS` foi removido das opções exibidas no cancelamento de oportunidade, restringindo os motivos disponíveis apenas aos grupos pertinentes ao fluxo.

---

## Correções

- **Cadastro de clientes — Loja emitente inválida ou vazia**
  Corrigido carregamento incorreto das lojas disponíveis e situação em que a loja emitente aparecia vazia. O sistema agora garante que a loja selecionada é sempre válida antes de prosseguir com o cadastro.

- **Comprometimento de estoque — Bloqueio indevido do botão em pedido clonado cancelado**
  Ajustadas as regras de bloqueio do botão de comprometimento de estoque que impediam a ação em pedidos clonados a partir de pedidos cancelados, mesmo quando o comprometimento deveria estar habilitado.

- **Chat — Foco perdido após ações do menu**
  O foco do cursor não retornava ao campo de mensagem após o usuário interagir com opções do menu de contexto do chat. Corrigido para que o foco seja devolvido automaticamente ao campo após essas ações.

---

## Chores

- `chore(1.26.0)`: atualiza versão do app para 1.26.0
- `chore`: atualiza versão do Vigga DS
