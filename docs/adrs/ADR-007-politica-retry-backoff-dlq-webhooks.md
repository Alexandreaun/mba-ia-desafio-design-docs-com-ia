# ADR-007: Política de Retry com Backoff Exponencial e Dead Letter Queue

**Status:** Aceita
**Date:** 2026-08-31
**Related ADRs:** ADR-006, ADR-008

## Status

Aceita. A estratégia de resiliência foi fechada formalmente na reunião de refinamento técnico ("Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h", `[09:17]` Larissa), incluindo o modelo de Dead Letter Queue (DLQ) e o endpoint de replay administrativo. O módulo de webhooks ainda não existe na base de código — trata-se de uma decisão de design fechada, pendente apenas de implementação.

## Contexto

O sistema de pedidos precisa notificar clientes B2B externos (Atlas, MaxDistribuição, Nova Cargo) sobre mudanças de status via webhook, mas endpoints de clientes falham ou ficam indisponíveis por períodos variáveis — a equipe já vivenciou na prática um cliente com indisponibilidade de duas horas durante manutenção planejada (`[09:16]` Diego). O problema de negócio central é evitar perda permanente de eventos de notificação sem, ao mesmo tempo, deixar tentativas de entrega "penduradas" indefinidamente caso o cliente nunca volte a responder. Essa janela de resiliência também molda o que times de suporte e operações podem prometer a clientes em termos de recuperação de eventos, e por isso precisa ser conhecida por essas equipes antes de a feature entrar em produção — não apenas pelo time de engenharia.

Do ponto de vista técnico, o módulo de webhooks ainda não existe em `src/` nem em `prisma/schema.prisma` — não há histórico de código anterior a analisar, e toda a evidência desta decisão vem da transcrição da reunião.

A política de retry aqui definida depende diretamente de duas outras decisões de arquitetura fechadas na mesma reunião: o padrão de outbox como origem dos eventos a serem entregues, e a existência de um worker dedicado em processo separado, rodando em modelo single-worker, responsável por executar as tentativas e decidir quando mover um evento para a DLQ. Essa dependência de execução single-worker é o que sustenta a suposição de que as tentativas de um mesmo evento não são processadas concorrentemente.

## Decisão

A equipe decidiu adotar backoff exponencial com 5 tentativas de entrega, totalizando uma janela de quase 15 horas entre a primeira falha e o esgotamento das tentativas. Esse número e essa progressão foram calibrados deliberadamente para cobrir janelas de indisponibilidade de cliente da ordem de horas (como o caso real de manutenção citado por Diego), sem manter eventos ativos por tempo indeterminado. A progressão de intervalos entre tentativas é:

1. 1ª nova tentativa: após 1 minuto.
2. 2ª nova tentativa: após 5 minutos.
3. 3ª nova tentativa: após 30 minutos.
4. 4ª nova tentativa: após 2 horas.
5. 5ª nova tentativa (última): após 12 horas.

Esgotadas as 5 tentativas, o evento é movido para uma tabela dedicada `webhook_dead_letter`, em vez de apenas sinalizado como "failed" na própria outbox, preservando a outbox principal limpa e mantendo evidência auditável para debug e reprocessamento (`[09:18]` Diego). Essa decisão inclui os seguintes elementos, todos fechados na reunião:

- Registro em `webhook_dead_letter` com payload do evento, motivo da falha e timestamp.
- Reintrodução manual do evento na outbox como pendente via endpoint administrativo (`POST /admin/webhooks/dead-letter/:id/replay`).
- Restrição de acesso ao endpoint de replay exclusivamente à role `ADMIN`, reaproveitando o mecanismo `requireRole` já existente no projeto em vez de criar um novo controle de autorização (`[09:35]-[09:36]` Sofia/Larissa).
- Exigência de log de auditoria registrando qual usuário administrador executou cada replay.

O timeout de 10 segundos por chamada HTTP do worker (`[09:42]` Diego) é tratado como parâmetro de configuração desta mesma política de resiliência, não como decisão arquitetural separada: um cliente que não responder dentro desse intervalo é considerado falha e entra no fluxo de retry descrito acima.

## Alternativas Consideradas

### Retry indefinido sem teto de tentativas

**Prós:**
- Nenhum evento seria descartado do fluxo de retry ativo.
- Simplicidade conceitual, sem necessidade de modelar uma DLQ separada.
- Elimina a decisão de calibrar um número "certo" de tentativas.

**Contras:**
- Risco explícito de eventos "pendurados para sempre" se o cliente nunca mais responder (`[09:15]` Diego).
- Não oferece um ponto de corte claro para intervenção operacional ou replay manual.
- Dificulta estimar e monitorar a carga de retries pendentes ao longo do tempo.
- Sem teto de tentativas, um cliente permanentemente offline consumiria capacidade do worker indefinidamente.

### 3 tentativas com backoff mais agressivo

**Prós:**
- Libera recursos do worker mais rapidamente por evento malsucedido.
- Sinaliza falha permanente antes, reduzindo o tempo de eventos "em voo".
- Progressão de backoff mais simples de calcular e testar.

**Contras:**
- Insuficiente para cobrir indisponibilidades de cliente de algumas horas — o próprio time relatou um caso real de duas horas de manutenção planejada (`[09:16]` Diego).
- Geraria falsos positivos de "cliente indisponível" para janelas de manutenção legítimas.
- Obrigaria replay manual com maior frequência, aumentando a carga operacional do time.
- Reduz a janela total de tolerância a falhas de quase 15 horas para poucas dezenas de minutos.

### Falha definitiva sinalizada na própria tabela outbox (sem DLQ separada)

**Prós:**
- Menor complexidade de modelo de dados, sem necessidade de uma tabela nova.
- Menos uma entidade e um fluxo de acesso a manter no longo prazo.
- Reaproveita a estrutura de outbox já definida em vez de introduzir um segundo modelo de persistência.

**Contras:**
- Polui a leitura operacional da outbox principal ao misturar eventos pendentes com eventos definitivamente falhos.
- Perde a separação de responsabilidades entre fila de entrega ativa e histórico de falhas para debug.
- Dificulta a criação de um fluxo de replay administrativo dedicado e auditável (`[09:18]` Diego).

## Consequências

A decisão garante um contrato de resiliência previsível e comunicável a clientes e times de suporte: eventos de notificação sobrevivem a até ~15 horas de indisponibilidade do cliente antes de exigir intervenção manual, e nenhuma falha é descartada silenciosamente — toda falha definitiva fica registrada em `webhook_dead_letter` com payload e motivo, disponível para reprocessamento via endpoint administrativo auditado. O reaproveitamento do `requireRole` existente evita introduzir um novo mecanismo de autorização só para este fluxo.

Em contrapartida, a equipe assume trade-offs operacionais explícitos ao adotar este desenho:

- O modelo pressupõe execução em um único worker (single-worker); qualquer evolução futura para múltiplos workers processando retries em paralelo exigirá reavaliar a coordenação de tentativas para não duplicar ou perder execuções.
- Não há notificação automática ao cliente final quando um evento cai em DLQ — isso foi explicitamente adiado para uma fase futura (`[09:37]-[09:38]` Marcos/Larissa), o que significa que, até lá, a detecção de problemas recorrentes de entrega depende do próprio cliente reportar ou de operação manual/interna.
- [NEEDS INPUT: não há definição, nem na transcrição nem no código, de um processo ou responsável formal para monitorar e revisar periodicamente os itens acumulados em `webhook_dead_letter`, apesar de a própria equipe ter levantado essa necessidade como consequência de longo prazo].
- [NEEDS INPUT: não foi discutido na reunião nem há evidência no código sobre por quanto tempo registros de DLQ já reprocessados (ou nunca reprocessados) devem ser mantidos antes de purga ou arquivamento].

Em suma, o trade-off central assumido é trocar simplicidade de curto prazo (menos infraestrutura, menos processos) por uma garantia de resiliência auditável e recuperável — ao custo de exigir, no futuro, definição de processo operacional de monitoramento da DLQ e de uma eventual reavaliação de arquitetura caso o volume de retries justifique múltiplos workers.

## Referências

- `TRANSCRICAO.md` (`[09:14]`-`[09:19]`) — debate e fechamento da política de retry, backoff e criação da tabela DLQ separada.
- `TRANSCRICAO.md` (`[09:35]`-`[09:36]`) — decisão de exigir role `ADMIN` e auditoria no endpoint de replay manual, reaproveitando controle de acesso existente.
- `src/middlewares/auth.middleware.ts:49` — função `requireRole`, mecanismo de autorização a ser reaproveitado diretamente pelo endpoint de replay.
- `src/shared/errors/http-errors.ts:27` (`NotFoundError`), `:33` (`ConflictError`), `:39` (`UnprocessableEntityError`) — convenção de hierarquia `AppError` de referência para os futuros erros `WEBHOOK_DELIVERY_FAILED`/`WEBHOOK_DEAD_LETTER_NOT_FOUND`.
- `prisma/schema.prisma` — ausência atual de modelo para `webhook_dead_letter`, confirmando que esta é uma decisão de design ainda não implementada.
