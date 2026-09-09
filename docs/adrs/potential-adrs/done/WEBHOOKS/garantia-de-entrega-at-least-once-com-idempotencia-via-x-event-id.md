# ADR em Potencial: Garantia de Entrega At-Least-Once com Idempotência via X-Event-Id

**Módulo**: WEBHOOKS
**Categoria**: Arquitetura (Protocolo / Contrato de API Externa)
**Prioridade**: Obrigatório Documentar (Pontuação: 125/150)
**Data de Identificação**: 2026-08-31

---

## O Que Foi Identificado

A equipe decidiu que a garantia de entrega de eventos de webhook será **at-least-once** (o cliente pode, em cenários de retry, receber o mesmo evento mais de uma vez), e não exactly-once. Para permitir que o cliente lide com essa possibilidade, cada evento carrega um **`event_id` (UUID) único, gerado no momento em que o evento entra na outbox**, enviado no header **`X-Event-Id`**. A responsabilidade de deduplicação fica explicitamente do lado do cliente, que deve usar o `event_id` para descartar entregas duplicadas.

Esta é uma decisão de contrato de API externa — define o que a plataforma promete (e não promete) aos consumidores de webhook, e desloca uma responsabilidade de engenharia (dedup) para os clientes B2B integradores. Como o módulo webhooks ainda não existe, toda a evidência é extraída de `TRANSCRICAO.md`. O `event_id` terá origem em `webhook_outbox` (ver ADR potencial do padrão Outbox), reforçando o acoplamento entre as duas decisões.

## Por Que Isso Pode Merecer uma ADR

- **Impacto**: Define uma cláusula do contrato público de integração com todos os consumidores de webhook, presentes e futuros — não é um detalhe interno, é algo que Marcos (PM) já se comprometeu a documentar de forma destacada no portal de desenvolvedores (`[09:26]`).
- **Trade-offs**: Exactly-once foi conscientemente descartado por exigir coordenação bilateral complexa entre plataforma e cliente; at-least-once com dedup client-side foi escolhido por ser o padrão adotado por provedores de referência (Stripe, GitHub), citados explicitamente na reunião como precedente de mercado.
- **Complexidade**: Ainda que simples de implementar do lado da plataforma (gerar UUID na inserção), desloca complexidade real para o lado do cliente — decisão que tem peso de produto, não só técnico (por isso a resposta de Sofia, "Isso joga responsabilidade pro cliente", seguida da defesa explícita de Diego).
- **Conhecimento da Equipe**: Toda comunicação com clientes sobre a feature (documentação, portal de desenvolvedor, suporte) depende de deixar claro que duplicatas são possíveis e esperadas.
- **Implicações Futuras**: Uma vez que clientes implementem dedup baseada em `X-Event-Id`, migrar para outra semântica de entrega no futuro (ex.: exactly-once) seria uma mudança de contrato potencialmente disruptiva para integrações já em produção.
- **Contexto Temporal**: Decisão fechada integralmente nesta reunião; sem histórico de código anterior a analisar.

## Evidências Encontradas na Base de Código

### Arquivos Principais
- `prisma/schema.prisma` — todas as entidades existentes usam UUID (`@db.Char(36)`) como chave primária (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`), estabelecendo a convenção de geração de identificadores que a reunião decide seguir também para `event_id` (`[09:50]-[09:51]` Larissa/Diego: "UUID, segue o padrão do resto do projeto. Tudo é uuid.").
- `package.json` — dependência `uuid` v11.0.3 já presente no projeto, disponível para geração do `event_id` sem necessidade de nova dependência.

### Evidência no Código
```prisma
// prisma/schema.prisma — convenção de UUID já estabelecida em todo o schema
model Order {
  id String @id @default(uuid()) @db.Char(36)
  // ...
}
```
Esta convenção existente é a base técnica direta citada na reunião para a decisão de que o `event_id` (assim como o id da própria linha da outbox) também será UUID, mantendo consistência com o restante do sistema.

### Evidência na Transcrição
- `[09:24]` Diego: enunciado da garantia — "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado."
- `[09:25]` Diego: mecanismo de deduplicação — "A gente manda um event_id no header, X-Event-Id, com um UUID gerado quando o evento entra na outbox. É único por evento. Se o cliente recebeu duas vezes, ele dedupica pelo event_id do lado dele."
- `[09:25]` Sofia: objeção explícita registrada — "Isso joga responsabilidade pro cliente."
- `[09:25]` Diego: defesa da escolha com precedente de mercado — "Joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim. Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos."
- `[09:26]` Marcos: compromisso de documentação para os clientes — "Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema."
- `[09:26]` Larissa: fechamento formal — "Beleza. At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."

### Análise de Impacto (Git)
- Histórico do Git não aplicável — a lógica de geração e envio de `event_id`/`X-Event-Id` ainda não existe no repositório. Único commit do projeto é `init repository`.

### Alternativas (explicitamente discutidas na transcrição)
- **Garantia exactly-once** — descartada (`[09:25]`): exigiria coordenação complexa entre plataforma e cliente, considerada desproporcional ao ganho.

## Questões a abordar na ADR (se criada)

- Qual problema estava sendo resolvido (impossibilidade prática de garantir exactly-once em um sistema distribuído com retry)?
- Por que at-least-once com dedup client-side foi escolhido em vez de exactly-once, e quais precedentes de mercado sustentam essa escolha?
- Como o `event_id` é gerado e propagado (origem na inserção da outbox, mesmo valor reenviado em todas as tentativas de retry de um mesmo evento)?
- Quais são as consequências de longo prazo para os clientes integradores (necessidade de implementarem e manterem lógica de dedup própria) e como isso será comunicado (portal de desenvolvedor)?

## ADRs Potenciais Relacionadas
- Padrão Outbox no MySQL para Entrega de Eventos de Webhook (o `event_id` nasce no momento da inserção na outbox)
- Autenticação HMAC-SHA256 com Secret por Endpoint (headers de segurança e idempotência compõem juntos o contrato de requisição enviado ao cliente)

## Observações Adicionais

- O formato completo do payload (`event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`, decisão de não incluir `items` para manter o payload enxuto, `[09:43]` Diego) e a lista completa de headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type`, `[09:44]-[09:45]`) são detalhes de especificação de contrato que devem ser consolidados dentro desta ADR (ou de um FDD/spec de API), não como ADRs separadas (Sinal de Alerta 5) — a decisão arquitetural relevante já é capturada aqui: at-least-once + dedup via `event_id`.
