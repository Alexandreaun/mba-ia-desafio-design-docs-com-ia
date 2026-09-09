# ADR-003: Publicação Atômica de Eventos de Webhook via Outbox dentro de `OrderService.changeStatus`

**Status:** Proposta
**Date:** 2026-08-31
**Related ADRs:** ADR-006

---

## Status

Proposta. A decisão foi debatida e fechada por consenso na reunião de refinamento técnico (`[09:03]`-`[09:08]`, `[09:40]`-`[09:41]`), mas ainda não há implementação no código: nenhuma linha de `src/modules/orders/order.service.ts` referencia outbox ou webhook hoje, a tabela `webhook_outbox` não existe em `prisma/schema.prisma`, e a função `publishWebhookEvent` ainda não foi criada.

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) formalizaram um pedido para serem notificados em tempo real (abaixo de 10 segundos é aceitável) quando o status de seus pedidos muda, eliminando o polling atual via `GET /orders` que hoje torna a integração lenta e cara para eles — a Atlas chegou a sinalizar risco de migração para um concorrente caso isso não seja entregue até o fim do trimestre (`[09:00]` Marcos). Do lado técnico, `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) já é hoje o único ponto que altera o status de um pedido, fazendo isso dentro de um único `prisma.$transaction` que valida a transição, debita/repõe estoque e insere uma linha em `OrderStatusHistory`.

A equipe descartou disparar o webhook de forma síncrona dentro dessa transação: uma chamada HTTP a um cliente lento ou indisponível travaria a mudança de status de outros pedidos, e não haveria como reverter a notificação em caso de rollback (`[09:03]-[09:05]` Bruno/Larissa). A solução escolhida foi o padrão outbox, com a inserção do evento ocorrendo na mesma transação SQL que já existe em `changeStatus`, de forma que o evento nunca exista sem o status correspondente ter sido persistido, nem vice-versa (`[09:06]-[09:08]` Diego).

Não há histórico de Git para enriquecer esta decisão com contexto temporal — o repositório tem um único commit de inicialização (`7ef4317`, 2026-06-24) e nenhuma linha de `order.service.ts` referencia outbox ou webhook hoje; toda a decisão vem da reunião de refinamento e ainda não foi implementada no código.

## Decisão

Estender `OrderService.changeStatus` para, dentro da mesma transação (`tx`) que já atualiza `Order.status`, debita/repõe estoque e grava `OrderStatusHistory`, também inserir o evento de mudança de status na tabela `webhook_outbox` (ainda a ser criada). A integração se dá por meio de uma função pura, `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `Prisma.TransactionClient` já aberto por `changeStatus`, em vez de o `OrderService` depender de um `WebhookRepository` completo do módulo WEBHOOKS (`[09:41]` Bruno/Diego).

Essa escolha unifica a necessidade de negócio (não perder notificações que os clientes B2B passarão a depender operacionalmente) com a limitação técnica existente (a transação de `changeStatus` já é a única fronteira de atomicidade do domínio): se a inserção na outbox falhar por qualquer motivo, a transação inteira sofre rollback — "não pode ter caso de status mudar e evento não sair" (`[09:40]` Bruno; `[09:41]` Diego). O contrato baseado em função pura recebendo `tx`, e não em injeção de repositório, mantém `OrderService` de baixo acoplamento a detalhes de persistência de outro domínio.

## Alternativas Consideradas

### Envio síncrono de webhook dentro de `OrderService.changeStatus`

- **Prós:** Implementação mais simples, sem necessidade de tabela outbox nem worker separado.
- **Prós:** Entrega imediata ao cliente, sem a latência mínima introduzida por um ciclo de polling.
- **Prós:** Menor superfície de código nova no curto prazo (nenhum módulo WEBHOOKS/worker adicional).
- **Contras:** Uma chamada HTTP lenta ou cliente indisponível trava a transação e bloqueia mudanças de status de outros pedidos (`[09:04]` Bruno).
- **Contras:** Sem possibilidade de rollback coerente caso o cliente esteja fora do ar no meio da chamada (`[09:04]` Bruno).
- **Contras:** Acopla a confiabilidade do domínio de pedidos à disponibilidade de sistemas externos de terceiros.

### Fila externa (Redis Streams) publicada fora da transação MySQL

- **Prós:** Desacopla completamente a infraestrutura de mensageria do banco transacional principal.
- **Prós:** Escalabilidade horizontal nativa para múltiplos consumidores.
- **Prós:** Modelo de fila é familiar para cenários de alto volume fora do domínio de pedidos.
- **Contras:** Reintroduz o problema clássico de dupla escrita (dual-write) entre MySQL e a fila, que o outbox no mesmo banco evita (`[09:07]` Diego).
- **Contras:** Exige subir e operar infraestrutura adicional (ex.: Redis Cluster), considerado overengineering para o tamanho da equipe (`[09:07]` Diego).
- **Contras:** Nenhuma garantia atômica nativa entre o commit da transação de pedidos e a publicação na fila.

### Injeção de `WebhookRepository` completo em `OrderService`

- **Prós:** Interface mais rica, com acesso direto a todas as operações de persistência do módulo WEBHOOKS.
- **Prós:** Padrão de repositório já é convenção no restante da base de código.
- **Prós:** Facilitaria eventuais consultas adicionais ao configurar novos tipos de evento.
- **Contras:** Acopla `OrderService` a detalhes de persistência de um domínio que não é o seu (`[09:41]` Diego).
- **Contras:** Amplia a superfície de dependência do `OrderService` para além do estritamente necessário (apenas registrar um evento na mesma transação).
- **Contras:** Dificulta testar `OrderService` isoladamente, exigindo mock de um repositório inteiro de outro módulo.

## Consequências

A decisão garante consistência forte entre a mudança de status e o registro do evento: nunca existirá um pedido cujo status mudou sem o evento correspondente na outbox, nem um evento "fantasma" sem mudança de status real por trás — atendendo diretamente ao requisito de negócio de notificação confiável para os clientes B2B. Estabelece também um precedente arquitetural: qualquer efeito colateral futuro de `changeStatus` (não só webhooks) terá que decidir explicitamente se entra ou não nessa fronteira transacional.

Em contrapartida, a transação de `changeStatus` — já descrita pela equipe como "pesada" (atualiza `Order`, `OrderStatusHistory` e estoque) — passa a incluir mais uma escrita, aumentando ligeiramente sua duração e a janela de contenção de locks no banco. O módulo ORDERS também passa a depender de uma função externa do módulo WEBHOOKS (ainda que via injeção de `tx`, não de repositório completo), criando um ponto de acoplamento entre módulos que precisa ser mantido estável à medida que o módulo WEBHOOKS evolui (worker de polling, retry e DLQ, tratados como decisões à parte, específicas desse módulo).

[NEEDS INPUT: Não há definição de como `publishWebhookEvent` deve consultar, dentro da mesma `tx`, a configuração de quais webhooks do cliente estão interessados em cada status antes de decidir inserir o evento — a transcrição sinaliza que a filtragem ocorre na inserção (`[09:33]-[09:34]` Bruno/Diego), mas não detalha a consulta em si nem seu impacto em duração de lock.]

[NEEDS INPUT: Não ficou definido na reunião nem há evidência no código se este padrão de "efeito colateral publicado dentro da mesma transação via `tx`" deve ser formalizado como convenção geral para futuros eventos de domínio além de webhooks, ou se é uma decisão específica apenas para este caso.]

## Referências

- `src/modules/orders/order.service.ts:126-179` — método `changeStatus`, transação atômica existente que esta decisão estende.
- `src/modules/orders/order.status.ts:1-37` — máquina de estados (`canTransition`, `shouldDebitStock`, `shouldReplenishStock`) consultada em cada transição, cada uma candidata a gerar um evento.
- `prisma/schema.prisma:74-131` — modelos `Order`, `OrderItem` e `OrderStatusHistory`; ainda não existe `webhook_outbox` no schema atual.
- `TRANSCRICAO.md [09:03]-[09:08]` — debate sobre rejeição do envio síncrono e decisão pelo padrão outbox (Bruno, Larissa, Diego).
- `TRANSCRICAO.md [09:40]-[09:41]` — decisão pela função pura `publishWebhookEvent(tx, ...)` em vez de injeção de repositório completo (Bruno, Diego).
