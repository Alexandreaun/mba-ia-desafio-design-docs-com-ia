# ADR em Potencial: Política de Retry com Backoff Exponencial e Dead Letter Queue

**Módulo**: WEBHOOKS
**Categoria**: Arquitetura (Resiliência / Confiabilidade)
**Prioridade**: Obrigatório Documentar (Pontuação: 115/150)
**Data de Identificação**: 2026-08-31

---

## O Que Foi Identificado

A equipe decidiu a estratégia completa de resiliência para falhas de entrega de webhook: **5 tentativas com backoff exponencial (1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas)**, totalizando quase 15 horas de janela entre a primeira falha e a última tentativa. Esgotadas as tentativas, o evento é movido para uma **tabela separada `webhook_dead_letter`** (não apenas marcado como "failed" na própria outbox), contendo payload, motivo da falha e timestamp. Um **endpoint administrativo de replay manual** (`POST /admin/webhooks/dead-letter/:id/replay`, exigindo role `ADMIN`) permite reintroduzir o evento na outbox como pendente.

Esta decisão define como o sistema se comporta perante indisponibilidade de clientes externos — um cenário que a equipe já vivenciou na prática ("já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada", `[09:16]` Diego). Como o módulo webhooks ainda não existe no código, toda a evidência é extraída de `TRANSCRICAO.md`; não há tabela `webhook_dead_letter` nem endpoint de replay implementados hoje.

## Por Que Isso Pode Merecer uma ADR

- **Impacto**: Determina a confiabilidade percebida pelos clientes B2B (Atlas, MaxDistribuição, Nova Cargo) da feature inteira — decide o que acontece quando a integração deles falha, e como a equipe interna recupera eventos perdidos.
- **Trade-offs**: A contagem de tentativas (5) e a progressão do backoff foram calibradas explicitamente contra dois extremos rejeitados na própria reunião — 3 tentativas (considerada agressiva demais para cobrir indisponibilidades de algumas horas) e retry indefinido (considerado um risco de eventos "pendurados para sempre").
- **Complexidade**: Introduz um novo modelo de dados (`webhook_dead_letter`) e um novo endpoint administrativo com controle de acesso e requisito de auditoria (quem executou o replay).
- **Conhecimento da Equipe**: Times de suporte/operações precisam saber que existe um mecanismo de replay manual e a janela de tempo (~15h) antes de um evento cair definitivamente em DLQ; isso molda expectativas de SLA reportadas a clientes.
- **Implicações Futuras**: Este desenho assume execução single-worker (ver ADR do worker); qualquer evolução para múltiplos workers processando retries em paralelo exigirá reavaliar coordenação de tentativas.
- **Contexto Temporal**: Decisão fechada integralmente nesta reunião; não há histórico de código anterior a analisar.

## Evidências Encontradas na Base de Código

### Arquivos Principais
- [`src/shared/errors/http-errors.ts`](../../../../../src/shared/errors/http-errors.ts) — hierarquia `AppError` existente (`ConflictError`, `NotFoundError`, `UnprocessableEntityError`, etc.), referência de convenção para os futuros erros `WEBHOOK_DELIVERY_FAILED`/`WEBHOOK_DEAD_LETTER_NOT_FOUND` que o replay endpoint precisará usar.
- [`src/middlewares/auth.middleware.ts`](../../../../../src/middlewares/auth.middleware.ts) — `requireRole(...roles)`, mecanismo que a reunião decide reaproveitar diretamente para proteger o endpoint de replay (`requireRole('ADMIN')`), sem criar novo mecanismo de autorização.
- `prisma/schema.prisma` — não contém (ainda) modelo `webhook_dead_letter` ou `WebhookDeadLetter`.

### Evidência no Código
```typescript
// src/middlewares/auth.middleware.ts (mecanismo a ser reaproveitado no endpoint de replay)
export function requireRole(...roles: AuthUser['role'][]): RequestHandler {
  return (req, _res, next) => {
    if (!req.user) { next(new UnauthorizedError()); return; }
    if (!roles.includes(req.user.role)) { next(new ForbiddenError('Insufficient permissions')); return; }
    next();
  };
}
```

### Evidência na Transcrição
- `[09:15]` Diego: proposta da estratégia — "Backoff exponencial. Tenta de novo depois de algum tempo, vai aumentando o intervalo, e depois de um teto de tentativas considera falha permanente e move pra DLQ."
- `[09:15]-[09:16]` Bruno/Diego: debate e rejeição de 3 tentativas — "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria."
- `[09:15]` Diego: rejeição de retry indefinido — risco de "evento ficar pendurado pra sempre se o cliente sumiu".
- `[09:17]` Larissa: fechamento formal — "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h."
- `[09:18]` Diego: decisão de tabela separada para DLQ — "Eu fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp. Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento."
- `[09:18]-[09:19]` Diego/Larissa: endpoint de replay manual — `POST /admin/webhooks/dead-letter/:id/replay`, recoloca o evento na outbox como pendente.
- `[09:35]-[09:36]` Sofia/Larissa: exigência de role `ADMIN` no endpoint de replay e requisito de auditoria — "Mexer em fila de entrega de notificação não é coisa de operador. E o endpoint de admin tem que logar quem fez o replay, pra auditoria." Decisão de reaproveitar `requireRole` já existente.

### Análise de Impacto (Git)
- Histórico do Git não aplicável — nem a tabela `webhook_dead_letter`, nem o endpoint de replay existem no repositório hoje. O único commit do projeto é `init repository`, sem granularidade temporal a explorar.

### Alternativas (explicitamente discutidas na transcrição)
- **Retry indefinido sem teto de tentativas** — rejeitada (`[09:15]`): risco de eventos pendurados indefinidamente.
- **3 tentativas (mais agressivo)** — rejeitada (`[09:15]-[09:16]`): insuficiente para cobrir indisponibilidades de cliente de algumas horas.
- **Marcar falha definitiva na própria tabela outbox (sem tabela DLQ separada)** — descartada implicitamente em favor de tabela dedicada (`[09:18]`), por clareza de leitura da outbox principal e por servir de evidência para debug/reprocessamento.

## Questões a abordar na ADR (se criada)

- Qual problema estava sendo resolvido (indisponibilidade temporária de clientes externos sem perda permanente de eventos)?
- Por que 5 tentativas com essa progressão específica de backoff, e não um número maior/menor ou backoff linear?
- Por que uma tabela DLQ separada em vez de um campo de status na própria outbox?
- Quais são as consequências de longo prazo (necessidade de processo operacional para monitorar e revisar a DLQ; ausência de notificação automática ao cliente final sobre falhas — adiada para fase futura, `[09:37]-[09:38]`)?

## ADRs Potenciais Relacionadas
- Padrão Outbox no MySQL para Entrega de Eventos de Webhook (a DLQ é o destino final dos eventos que a outbox não conseguiu entregar)
- Worker de Entrega em Processo Separado com Polling (é o worker que executa as tentativas de retry e decide quando mover para DLQ)

## Observações Adicionais

- Notificação por e-mail ao cliente em caso de falhas repetidas foi explicitamente **adiada** para fase futura (`[09:37]-[09:38]` Marcos/Larissa: "Não. Email tá fora de escopo dessa fase") — não é uma decisão confirmada e não deve constar como parte desta ADR além de uma nota de "fora de escopo".
- O timeout de 10 segundos do HTTP call do worker (`[09:42]` Diego/Sofia) é um parâmetro que compõe esta política de resiliência — deve ser incluído como detalhe de configuração dentro desta ADR formal, não como ADR separada (Sinal de Alerta 5).
