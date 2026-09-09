# ADR em Potencial: Padrão Outbox no MySQL para Entrega de Eventos de Webhook

**Módulo**: WEBHOOKS
**Categoria**: Arquitetura (Infraestrutura / Consistência de Dados)
**Prioridade**: Obrigatório Documentar (Pontuação: 145/150)
**Data de Identificação**: 2026-08-31

---

## O Que Foi Identificado

A equipe decidiu que a notificação de mudança de status de pedido para clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) não será disparada de forma síncrona dentro de `OrderService.changeStatus`, nem via infraestrutura de mensageria externa (Redis Streams). Em vez disso, será adotado o **padrão Outbox transacional sobre o MySQL já existente**: dentro da mesma transação SQL que atualiza `Order.status` e insere em `OrderStatusHistory`, uma linha é inserida em uma nova tabela `webhook_outbox` contendo o evento a ser entregue. Um worker separado (tratado em outra ADR potencial) lê essa tabela de forma assíncrona e realiza as chamadas HTTP.

Esta é a decisão fundacional de toda a feature: garante que a existência do evento de notificação é atomicamente consistente com a mudança de status que o originou — se a transação principal falhar, o evento nunca existiu; se ela for commitada, o evento está garantidamente registrado. Como o módulo `webhooks` ainda não existe no código (`git log` mostra um único commit de inicialização do repositório, sem histórico incremental), toda a evidência desta decisão vem exclusivamente de `TRANSCRICAO.md` — o que é esperado, não uma fraqueza da análise (conforme já registrado em `docs/adrs/mapping.md`, seção "Diretrizes para a Fase 2"). O ponto de integração real e já existente no código, `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), é onde a inserção na outbox precisará ser encaixada dentro do `prisma.$transaction` já existente.

## Por Que Isso Pode Merecer uma ADR

- **Impacto**: Define o mecanismo de consistência entre o núcleo transacional mais crítico do sistema (`OrderService.changeStatus`) e a primeira integração externa (outbound) que a plataforma passará a ter. Qualquer decisão futura de escalar/substituir esse mecanismo (ex.: migrar para uma fila dedicada) exige revisitar este ADR.
- **Trade-offs**: Explicitamente comparado e preferido em relação a duas alternativas debatidas e descartadas — disparo síncrono no service e Redis Streams/fila externa — por razões de simplicidade operacional (equipe pequena) e por evitar acoplamento de disponibilidade externa à transação de negócio.
- **Complexidade**: Introduz um novo padrão arquitetural (Outbox) inédito no projeto, com implicações de schema (`webhook_outbox`), índices, política de arquivamento e acoplamento com o worker de entrega.
- **Conhecimento da Equipe**: Qualquer engenheiro que futuramente altere `OrderService.changeStatus` precisa entender que a inserção na outbox faz parte do contrato transacional daquele método — remover ou mover essa inserção para fora da transação quebra a garantia de consistência.
- **Implicações Futuras**: A escolha por MySQL (sem `LISTEN/NOTIFY`) em vez de uma fila dedicada implica polling (ver ADR potencial do worker) como consequência direta, e limita a garantia de ordering a um cenário single-worker — uma limitação documentada e aceita conscientemente.
- **Contexto Temporal**: Decisão tomada integralmente nesta reunião de ~55 minutos (09/2026); não há histórico de código anterior porque a implementação ainda não começou.

## Evidências Encontradas na Base de Código

### Arquivos Principais
- [`src/modules/orders/order.service.ts`](../../../../../src/modules/orders/order.service.ts) — linhas 126-179, método `changeStatus`. Ponto de integração real e único candidato natural para a inserção na outbox: já executa dentro de `this.prisma.$transaction(async (tx) => {...})`, já teria acesso ao `tx` client necessário para inserir na `webhook_outbox` de forma atômica.
- `prisma/schema.prisma` — não contém (ainda) nenhum modelo `webhook_outbox`, `WebhookEvent` ou equivalente. Confirma que a decisão é 100% prospectiva.

### Evidência no Código
```typescript
// src/modules/orders/order.service.ts:139-179 (changeStatus, transação existente)
return this.prisma.$transaction(async (tx) => {
  const order = await tx.order.findUnique({ where: { id }, include: { items: true } });
  // ... validação de transição, débito/reposição de estoque ...
  await tx.order.update({ where: { id }, data: { status: to } });
  await tx.orderStatusHistory.create({ data: { orderId: id, fromStatus: from, toStatus: to, ... } });
  // <- ponto de inserção proposto para publishWebhookEvent(tx, order, from, to)
  const refreshed = await tx.order.findUnique({ ... });
  return refreshed!;
});
```

### Evidência na Transcrição
- `[09:03]-[09:05]` Larissa/Bruno: descarte da alternativa síncrona ("Síncrono não rola... se acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status").
- `[09:06]` Diego: proposta e explicação do padrão Outbox ("dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também insere uma linha numa tabela tipo webhook_outbox... Não tem inconsistência possível").
- `[09:07]` Diego/Larissa: descarte explícito de Redis Streams por overengineering para o tamanho do time.
- `[09:08]` Larissa: fechamento formal — "Tá decidido então: outbox em MySQL."
- `[09:40]-[09:41]` Bruno/Diego: reforço da criticidade da atomicidade — "Se ficar fora da transação, perde a garantia toda."
- `[09:41]` Bruno/Diego: proposta técnica concreta de uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` client da transação atual, evitando injetar um repository inteiro no `OrderService`.
- `[09:51]-[09:52]` Larissa/Bruno/Diego: decisão complementar de que o payload do evento é renderizado (snapshot) no momento da inserção na outbox, não recalculado no envio — detalhe de design que integra esta mesma decisão de Outbox.

### Análise de Impacto (Git)
- Histórico do Git não aplicável para o padrão em si: a tabela `webhook_outbox` e o código de integração ainda não existem no repositório.
- `git log --follow` sobre `src/modules/orders/order.service.ts` retorna um único commit (`init repository`), sem granularidade temporal — não há evolução incremental a analisar para o ponto de integração.

### Alternativas (explicitamente discutidas na transcrição)
- **Disparo síncrono dentro de `OrderService.changeStatus`** — rejeitada (`[09:03]-[09:05]`): risco de travar a transação de mudança de status por cliente lento; impossibilidade de rollback caso o cliente esteja offline.
- **Redis Streams / fila externa dedicada** — rejeitada (`[09:07]`): considerada overengineering para o tamanho da equipe; exigiria subir infraestrutura nova.
- **Trigger de banco para notificar o worker reativamente** — rejeitada (`[09:09]`): MySQL não possui `LISTEN/NOTIFY` como Postgres; um trigger só executa SQL, não notifica processos externos.

## Questões a abordar na ADR (se criada)

- Qual problema estava sendo resolvido (necessidade de notificação assíncrona sem comprometer a transação de negócio)?
- Por que Outbox no MySQL existente foi escolhido em vez de fila externa ou disparo síncrono?
- Qual o desenho de schema da tabela `webhook_outbox` (campos, índices em status/created_at mencionados em `[09:08]`, política de arquivamento após 30 dias)?
- Quais são as consequências de longo prazo (acoplamento entre `OrderService` e a função `publishWebhookEvent`, limite de ordering documentado na ADR do worker)?

## ADRs Potenciais Relacionadas
- Worker de Entrega em Processo Separado com Polling (consome a `webhook_outbox`)
- Política de Retry com Backoff Exponencial e Dead Letter Queue (trata falhas de entrega dos eventos gerados pela outbox)
- Garantia de Entrega At-Least-Once com Idempotência via X-Event-Id (o `event_id` é gerado no momento da inserção na outbox)

## Observações Adicionais

- Detalhes consolidados nesta ADR (não devem virar ADRs separadas, por Sinal de Alerta 5 — granularidade excessiva): uso de UUID como PK da outbox (`[09:50]-[09:51]`, consistente com o restante do schema, decisão trivial de convenção já estabelecida no projeto), payload como snapshot renderizado na inserção (`[09:51]-[09:52]`), e filtro de eventos por status do webhook aplicado na inserção, não no envio (`[09:33]-[09:34]`).
- A limitação de ordering (garantia apenas por `order_id`, apenas em cenário single-worker, `[09:12]-[09:13]`) é uma consequência direta desta decisão combinada com a do worker; deve ser documentada na seção de Consequências da ADR formal, não como decisão separada.
