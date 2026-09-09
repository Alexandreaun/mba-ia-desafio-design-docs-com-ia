# ADR-005: Garantia de Entrega At-Least-Once com Idempotência via X-Event-Id

**Status:** Aceita
**Date:** 31-08-2026

---

## Status

Aceita. A decisão foi debatida e fechada de forma consensual durante a reunião de refinamento
técnico ([09:26] Larissa: "Beleza. At-least-once com X-Event-Id pra dedup do lado do cliente.
Decisão."). A implementação do módulo de webhooks ainda não existe na base de código — o projeto
conta apenas com o commit inicial (`init repository`) — portanto esta ADR formaliza um contrato
de API já decidido, mas ainda pendente de desenvolvimento.

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) solicitaram notificação em
tempo real de mudanças de status de pedidos, hoje resolvida via polling manual em `GET /orders`,
considerado lento e caro para eles ([09:00]-[09:02] Marcos). Ao desenhar o contrato de entrega
desses eventos via webhook, a equipe identificou que, em um sistema distribuído com retries
(decorrente do próprio mecanismo de backoff definido para falhas de entrega), é impraticável
garantir que cada evento chegue exatamente uma vez ao cliente sem uma coordenação bilateral cara
entre plataforma e integrador.

A equipe optou por garantir a entrega **at-least-once**, aceitando que o cliente pode receber o
mesmo evento mais de uma vez, e transferiu a responsabilidade de deduplicação para o lado do
cliente através de um identificador único por evento. Esse identificador (`event_id`, um UUID) é
gerado no momento em que o evento é inserido na tabela de outbox e propagado em todas as
tentativas de reenvio do mesmo evento via header `X-Event-Id`. A escolha de UUID como formato do
identificador segue a convenção já estabelecida em todo o `prisma/schema.prisma`, onde todas as
entidades (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`) usam UUID
como chave primária ([09:50]-[09:51] Larissa/Diego), e é viabilizada sem nova dependência pela
biblioteca `uuid` já presente no `package.json`.

## Decisão

A plataforma garantirá entrega **at-least-once** para eventos de webhook, não exactly-once, e
delegará ao cliente a responsabilidade de deduplicação via o header `X-Event-Id`, cujo valor é
um UUID gerado uma única vez no momento da inserção do evento na `webhook_outbox` e reenviado
inalterado em todas as tentativas de retry daquele mesmo evento.

Essa escolha se justifica por dois fatores combinados: técnico e de mercado. Tecnicamente,
garantir exactly-once exigiria coordenação transacional entre dois sistemas independentes
(plataforma e cliente), o que é desproporcional ao ganho e conflita com a decisão paralela de
usar backoff exponencial com múltiplas tentativas de reenvio, que por natureza pode gerar
entregas duplicadas em cenários de timeout do lado do cliente (ver decisão de retry/DLQ, tratada
em ADR separada). Do ponto de vista de mercado, at-least-once com deduplicação client-side via
identificador único é o padrão adotado por provedores de referência como Stripe e GitHub ([09:25]
Diego), reduzindo a curva de aprendizado dos clientes B2B integradores que já conhecem esse
contrato.

## Alternativas Consideradas

### Garantia Exactly-Once

Modelo em que a plataforma garantiria que cada evento fosse entregue e processado exatamente uma
vez, sem possibilidade de duplicação do lado do cliente.

**Prós:**
- Elimina a necessidade de o cliente implementar lógica de deduplicação própria.
- Simplifica o modelo mental de integração ("cada evento chega uma única vez").

**Contras:**
- Exige coordenação bilateral complexa entre plataforma e cliente (ex.: transações distribuídas,
  acknowledgements idempotentes de dois lados), considerada desproporcional ao ganho ([09:25]
  Diego).
- Não é compatível de forma simples com o mecanismo de retry com backoff já decidido para
  tolerar indisponibilidade temporária do cliente, que por natureza reintroduz risco de
  duplicação.
- Não é o padrão adotado por provedores de referência de mercado consultados pela equipe
  (Stripe, GitHub), o que aumentaria o esforço de integração percebido pelos clientes B2B.

## Consequências

**Positivas**: a plataforma mantém uma arquitetura de entrega simples do lado do backend — não
há necessidade de rastrear confirmações de processamento do cliente nem de lógica de coordenação
distribuída — o que é compatível com o tamanho reduzido do time de engenharia e com a decisão de
reaproveitar a infraestrutura existente (MySQL, sem filas externas). O identificador único por
evento também sustenta a rastreabilidade do histórico de entregas exposto ao cliente
(`GET /webhooks/:id/deliveries`).

**Negativas**: a decisão desloca uma responsabilidade de engenharia real (deduplicação) para os
clientes B2B integradores, gerando uma objeção explícita durante a reunião ([09:25] Sofia: "Isso
joga responsabilidade pro cliente"). Isso implica um trade-off assumido conscientemente pela
equipe: menor complexidade e custo de infraestrutura no lado da plataforma em troca de maior
esforço de implementação exigido de cada cliente integrador, que precisa manter e testar sua
própria lógica de dedup baseada em `X-Event-Id`. Esse trade-off também gera um risco de contrato
de longo prazo: uma vez que clientes implementem dedup baseada nesse identificador, uma futura
migração para semântica exactly-once seria uma mudança potencialmente disruptiva para
integrações já em produção.

Como mitigação, a equipe decidiu que essa garantia e a necessidade de dedup do lado do cliente
serão documentadas de forma destacada no portal de desenvolvedor ([09:26] Marcos), reduzindo o
risco de integrações mal implementadas por falta de conhecimento do contrato.

[NEEDS INPUT: Não há evidência na transcrição ou no código sobre um SLA ou compromisso formal de
comunicação a clientes já integrados caso a garantia de entrega evolua futuramente para
exactly-once — validar se a área de produto/jurídico precisa formalizar esse aviso no contrato
de integração.]

## Referências

- `TRANSCRICAO.md` [09:24]-[09:26] — debate decisivo sobre a garantia at-least-once, a objeção
  de Sofia e a defesa de Diego com precedente de mercado (Stripe, GitHub).
- `TRANSCRICAO.md` [09:50]-[09:51] — confirmação da convenção de UUID para o `event_id`,
  alinhada ao restante do projeto.
- `prisma/schema.prisma:26,41,57,75,100,117` — convenção existente de UUID (`@db.Char(36)`) como
  chave primária em todas as entidades do domínio, base técnica direta para o formato do
  `event_id`.
- `package.json:32` — dependência `uuid` v11.0.3 já disponível no projeto para geração do
  identificador, sem necessidade de nova dependência.
