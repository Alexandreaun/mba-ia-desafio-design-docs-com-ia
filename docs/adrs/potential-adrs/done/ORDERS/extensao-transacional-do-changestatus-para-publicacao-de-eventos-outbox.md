# ADR em Potencial: Extensão Transacional de `OrderService.changeStatus` para Publicação Atômica de Eventos (Outbox)

**Módulo**: ORDERS
**Categoria**: Arquitetura (integração/consistência transacional)
**Prioridade**: Obrigatório Documentar (Pontuação: 135/150)
**Data de Identificação**: 2026-08-31

---

## O Que Foi Identificado

`OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) é hoje o único ponto do sistema que muda o status de um pedido, e faz isso dentro de um único `prisma.$transaction`: valida a transição via `canTransition`, debita/repõe estoque (`debitStock`/`replenishStock`), atualiza `Order.status` e insere uma linha em `OrderStatusHistory`. Esse método não tem nenhum efeito colateral fora da transação — é uma unidade atômica fechada, e é exatamente esse fechamento que a equipe decidiu preservar e estender na reunião de refinamento.

Na discussão sobre como a plataforma vai notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) de mudanças de status, a equipe rejeitou explicitamente o envio síncrono de webhook dentro do `OrderService` (`[09:03]-[09:05]` Larissa/Bruno: "a transação de mudança de status hoje já é pesada... se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status para outros pedidos"; "se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá"). Em vez disso, decidiu-se pelo padrão outbox (`[09:06]-[09:08]` Diego/Larissa) — e a peça central dessa decisão, do ponto de vista do módulo ORDERS, é que **a inserção do evento na tabela `webhook_outbox` deve ocorrer dentro da mesma transação SQL que já existe em `changeStatus`**, não depois, não em background, não em outro processo: "Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair" (`[09:40]` Bruno); "Essencial. Se ficar fora da transação, perde a garantia toda" (`[09:41]` Diego).

A equipe também definiu a forma concreta de integração: uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `Prisma.TransactionClient` (`tx`) já aberto pelo `changeStatus`, em vez de injetar um `WebhookRepository` inteiro no `OrderService` (`[09:41]` Bruno/Diego: "Vou propor uma função... que aceita o tx client da transação atual"; "Boa, função pura recebendo o tx. Não precisa injetar repository inteiro."). Isso é um detalhe de implementação da mesma decisão maior (consolidado aqui, não como ADR separado, conforme Sinal de Alerta 5), mas é relevante porque define o contrato de acoplamento entre os módulos ORDERS e WEBHOOKS: `OrderService` passa a depender de uma função externa que aceita seu `tx`, e não do inverso.

Não há histórico de Git para enriquecer esta decisão com contexto temporal: o repositório tem um único commit de inicialização (`7ef4317 init repository`, 2026-06-24) — `order.service.ts` não tem evolução registrada. O enriquecimento via Git é, portanto, não informativo, conforme já sinalizado em `mapping.md`.

## Por Que Isso Pode Merecer uma ADR

- **Impacto**: Define o limite de atomicidade/consistência do fluxo mais crítico do domínio (mudança de status de pedido). Qualquer novo tipo de efeito colateral de `changeStatus` no futuro (não só webhooks) terá que decidir se entra dentro ou fora dessa transação, usando este precedente.
- **Trade-offs**: Ganha-se garantia forte de consistência (evento nunca é perdido nem "fantasma") ao custo de acoplar a duração da transação de estoque/status à escrita na outbox, e de acoplar `OrderService` a uma função externa do módulo WEBHOOKS (ainda que via injeção de `tx`, não de repository).
- **Complexidade**: Baixa complexidade de código, mas alta relevância de design — é o tipo de decisão "óbvia depois de explicada, não óbvia antes".
- **Conhecimento da Equipe**: Qualquer engenheiro que tocar `OrderService.changeStatus` no futuro (ou revisar PRs que o modifiquem) precisa entender por que a chamada a `publishWebhookEvent` (quando implementada) tem que ficar dentro do `tx` e nunca ser movida para depois do `await this.prisma.$transaction(...)`.
- **Implicações Futuras**: Estabelece o padrão para qualquer evento de domínio futuro (não só webhooks) que a plataforma queira publicar de forma confiável a partir de mudanças de estado de pedido — vira precedente arquitetural reutilizável.
- **Contexto Temporal**: Não há evolução de Git a reportar (commit único de inicialização); a decisão é inteiramente proveniente da reunião de refinamento, ainda não implementada no código.

## Evidências Encontradas na Base de Código

### Arquivos Principais
- [`src/modules/orders/order.service.ts`](../../../../../src/modules/orders/order.service.ts) - Linhas 126-179 (`changeStatus`) - demonstra a transação atômica existente que a decisão da reunião propõe estender.
- [`src/modules/orders/order.status.ts`](../../../../../src/modules/orders/order.status.ts) - Linhas 1-37 - máquina de estados consultada por `changeStatus` (`canTransition`, `shouldDebitStock`, `shouldReplenishStock`); relevante porque cada transição válida vira um evento candidato a publicação.
- [`prisma/schema.prisma`](../../../../../prisma/schema.prisma) - Linhas 74-131 (`Order`, `OrderItem`, `OrderStatusHistory`) - schema atual não tem `webhook_outbox`; a tabela ainda precisa ser criada e sua inserção ocorrer no mesmo `$transaction`.

### Evidência no Código
```typescript
// src/modules/orders/order.service.ts:131-167 (trecho relevante)
return this.prisma.$transaction(async (tx) => {
  const order = await tx.order.findUnique({ where: { id }, include: { items: true } });
  if (!order) throw new NotFoundError('Order');

  const from = order.status;
  const to = input.toStatus;
  // ... validação de transição ...

  if (shouldDebitStock(from, to)) {
    await this.debitStock(tx, order.items);
  }
  if (shouldReplenishStock(from, to)) {
    await this.replenishStock(tx, order.items);
  }

  await tx.order.update({ where: { id }, data: { status: to } });
  await tx.orderStatusHistory.create({
    data: { orderId: id, fromStatus: from, toStatus: to, changedById: userId, reason: input.reason ?? null },
  });
  // Ponto de extensão decidido em reunião: chamar publishWebhookEvent(tx, order, from, to) AQUI,
  // ainda dentro do mesmo tx, antes do fim da transação.
  // ...
});
```

### Análise de Impacto
- Introduzido: não aplicável — `changeStatus` já existe hoje (init único, 2026-06-24); a decisão de webhooks é sobre estendê-lo, ainda não implementada.
- Modificado: histórico de Git não disponível/não informativo (commit único `7ef4317 init repository`; `git log --follow` e `git log -10` sobre `order.service.ts` retornam apenas essa entrada).
- Afeta: 1 arquivo diretamente (`order.service.ts`), mas define o contrato de integração para todo o módulo WEBHOOKS ainda a ser criado (`src/modules/webhooks/*`, tabela `webhook_outbox`).
- Temas da reunião: "atomicidade", "rollback", "consistência", "sem inconsistência possível" (`[09:06]` Diego).

### Alternativas Consideradas (explícitas na reunião)
- **Envio síncrono dentro do `OrderService`** — rejeitada (`[09:03]-[09:05]`): risco de travar a transação com HTTP call lento e impossibilidade de rollback caso o cliente esteja offline.
- **Fila externa (Redis Streams)** com publicação fora da transação MySQL — rejeitada (`[09:07]` Diego/Larissa): "a gente é um time pequeno, subir Redis Cluster pra isso é overengineering"; também reintroduziria o problema de dupla escrita (dual-write) entre MySQL e a fila, que o outbox no mesmo banco evita.
- **Trigger de banco de dados** para notificar o worker reativamente — rejeitada (`[09:09]` Diego): MySQL não tem `LISTEN/NOTIFY` como Postgres.
- **Injetar um `WebhookRepository` completo no `OrderService`** — implicitamente descartada em favor de uma função pura recebendo o `tx` (`[09:41]` Diego: "não precisa injetar repository inteiro"), preservando o baixo acoplamento do `OrderService` a detalhes de persistência de outro domínio.

## Questões a Abordar na ADR (se criada)

- Qual é exatamente o contrato de `publishWebhookEvent(tx, order, fromStatus, toStatus)` (assinatura, tipo de retorno, tratamento de erro)?
- O que acontece se a inserção na outbox falhar por outro motivo que não uma violação de negócio (ex.: erro de schema) — o `changeStatus` inteiro deve falhar (rollback total), incluindo a mudança de status e o débito de estoque?
- Esse mesmo padrão de "efeito colateral publicado dentro da transação via `tx`" deve ser formalizado como convenção para futuros efeitos colaterais de domínio, além de webhooks?
- Como testes de integração de `OrderService.changeStatus` (hoje sem asserções de outbox) precisarão ser estendidos para verificar a presença do evento na mesma transação?

## ADRs Potenciais Relacionadas

- Decisão mais ampla do padrão Outbox no MySQL (worker de polling, retry/backoff, DLQ) — pertence ao módulo **WEBHOOKS** (ainda não implementado; sem âncora em código de ORDERS além deste ponto de integração). Não foi criada uma ADR potencial separada para ORDERS cobrindo essas decisões, pois elas não têm evidência de código no módulo ORDERS — apenas esta transação é o ponto real de acoplamento. Recomenda-se que a análise da Fase 2 do módulo WEBHOOKS trate outbox/worker/retry/DLQ/HMAC/at-least-once como suas próprias ADRs (`must-document/WEBHOOKS/`).

## Observações Adicionais

- **Decisões da reunião com âncora indireta em ORDERS, mas sem código a documentar (não viraram ADR de ORDERS)**: worker separado com polling de 2s (`[09:09]-[09:11]`), retry com backoff 1m/5m/30m/2h/12h e DLQ em tabela própria (`[09:15]-[09:18]`), HMAC-SHA256 com secret por endpoint e rotação (`[09:20]-[09:22]`), at-least-once com `X-Event-Id` (`[09:24]-[09:26]`). Todas são **decisões confirmadas** na transcrição, mas pertencem estruturalmente ao módulo WEBHOOKS — não têm nenhum arquivo, classe ou tabela em `src/modules/orders/*` ou `prisma/schema.prisma` (seção `Order*`) que as implemente ou que dependa diretamente delas, exceto através do único ponto de acoplamento já documentado nesta ADR potencial (a chamada dentro de `changeStatus`). Sinalizado aqui como observação de acoplamento, não como decisão a ser forçada no escopo de ORDERS.
- **Incerteza sinalizada**: a assinatura exata de `publishWebhookEvent` e o comportamento em caso de filtro "nenhum webhook do customer quer aquele status" (`[09:33]-[09:34]`, decisão de módulo WEBHOOKS: filtrar na inserção, não inserir se não houver interessados) implica que a função precisa fazer uma consulta de configuração de webhook *dentro* do mesmo `tx` antes de decidir inserir ou não — esse subfluxo não foi detalhado com profundidade suficiente na reunião (não há confirmação explícita de como a função consulta a configuração do cliente); deve ser tratado como questão em aberto na ADR formal, não como decisão já fechada.
- Estado atual do código: nenhuma linha de `order.service.ts` faz qualquer referência a outbox/webhook hoje — toda a evidência de código nesta ADR potencial é sobre a estrutura *existente* que será estendida, não sobre uma implementação já presente.
