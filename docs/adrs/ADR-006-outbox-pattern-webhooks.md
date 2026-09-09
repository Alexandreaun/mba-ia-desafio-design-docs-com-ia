# ADR-006: Padrão Outbox no MySQL para Entrega de Eventos de Webhook

**Status:** Proposta
**Data:** 2026-08-31
**Related ADRs:** ADR-003

## Status

Proposta. A decisão foi debatida e fechada por unanimidade na reunião de refinamento técnico ("Tá decidido então: outbox em MySQL"), mas ainda não há implementação no código: não existe modelo `webhook_outbox` (ou equivalente) em `prisma/schema.prisma`, nem qualquer inserção correspondente no fluxo de mudança de status de pedidos. O status será promovido para Aceita quando a tabela e o ponto de integração forem implementados conforme aqui descrito.

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitaram notificação de mudança de status de pedidos em até 10 segundos, hoje resolvida via polling manual no `GET /orders`, considerado lento e caro para eles — com risco explícito de perda de contrato caso não seja entregue até o fim do trimestre. A equipe descartou disparo síncrono de HTTP dentro da transação de mudança de status de pedidos porque essa transação já é pesada (atualiza o pedido, registra seu histórico de status e movimenta o estoque dos produtos envolvidos) e um cliente externo lento ou indisponível travaria a mudança de status de outros pedidos, sem possibilidade de rollback coerente.

A solução técnica proposta foi o padrão Outbox transacional sobre o MySQL já em uso: dentro da mesma transação SQL que atualiza `Order` e `OrderStatusHistory`, é inserida uma linha em uma nova tabela `webhook_outbox` com o evento a ser entregue. Um worker assíncrono separado (tratado em ADR própria) lê essa tabela e realiza as chamadas HTTP. Como o módulo de webhooks ainda não existe no código, toda a evidência desta decisão é prospectiva e vem da transcrição da reunião — o que é esperado nesta fase de design, não uma lacuna de análise.

Do ponto de vista de negócio, o prazo é curto: o pedido dos três clientes B2B foi formalizado com risco explícito de churn até o fim do trimestre, o que pautou a preferência da equipe por uma solução que reaproveitasse a infraestrutura de banco já operada (MySQL), em vez de uma alternativa que exigisse aprendizado ou operação de um componente novo antes do prazo.

## Decisão

Adotar o padrão Outbox transacional sobre o MySQL existente, com a inserção do evento acontecendo dentro da mesma transação de mudança de status de pedidos já existente (`src/modules/orders/order.service.ts:131-178`). Isso garante atomicidade entre a mudança de status e o registro do evento a ser notificado: se a transação principal falhar, o evento nunca existe; se ela for commitada, o evento está garantidamente registrado, sem estado intermediário inconsistente possível.

A escolha por reutilizar o MySQL — em vez de subir uma fila dedicada — foi motivada pelo tamanho pequeno da equipe e pela ausência de qualquer infraestrutura de mensageria já operada pelo time, evitando acoplar a disponibilidade de um componente novo à transação de negócio mais crítica do sistema. A equipe também definiu que a inserção deve ocorrer por meio de uma função pura que recebe o client de transação já em uso, evitando injetar um repository completo de webhooks dentro do serviço de pedidos.

## Alternativas Consideradas

### Disparo síncrono de HTTP na transação de mudança de status

Chamar o endpoint do cliente B2B diretamente dentro da própria transação de mudança de status, sem componente intermediário.

- Prós: Menor complexidade inicial, sem necessidade de worker ou tabela adicional; entrega imediata ao cliente.
- Prós: Não requer novo componente de infraestrutura nem novo processo a operar.
- Contras: Cliente externo lento ou indisponível trava a transação de mudança de status de outros pedidos.
- Contras: Não há como fazer rollback coerente da mudança de status caso a chamada HTTP falhe, misturando falha de infraestrutura externa com a lógica de negócio.

### Fila externa dedicada (Redis Streams ou equivalente)

Publicar o evento em uma fila de mensageria dedicada, fora do MySQL, consumida por um worker independente.

- Prós: Desacopla completamente a entrega de eventos da transação de banco, com maior throughput potencial.
- Prós: Suporta múltiplos consumidores e escalonamento horizontal nativo.
- Contras: Exige subir e operar infraestrutura nova, considerada overengineering para o tamanho atual da equipe.
- Contras: Introduz um segundo ponto de falha (disponibilidade da fila) sem ganho imediato de valor para o volume atual de eventos.

### Trigger de banco para notificar o worker reativamente

Usar um trigger de banco de dados disparado na atualização de `Order` para acionar o worker de forma reativa, em vez de por polling.

- Prós: Reduziria a latência de detecção de novos eventos em relação ao polling.
- Contras: MySQL não possui um mecanismo nativo equivalente ao `LISTEN/NOTIFY` do PostgreSQL.
- Contras: Um trigger de banco só executa SQL adicional; não é capaz de notificar um processo externo, exigindo soluções alternativas (escrita em arquivo, chamada a endpoint) consideradas inadequadas pela equipe.

## Consequências

A consistência entre a mudança de status do pedido e a existência do evento de notificação passa a ser garantida transacionalmente, eliminando a classe de bug em que um pedido muda de status mas o cliente B2B nunca é notificado (ou é notificado sem a mudança ter realmente ocorrido). Em contrapartida, a inserção na outbox passa a fazer parte do contrato transacional da mudança de status de pedidos: qualquer alteração futura nesse fluxo precisa preservar essa inserção dentro da mesma transação, sob risco de romper silenciosamente a garantia de consistência — esse acoplamento é o trade-off central assumido pela equipe.

A escolha de MySQL sem mecanismo de notificação nativo implica que a leitura da outbox só pode ocorrer via polling por um worker separado (decisão tratada em ADR própria), o que introduz uma latência mínima inerente ao invés de entrega verdadeiramente orientada a eventos. Além disso, a garantia de ordering de entrega é limitada: só é assegurada por `order_id` e apenas enquanto houver um único worker em execução; caso o sistema evolua para múltiplos workers em paralelo no futuro, essa garantia deixa de existir e exigiria particionamento ou lock pessimista, o que a equipe documentou como limitação conhecida e aceita conscientemente para o escopo atual.

Por fim, o payload do evento é renderizado como snapshot no momento da inserção na outbox (e não recalculado no momento do envio), garantindo que o evento reflita fielmente o estado do pedido no instante da mudança de status, mesmo que o pedido seja alterado posteriormente — decisão complementar que reforça a consistência temporal do mecanismo de outbox.

## Referências

- `TRANSCRICAO.md` [09:03]-[09:08] — debate e fechamento da decisão de Outbox em MySQL, com descarte de disparo síncrono e de Redis Streams.
- `TRANSCRICAO.md` [09:40]-[09:41] — reforço da necessidade de atomicidade transacional e proposta de uma função pura para a inserção do evento, recebendo o client de transação atual.
- `TRANSCRICAO.md` [09:51]-[09:52] — decisão complementar de snapshot do payload no momento da inserção na outbox.
- `src/modules/orders/order.service.ts:131-178` — transação existente da mudança de status de pedidos, ponto de integração real da inserção na outbox.
- `prisma/schema.prisma` — ausência atual de modelo `webhook_outbox`, confirmando que esta é uma decisão prospectiva.
