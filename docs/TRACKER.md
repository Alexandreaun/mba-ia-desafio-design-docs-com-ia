# Tracker de Rastreabilidade

Gerado em: 2026-09-12
Documentos analisados: docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/ADR-001-estrategia-autenticacao-jwt-stateless-auth.md, docs/adrs/ADR-002-prisma-orm-camada-acesso-dados-config.md, docs/adrs/ADR-003-publicacao-atomica-eventos-outbox-orders.md, docs/adrs/ADR-004-autenticacao-hmac-secret-por-endpoint-webhooks.md, docs/adrs/ADR-005-garantia-entrega-at-least-once-webhooks.md, docs/adrs/ADR-006-outbox-pattern-webhooks.md, docs/adrs/ADR-007-politica-retry-backoff-dlq-webhooks.md, docs/adrs/ADR-008-worker-entrega-processo-separado-polling-webhooks.md
Cobertura: 297 de 297 itens identificados rastreados (100%), sobre um total de 339 itens inventariados (42 já marcados como hipótese/lacuna pelos próprios documentos-fonte, excluídos do denominador; 0 divergências)

---

## Legenda

- **Fonte `TRANSCRICAO`**: item rastreado a um trecho de `TRANSCRICAO.md`, no formato `[hh:mm] Nome`.
- **Fonte `CODIGO`**: item rastreado a um caminho real de arquivo do código-fonte do projeto.

---

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência de entrega abaixo de 10s aceito como tempo real, pior caso 2s (polling) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Garantir 100% de correspondência transição-evento via atomicidade transacional | TRANSCRICAO | [09:06] Diego |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Reter clientes B2B entregando dentro do prazo comunicado (3 sprints, fim do trimestre) | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Tolerar indisponibilidade de cliente por ~15h via 5 tentativas com backoff | TRANSCRICAO | [09:17] Diego |
| PRD-ESCOPO-01 | docs/PRD.md | Item Incluso | CRUD completo de webhook, secret gerada e devolvida só na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-ESCOPO-02 | docs/PRD.md | Item Incluso | Filtro de interesse por status de pedido na inserção do evento | TRANSCRICAO | [09:34] Bruno |
| PRD-ESCOPO-03 | docs/PRD.md | Item Incluso | Consulta de histórico de entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-ESCOPO-04 | docs/PRD.md | Item Incluso | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-ESCOPO-05 | docs/PRD.md | Item Incluso | Endpoint admin de replay de DLQ restrito a ADMIN, com log de auditoria | TRANSCRICAO | [09:18] Diego |
| PRD-ESCOPO-06 | docs/PRD.md | Item Incluso | Publicação atômica do evento dentro da transação de mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-ESCOPO-07 | docs/PRD.md | Item Incluso | Entrega via worker separado, HMAC-SHA256, at-least-once, retry com backoff | TRANSCRICAO | [09:11] Diego |
| PRD-FORA-01 | docs/PRD.md | Item Fora de Escopo | Notificação por e-mail em falhas recorrentes, adiada para fase futura | TRANSCRICAO | [09:37] Larissa |
| PRD-FORA-02 | docs/PRD.md | Item Fora de Escopo | Rate limiting de envio para clientes com picos de eventos | TRANSCRICAO | [09:39] Larissa |
| PRD-FORA-03 | docs/PRD.md | Item Fora de Escopo | Dashboard visual para o cliente gerenciar webhooks | TRANSCRICAO | [09:40] Larissa |
| PRD-FORA-04 | docs/PRD.md | Item Fora de Escopo | Webhooks inbound descartados; fluxo estritamente outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-FORA-05 | docs/PRD.md | Item Fora de Escopo | Arquivamento/purga de eventos entregues após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-FORA-06 | docs/PRD.md | Item Fora de Escopo | Múltiplos workers em paralelo com ordering global | TRANSCRICAO | [09:13] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | FR-001 Cadastro de webhook via POST com URL, status de interesse e secret gerada | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | FR-002 Edição de webhook via PATCH | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | FR-003 Remoção de webhook via DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | FR-004 Listagem de webhooks por customer, customerId fora do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | FR-005 Filtragem de eventos por status de interesse na inserção do outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | FR-006 Consulta de histórico de entregas via GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | FR-007 Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | FR-008 Entrega assinada HMAC-SHA256 com timeout de 10s | TRANSCRICAO | [09:42] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | FR-009 Retry com backoff exponencial 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | FR-010 DLQ e reprocessamento manual restrito a ADMIN com auditoria | TRANSCRICAO | [09:18] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega <10s comum, pior caso 2s | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Worker independente de reinícios/deploys da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 com secret exclusiva por endpoint, nunca global | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | URLs de webhook obrigatoriamente HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB com erro explícito | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-07-T | docs/PRD.md | Requisito Não Funcional | CRUD exige apenas JWT válido; replay de DLQ exige role ADMIN | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-07-C | docs/PRD.md | Requisito Não Funcional | Replay de DLQ reaproveita mecanismo requireRole já existente | CODIGO | src/middlewares/auth.middleware.ts:49-61 |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Histórico de entregas consultável pelo cliente via API | TRANSCRICAO | [09:34] Marcos |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Falhas definitivas auditáveis em DLQ com motivo registrado | TRANSCRICAO | [09:18] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Módulo reaproveita logger Pino já existente, sem novo mecanismo | TRANSCRICAO | [09:29] Bruno |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Inserção do evento na mesma transação SQL da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once com dedup via X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Ordering apenas por order_id e apenas em regime single-worker | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-14 | docs/PRD.md | Requisito Não Funcional | Novos endpoints seguem convenção de versionamento /api/v1 | CODIGO | src/app.ts:67 |
| PRD-NFR-15 | docs/PRD.md | Requisito Não Funcional | Nenhuma rota/comportamento externo dos módulos existentes é alterado | CODIGO | src/routes/index.ts |
| PRD-NFR-16 | docs/PRD.md | Requisito Não Funcional | Toda execução do replay administrativo gera registro de auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-17 | docs/PRD.md | Requisito Não Funcional | Acessibilidade não aplicável; feature sem interface visual | TRANSCRICAO | [09:39] Larissa |
| PRD-DEC-01 | docs/PRD.md | Decisão | Publicação atômica do evento dentro da transação de changeStatus (ADR-003) | TRANSCRICAO | [09:41] Diego |
| PRD-DEC-02 | docs/PRD.md | Decisão | Padrão outbox sobre MySQL existente, sem fila externa (ADR-006) | TRANSCRICAO | [09:07] Diego |
| PRD-DEC-03 | docs/PRD.md | Decisão | HMAC-SHA256, secret única por endpoint, rotação grace period 24h (ADR-004) | TRANSCRICAO | [09:22] Diego |
| PRD-DEC-04 | docs/PRD.md | Decisão | Garantia at-least-once com dedup via X-Event-Id (ADR-005) | TRANSCRICAO | [09:25] Diego |
| PRD-DEC-05 | docs/PRD.md | Decisão | Retry backoff exponencial 5 tentativas e DLQ em tabela separada (ADR-007) | TRANSCRICAO | [09:17] Larissa |
| PRD-DEC-06 | docs/PRD.md | Decisão | Worker em processo separado, polling a cada 2s (ADR-008) | TRANSCRICAO | [09:11] Diego |
| PRD-DEC-07 | docs/PRD.md | Decisão | Reuso da autenticação JWT/RBAC existente sem alterações (ADR-001) | TRANSCRICAO | [09:36] Marcos |
| PRD-DEC-08 | docs/PRD.md | Decisão | Prisma como camada de acesso única, PrismaClient próprio por processo (ADR-002) | TRANSCRICAO | [09:29] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Ponto de integração único em OrderService.changeStatus | CODIGO | src/modules/orders/order.service.ts:126-179 |
| PRD-DEP-02 | docs/PRD.md | Dependência | Hierarquia AppError e middleware de erro centralizado reaproveitados | CODIGO | src/shared/errors/app-error.ts |
| PRD-DEP-03 | docs/PRD.md | Dependência | Autenticação e RBAC existentes (authenticate/requireRole) reaproveitados | CODIGO | src/middlewares/auth.middleware.ts:27-61 |
| PRD-DEP-04-C | docs/PRD.md | Dependência | Schema atual não possui nenhum modelo de webhook/outbox/DLQ | CODIGO | prisma/schema.prisma |
| PRD-DEP-04-T | docs/PRD.md | Dependência | Novas tabelas devem seguir convenção de UUID como chave primária | TRANSCRICAO | [09:51] Larissa |
| PRD-DEP-05-T | docs/PRD.md | Dependência | Novo processo worker (src/worker.ts) espelhando src/server.ts | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-05-C | docs/PRD.md | Dependência | Worker deve instanciar PrismaClient próprio via createPrismaClient() | CODIGO | src/config/database.ts |
| PRD-DEP-06 | docs/PRD.md | Dependência | Revisão de segurança dedicada da Sofia antes do deploy (2 dias úteis) | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-07 | docs/PRD.md | Dependência | Comunicação de prazo aos três clientes B2B pelo Marcos | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-08 | docs/PRD.md | Dependência | Sessão de revisão técnica de design antes da implementação | TRANSCRICAO | [09:50] Larissa |
| PRD-RISCO-01 | docs/PRD.md | Risco | Worker sem supervisão/restart definido pode interromper entrega silenciosamente | TRANSCRICAO | [09:11] Diego |
| PRD-RISCO-02 | docs/PRD.md | Risco | DLQ pode acumular falhas sem monitoramento formal | TRANSCRICAO | [09:18] Diego |
| PRD-RISCO-03 | docs/PRD.md | Risco | Risco de churn da Atlas Comercial se prazo de 3 sprints não for cumprido | TRANSCRICAO | [09:00] Marcos |
| PRD-RISCO-04 | docs/PRD.md | Risco | Vazamento de secret compromete autenticidade de eventos | TRANSCRICAO | [09:22] Diego |
| PRD-CA-01 | docs/PRD.md | Critério de Aceitação | Cliente cadastra/edita/remove/lista webhooks, secret só na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de Aceitação | Mudança de status gera exatamente 1 evento por webhook interessado | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-03 | docs/PRD.md | Critério de Aceitação | Falha na inserção do evento causa rollback completo da transação | TRANSCRICAO | [09:40] Bruno |
| PRD-CA-04 | docs/PRD.md | Critério de Aceitação | Chamada de entrega inclui headers de evento, assinatura e webhook id | TRANSCRICAO | [09:44] Diego |
| PRD-CA-05 | docs/PRD.md | Critério de Aceitação | Evento reenviado preserva mesmo identificador em todas as tentativas | TRANSCRICAO | [09:25] Diego |
| PRD-CA-06 | docs/PRD.md | Critério de Aceitação | Falha segue progressão de 5 tentativas e move para DLQ só ao esgotar | TRANSCRICAO | [09:17] Diego |
| PRD-CA-07 | docs/PRD.md | Critério de Aceitação | Admin reprocessa evento em DLQ, ação registrada em log de auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-08 | docs/PRD.md | Critério de Aceitação | Usuário sem role ADMIN recebe acesso negado no replay | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-09 | docs/PRD.md | Critério de Aceitação | Cliente consulta histórico completo de entregas, incluindo falhas | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-10 | docs/PRD.md | Critério de Aceitação | Rotação de secret mantém secret anterior válida por 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-11 | docs/PRD.md | Critério de Aceitação | Cadastro com URL não HTTPS é rejeitado sem persistir registro | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-12 | docs/PRD.md | Critério de Aceitação | Nenhuma rota/comportamento hoje existente muda de comportamento externo | CODIGO | src/routes/index.ts |
| PRD-CA-13 | docs/PRD.md | Critério de Aceitação | Revisão de segurança da Sofia ocorre e é aprovada antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-TESTE-01 | docs/PRD.md | Estratégia de Teste | Teste de integração do fluxo transacional completo com rollback forçado | TRANSCRICAO | [09:40] Bruno |
| PRD-TESTE-02 | docs/PRD.md | Estratégia de Teste | Teste unitário da lógica de filtragem de interesse por status | TRANSCRICAO | [09:34] Bruno |
| PRD-TESTE-03 | docs/PRD.md | Estratégia de Teste | Teste de integração ponta a ponta do ciclo de retry e DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-TESTE-04 | docs/PRD.md | Estratégia de Teste | Teste de segurança da assinatura HMAC-SHA256 e grace period de secret | TRANSCRICAO | [09:21] Sofia |
| PRD-TESTE-05 | docs/PRD.md | Estratégia de Teste | Teste de permissão do endpoint de replay de DLQ (ADMIN vs não-ADMIN) | TRANSCRICAO | [09:36] Sofia |
| PRD-TESTE-06 | docs/PRD.md | Estratégia de Teste | Teste de validação de schema para URL não HTTPS e payload acima de 64KB | TRANSCRICAO | [09:24] Sofia |
| PRD-TESTE-07 | docs/PRD.md | Estratégia de Teste | Testes de integração/unitários seguindo padrão Vitest com banco real truncado | CODIGO | tests/setup.ts |
| PRD-TESTE-08 | docs/PRD.md | Estratégia de Teste | Revisão de segurança dedicada da Sofia como gate obrigatório antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-TESTE-09 | docs/PRD.md | Estratégia de Teste | Sessão de revisão técnica de design entre Larissa, Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| RFC-PROB-01 | docs/RFC.md | Requisito Funcional | Polling caro/lento para clientes B2B, risco concreto de churn | TRANSCRICAO | [09:00] Marcos |
| RFC-PROB-02 | docs/RFC.md | Restrição | Sistema não possui ponto de extensão para eventos de domínio assíncronos | CODIGO | src/modules/orders/order.service.ts:126-179 |
| RFC-OBJ-01 | docs/RFC.md | Objetivo | Notificar clientes B2B com latência <10s sem depender de polling | TRANSCRICAO | [09:02] Marcos |
| RFC-OBJ-02 | docs/RFC.md | Objetivo | Garantir que notificação nunca fique inconsistente com estado do pedido | TRANSCRICAO | [09:40] Bruno |
| RFC-OBJ-03 | docs/RFC.md | Objetivo | Não introduzir acoplamento síncrono com disponibilidade de sistemas externos | TRANSCRICAO | [09:04] Bruno |
| RFC-OBJ-04 | docs/RFC.md | Objetivo | Fornecer autenticação e integridade verificável para eventos entregues | TRANSCRICAO | [09:20] Sofia |
| RFC-OBJ-05 | docs/RFC.md | Objetivo | Reaproveitar ao máximo os padrões arquiteturais já estabelecidos | TRANSCRICAO | [09:30] Larissa |
| RFC-OBJ-06 | docs/RFC.md | Objetivo | Entregar a primeira versão dentro da estimativa de três sprints | TRANSCRICAO | [09:46] Larissa |
| RFC-FORA-01 | docs/RFC.md | Item Fora de Escopo | Webhooks inbound; fluxo estritamente outbound | TRANSCRICAO | [09:02] Sofia |
| RFC-FORA-02 | docs/RFC.md | Item Fora de Escopo | Notificação alternativa por e-mail adiada para fase futura | TRANSCRICAO | [09:37] Larissa |
| RFC-FORA-03 | docs/RFC.md | Item Fora de Escopo | Rate limiting de envio deixado como observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-FORA-04 | docs/RFC.md | Item Fora de Escopo | Dashboard visual do cliente fora de escopo desta fase | TRANSCRICAO | [09:40] Larissa |
| RFC-FORA-05 | docs/RFC.md | Item Fora de Escopo | Endurecimento futuro de RBAC no CRUD de webhook não decidido | TRANSCRICAO | [09:37] Sofia |
| RFC-FORA-06 | docs/RFC.md | Item Fora de Escopo | Arquivamento/purga de eventos entregues após 30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-FORA-07 | docs/RFC.md | Item Fora de Escopo | Suporte a múltiplos workers com ordering global | TRANSCRICAO | [09:13] Diego |
| RFC-RF-01 | docs/RFC.md | Requisito Funcional | Cadastro de webhook com URL, status de interesse, secret gerada na criação | TRANSCRICAO | [09:31] Marcos |
| RFC-RF-02 | docs/RFC.md | Requisito Funcional | Editar (PATCH), remover (DELETE) e listar (GET) webhooks por customer | TRANSCRICAO | [09:33] Bruno |
| RFC-RF-03 | docs/RFC.md | Requisito Funcional | Filtrar inserção do evento por webhooks interessados no status resultante | TRANSCRICAO | [09:34] Diego |
| RFC-RF-04 | docs/RFC.md | Requisito Funcional | Consulta de histórico de entregas via GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| RFC-RF-05 | docs/RFC.md | Requisito Funcional | Endpoint admin de replay de DLQ restrito a role ADMIN | TRANSCRICAO | [09:35] Diego |
| RFC-RF-06 | docs/RFC.md | Requisito Funcional | Replay administrativo registrado em log de auditoria | TRANSCRICAO | [09:36] Sofia |
| RFC-RF-07 | docs/RFC.md | Requisito Funcional | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| RFC-RF-08 | docs/RFC.md | Requisito Funcional | event_id único permanece o mesmo em todas as tentativas de reenvio | TRANSCRICAO | [09:25] Diego |
| RFC-RF-09 | docs/RFC.md | Requisito Funcional | Restante do CRUD exige apenas JWT válido, sem role específica | TRANSCRICAO | [09:37] Sofia |
| RFC-RF-10 | docs/RFC.md | Contrato Público | Headers de entrega X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| RFC-RF-11 | docs/RFC.md | Contrato Público | Payload enxuto sem itens do pedido; detalhes via GET /orders/:id | TRANSCRICAO | [09:43] Diego |
| RFC-RNF-01 | docs/RFC.md | Requisito Não Funcional | Latência <10s comum, pior caso 2s pela cadência de polling | TRANSCRICAO | [09:09] Diego |
| RFC-RNF-02 | docs/RFC.md | Requisito Não Funcional | Garantia transacional de atomicidade entre status e evento | TRANSCRICAO | [09:06] Diego |
| RFC-RNF-03 | docs/RFC.md | Requisito Não Funcional | Worker não afetado por reinícios/deploys da API, e vice-versa | TRANSCRICAO | [09:11] Diego |
| RFC-RNF-04 | docs/RFC.md | Requisito Não Funcional | Autenticidade via HMAC-SHA256, secret exclusiva, HTTPS obrigatório | TRANSCRICAO | [09:21] Sofia |
| RFC-RNF-05 | docs/RFC.md | Requisito Não Funcional | Limite de payload 64KB, erro explícito em caso de excesso | TRANSCRICAO | [09:24] Sofia |
| RFC-RNF-06 | docs/RFC.md | Requisito Não Funcional | Tolerância a indisponibilidade por ~15h via retry com backoff | TRANSCRICAO | [09:16] Diego |
| RFC-RNF-07 | docs/RFC.md | Requisito Não Funcional | Histórico de entregas consultável e falhas auditáveis em DLQ | TRANSCRICAO | [09:34] Marcos |
| RFC-RNF-08-T | docs/RFC.md | Requisito Não Funcional | Módulo segue convenção estrutural e prefixo de erro WEBHOOK_* | TRANSCRICAO | [09:28] Bruno |
| RFC-RNF-08-C | docs/RFC.md | Requisito Não Funcional | Hierarquia AppError já existente referência para novos erros | CODIGO | src/shared/errors/http-errors.ts |
| RFC-RNF-09 | docs/RFC.md | Requisito Não Funcional | Desenho aceita premissa single-worker; múltiplos workers é TBD | TRANSCRICAO | [09:12] Diego |
| RFC-RESTR-01-T | docs/RFC.md | Restrição | Time pequeno motivou reaproveitar MySQL em vez de Redis Streams | TRANSCRICAO | [09:07] Diego |
| RFC-RESTR-01-C | docs/RFC.md | Restrição | Projeto usa MySQL como único banco de dados via Prisma ORM | CODIGO | prisma/schema.prisma:5-9 |
| RFC-RESTR-02 | docs/RFC.md | Restrição | MySQL não oferece LISTEN/NOTIFY; trigger só executa SQL | TRANSCRICAO | [09:09] Diego |
| RFC-RESTR-03 | docs/RFC.md | Restrição | Autenticação interna JWT/RBAC reaproveitada sem alteração | CODIGO | src/middlewares/auth.middleware.ts:27-61 |
| RFC-RESTR-04 | docs/RFC.md | Restrição | Novo erro de webhook deve seguir hierarquia AppError com prefixo WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts |
| RFC-RESTR-05 | docs/RFC.md | Restrição | Prazo comercial de 3 sprints, fim de trimestre condicionando contrato Atlas | TRANSCRICAO | [09:45] Larissa |
| RFC-RESTR-06 | docs/RFC.md | Restrição | Secret única por endpoint motivada por incidente real de vazamento | TRANSCRICAO | [09:22] Diego |
| RFC-RESTR-07-T | docs/RFC.md | Restrição | Nova tabela de outbox deve seguir convenção de UUID do schema atual | TRANSCRICAO | [09:51] Larissa |
| RFC-RESTR-07-C | docs/RFC.md | Restrição | Todas as entidades do schema atual usam UUID como chave primária | CODIGO | prisma/schema.prisma |
| RFC-RESTR-08 | docs/RFC.md | Restrição | Módulo webhooks segue convenção estrutural routes/controller/service/repository | TRANSCRICAO | [09:27] Bruno |
| RFC-ALT-01 | docs/RFC.md | Alternativa Considerada | Disparo síncrono de webhook dentro de changeStatus, descartado | TRANSCRICAO | [09:03] Larissa |
| RFC-ALT-02 | docs/RFC.md | Alternativa Considerada | Fila externa dedicada (Redis Streams), descartada por overengineering | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Considerada | Trigger de banco para acionar worker reativamente, descartado | TRANSCRICAO | [09:09] Diego |
| RFC-TRADEOFF-01 | docs/RFC.md | Trade-off | Consistência forte vs. latência mínima de até 2s do polling | TRANSCRICAO | [09:09] Diego |
| RFC-TRADEOFF-02 | docs/RFC.md | Trade-off | Simplicidade operacional (single-worker) vs. throughput futuro | TRANSCRICAO | [09:12] Diego |
| RFC-TRADEOFF-03 | docs/RFC.md | Trade-off | Reuso do MySQL existente vs. componente de mensageria dedicado | TRANSCRICAO | [09:07] Diego |
| RFC-TRADEOFF-04 | docs/RFC.md | Trade-off | Dedup delegada ao cliente vs. simplicidade de entrega no backend | TRANSCRICAO | [09:25] Sofia |
| RFC-TRADEOFF-05 | docs/RFC.md | Trade-off | Janela de resiliência ampla (~15h) vs. eventos em voo por mais tempo | TRANSCRICAO | [09:16] Diego |
| RFC-TRADEOFF-06 | docs/RFC.md | Trade-off | Isolamento de credenciais por endpoint vs. complexidade de gestão | TRANSCRICAO | [09:21] Sofia |
| RFC-TRADEOFF-07 | docs/RFC.md | Trade-off | Acoplamento transacional entre domínios vs. garantia de consistência | TRANSCRICAO | [09:41] Diego |
| RFC-RISCO-01 | docs/RFC.md | Risco | Worker de longa duração sem mecanismo de supervisão/restart definido | TRANSCRICAO | [09:11] Diego |
| RFC-RISCO-02 | docs/RFC.md | Risco | Aumento de contenção de locks na transação já pesada de changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-RISCO-03 | docs/RFC.md | Risco | Vazamento de secret comprometendo autenticidade de eventos | TRANSCRICAO | [09:22] Diego |
| RFC-RISCO-04 | docs/RFC.md | Risco | Cliente processa o mesmo evento mais de uma vez (at-least-once) | TRANSCRICAO | [09:24] Diego |
| RFC-RISCO-05 | docs/RFC.md | Risco | Eventos pendurados indefinidamente para cliente permanentemente offline | TRANSCRICAO | [09:15] Diego |
| RFC-RISCO-06 | docs/RFC.md | Risco | Falha definitiva de entrega passa despercebida sem monitoramento de DLQ | TRANSCRICAO | [09:18] Diego |
| RFC-RISCO-07 | docs/RFC.md | Risco | Pico de eventos para um cliente sobrecarregando seu endpoint | TRANSCRICAO | [09:38] Diego |
| RFC-RISCO-08 | docs/RFC.md | Risco | Perda de garantia de ordering ao evoluir para múltiplos workers | TRANSCRICAO | [09:12] Diego |
| RFC-RISCO-09 | docs/RFC.md | Risco | Migração futura para exactly-once disruptiva para integrações já em produção | TRANSCRICAO | [09:25] Diego |
| RFC-QA-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio para clientes com picos de eventos | TRANSCRICAO | [09:38] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em Aberto | Notificação automática ao cliente em falhas recorrentes (ex.: e-mail) | TRANSCRICAO | [09:37] Marcos |
| RFC-QA-03 | docs/RFC.md | Questão em Aberto | Endurecimento futuro de RBAC no CRUD de configuração de webhook | TRANSCRICAO | [09:36] Sofia |
| ADR-001 | docs/adrs/ADR-001-estrategia-autenticacao-jwt-stateless-auth.md | Decisão | Autenticação stateless via JWT com verificação de senha via bcrypt | CODIGO | src/modules/auth/auth.service.ts:31-50 |
| ADR-001-CONSEQ-01 | docs/adrs/ADR-001-estrategia-autenticacao-jwt-stateless-auth.md | Consequência | Sem refresh token nem mecanismo de revogação/blacklist de JWT | CODIGO | src/middlewares/auth.middleware.ts:27-47 |
| ADR-001-CONSEQ-02 | docs/adrs/ADR-001-estrategia-autenticacao-jwt-stateless-auth.md | Consequência | Sem estratégia de rotação para o JWT_SECRET | CODIGO | src/config/env.ts:8 |
| ADR-002 | docs/adrs/ADR-002-prisma-orm-camada-acesso-dados-config.md | Decisão | Prisma como ORM único, PrismaClient singleton por processo estendido a 2 processos | TRANSCRICAO | [09:30] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-prisma-orm-camada-acesso-dados-config.md | Alternativa Considerada | Compartilhar única instância de PrismaClient entre API e worker, descartada | TRANSCRICAO | [09:29] Diego |
| ADR-002-CONSEQ-01 | docs/adrs/ADR-002-prisma-orm-camada-acesso-dados-config.md | Consequência | Acoplamento forte ao DSL do Prisma; migração de ORM seria custo altíssimo | CODIGO | prisma/schema.prisma |
| ADR-003 | docs/adrs/ADR-003-publicacao-atomica-eventos-outbox-orders.md | Decisão | publishWebhookEvent(tx, order, from, to) inserido na mesma transação de changeStatus | TRANSCRICAO | [09:41] Bruno |
| ADR-003-ALT-01 | docs/adrs/ADR-003-publicacao-atomica-eventos-outbox-orders.md | Alternativa Considerada | Envio síncrono de webhook dentro de changeStatus, descartado | TRANSCRICAO | [09:04] Bruno |
| ADR-003-ALT-02 | docs/adrs/ADR-003-publicacao-atomica-eventos-outbox-orders.md | Alternativa Considerada | Fila externa (Redis Streams) publicada fora da transação, descartada | TRANSCRICAO | [09:07] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-publicacao-atomica-eventos-outbox-orders.md | Alternativa Considerada | Injeção de WebhookRepository completo em OrderService, descartada | TRANSCRICAO | [09:41] Diego |
| ADR-003-CONSEQ-01 | docs/adrs/ADR-003-publicacao-atomica-eventos-outbox-orders.md | Consequência | Transação de changeStatus, já pesada, passa a incluir mais uma escrita | TRANSCRICAO | [09:04] Bruno |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-secret-por-endpoint-webhooks.md | Decisão | HMAC-SHA256 por evento, secret única por endpoint, rotação com grace period 24h | TRANSCRICAO | [09:20] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-autenticacao-hmac-secret-por-endpoint-webhooks.md | Alternativa Considerada | Secret global compartilhada entre endpoints, descartada por incidente real | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONSEQ-01 | docs/adrs/ADR-004-autenticacao-hmac-secret-por-endpoint-webhooks.md | Consequência | Plataforma assume gestão de ciclo de vida de credenciais por cliente | TRANSCRICAO | [09:22] Diego |
| ADR-005 | docs/adrs/ADR-005-garantia-entrega-at-least-once-webhooks.md | Decisão | Entrega at-least-once com dedup client-side via X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-garantia-entrega-at-least-once-webhooks.md | Alternativa Considerada | Garantia exactly-once, descartada por complexidade de coordenação | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONSEQ-01 | docs/adrs/ADR-005-garantia-entrega-at-least-once-webhooks.md | Consequência | Responsabilidade de deduplicação deslocada para o cliente B2B | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-outbox-pattern-webhooks.md | Decisão | Padrão Outbox transacional sobre MySQL existente para eventos de webhook | TRANSCRICAO | [09:08] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-outbox-pattern-webhooks.md | Alternativa Considerada | Disparo síncrono de HTTP na transação de changeStatus, descartado | TRANSCRICAO | [09:04] Bruno |
| ADR-006-ALT-02 | docs/adrs/ADR-006-outbox-pattern-webhooks.md | Alternativa Considerada | Fila externa dedicada (Redis Streams), descartada por overengineering | TRANSCRICAO | [09:07] Diego |
| ADR-006-ALT-03 | docs/adrs/ADR-006-outbox-pattern-webhooks.md | Alternativa Considerada | Trigger de banco para notificar worker reativamente, descartado | TRANSCRICAO | [09:09] Diego |
| ADR-006-CONSEQ-01 | docs/adrs/ADR-006-outbox-pattern-webhooks.md | Consequência | Payload renderizado como snapshot no momento da inserção na outbox | TRANSCRICAO | [09:52] Larissa |
| ADR-007 | docs/adrs/ADR-007-politica-retry-backoff-dlq-webhooks.md | Decisão | Backoff exponencial 5 tentativas (1m/5m/30m/2h/12h) e DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-politica-retry-backoff-dlq-webhooks.md | Alternativa Considerada | Retry indefinido sem teto de tentativas, descartado | TRANSCRICAO | [09:15] Diego |
| ADR-007-ALT-02 | docs/adrs/ADR-007-politica-retry-backoff-dlq-webhooks.md | Alternativa Considerada | 3 tentativas com backoff mais agressivo, descartada | TRANSCRICAO | [09:16] Diego |
| ADR-007-ALT-03 | docs/adrs/ADR-007-politica-retry-backoff-dlq-webhooks.md | Alternativa Considerada | Falha definitiva sinalizada na própria outbox sem DLQ separada, descartada | TRANSCRICAO | [09:18] Diego |
| ADR-007-CONSEQ-01 | docs/adrs/ADR-007-politica-retry-backoff-dlq-webhooks.md | Consequência | Sem notificação automática ao cliente quando evento cai em DLQ | TRANSCRICAO | [09:37] Marcos |
| ADR-008 | docs/adrs/ADR-008-worker-entrega-processo-separado-polling-webhooks.md | Decisão | Worker em processo Node separado (src/worker.ts), polling a cada 2s | TRANSCRICAO | [09:09] Diego |
| ADR-008-ALT-01 | docs/adrs/ADR-008-worker-entrega-processo-separado-polling-webhooks.md | Alternativa Considerada | Mecanismo reativo via trigger de banco de dados, descartado | TRANSCRICAO | [09:09] Diego |
| ADR-008-ALT-02 | docs/adrs/ADR-008-worker-entrega-processo-separado-polling-webhooks.md | Alternativa Considerada | Worker executado dentro do mesmo processo da API, descartado | TRANSCRICAO | [09:11] Diego |
| ADR-008-CONSEQ-01 | docs/adrs/ADR-008-worker-entrega-processo-separado-polling-webhooks.md | Consequência | Garantia de ordering limitada a order_id e apenas em regime single-worker | TRANSCRICAO | [09:12] Diego |
| FDD-OBJ-01 | docs/FDD.md | Objetivo Técnico | Latência de caso comum <10s, pior caso igual ao intervalo de polling (2s) | TRANSCRICAO | [09:09] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo Técnico | Invariante de consistência forte: transação commitada implica evento existente | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-03 | docs/FDD.md | Objetivo Técnico | Eliminar acoplamento síncrono entre disponibilidade externa e changeStatus | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-04 | docs/FDD.md | Objetivo Técnico | Autenticidade/integridade via HMAC-SHA256 com rotação sem downtime | TRANSCRICAO | [09:21] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo Técnico | Entrega at-least-once com dedup via X-Event-Id constante entre tentativas | TRANSCRICAO | [09:25] Diego |
| FDD-OBJ-06 | docs/FDD.md | Objetivo Técnico | Nenhuma falha de entrega descartada silenciosamente; DLQ auditável | TRANSCRICAO | [09:18] Diego |
| FDD-OBJ-07 | docs/FDD.md | Objetivo Técnico | Reaproveitar 100% dos padrões arquiteturais já estabelecidos no projeto | TRANSCRICAO | [09:30] Larissa |
| FDD-ESCOPO-01 | docs/FDD.md | Item Incluso | CRUD completo de webhook com secret devolvida apenas na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-ESCOPO-02 | docs/FDD.md | Item Incluso | Filtragem de interesse por status no momento da inserção do evento | TRANSCRICAO | [09:34] Bruno |
| FDD-ESCOPO-03 | docs/FDD.md | Item Incluso | Consulta de histórico de entregas via GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-ESCOPO-04 | docs/FDD.md | Item Incluso | Endpoint administrativo de replay de DLQ restrito a role ADMIN | TRANSCRICAO | [09:18] Diego |
| FDD-ESCOPO-05 | docs/FDD.md | Item Incluso | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-ESCOPO-06 | docs/FDD.md | Item Incluso | Publicação atômica do evento dentro da transação de changeStatus | TRANSCRICAO | [09:41] Diego |
| FDD-ESCOPO-07 | docs/FDD.md | Item Incluso | Worker de entrega em processo separado com polling a cada 2s | TRANSCRICAO | [09:09] Diego |
| FDD-FORA-01 | docs/FDD.md | Item Fora de Escopo | Webhooks inbound; fluxo estritamente outbound | TRANSCRICAO | [09:02] Sofia |
| FDD-FORA-02 | docs/FDD.md | Item Fora de Escopo | Notificação por e-mail em falhas recorrentes, adiada | TRANSCRICAO | [09:37] Larissa |
| FDD-FORA-03 | docs/FDD.md | Item Fora de Escopo | Rate limiting de envio de webhooks | TRANSCRICAO | [09:39] Diego |
| FDD-FORA-04 | docs/FDD.md | Item Fora de Escopo | Dashboard visual para o cliente gerenciar webhooks | TRANSCRICAO | [09:39] Larissa |
| FDD-FORA-05 | docs/FDD.md | Item Fora de Escopo | Endurecimento futuro de RBAC no CRUD de configuração de webhook | TRANSCRICAO | [09:37] Marcos |
| FDD-FORA-06 | docs/FDD.md | Item Fora de Escopo | Arquivamento/purga de eventos entregues após 30 dias | TRANSCRICAO | [09:08] Diego |
| FDD-FORA-07 | docs/FDD.md | Item Fora de Escopo | Múltiplos workers em paralelo com ordering global | TRANSCRICAO | [09:13] Diego |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo Detalhado | Cadastro de webhook via POST com JWT válido de qualquer role | TRANSCRICAO | [09:31] Marcos |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo Detalhado | PATCH /api/v1/orders/:id/status chega a OrderController.changeStatus | CODIGO | src/modules/orders/order.routes.ts:19-23 |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo Detalhado | OrderController.changeStatus delega a OrderService.changeStatus | CODIGO | src/modules/orders/order.controller.ts:38-46 |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo Detalhado | Transação inteira (status, histórico, estoque, outbox) commitada/revertida atomicamente | TRANSCRICAO | [09:41] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo Detalhado | Worker independente lê outbox em polling, aplica retry/DLQ | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo Detalhado | Cliente consulta deliveries; admin reprocessa via replay de DLQ | TRANSCRICAO | [09:34] Marcos |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo Detalhado | Inserção do evento ocorre exclusivamente dentro de OrderService.changeStatus | CODIGO | src/modules/orders/order.service.ts:131-177 |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo Detalhado | Ponto de inserção proposto após orderStatusHistory.create e antes de refreshed | CODIGO | src/modules/orders/order.service.ts:159-176 |
| FDD-FLUXO-09 | docs/FDD.md | Fluxo Detalhado | publishWebhookEvent é função pura recebendo tx, sem repository completo | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-10 | docs/FDD.md | Fluxo Detalhado | Consulta webhook_config por customerId ativo e toStatus de interesse | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-11 | docs/FDD.md | Fluxo Detalhado | Payload do evento com event_id, event_type, timestamp, order_id, total_cents etc. | TRANSCRICAO | [09:43] Diego |
| FDD-FLUXO-12 | docs/FDD.md | Fluxo Detalhado | Falha na inserção do evento propaga rollback de toda a transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-13 | docs/FDD.md | Fluxo Detalhado | Entry-point dedicado src/worker.ts, script npm run worker (a ser criado) | TRANSCRICAO | [09:11] Diego |
| FDD-FLUXO-14 | docs/FDD.md | Fluxo Detalhado | Worker instancia própria PrismaClient via createPrismaClient() | CODIGO | src/config/database.ts:4-10 |
| FDD-FLUXO-15 | docs/FDD.md | Fluxo Detalhado | Loop de polling a cada 2s buscando eventos pendentes em lote pequeno | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-16 | docs/FDD.md | Fluxo Detalhado | Assinatura HMAC-SHA256 e POST com timeout de 10s por evento | TRANSCRICAO | [09:42] Diego |
| FDD-FLUXO-17 | docs/FDD.md | Fluxo Detalhado | Resposta 2xx marca evento como DELIVERED e grava histórico de entrega | TRANSCRICAO | [09:34] Marcos |
| FDD-FLUXO-18 | docs/FDD.md | Fluxo Detalhado | attemptCount incrementado e nextAttemptAt recalculado a cada falha (backoff) | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-19 | docs/FDD.md | Fluxo Detalhado | Cada tentativa gera linha de histórico consultável, preservando todas as tentativas | TRANSCRICAO | [09:34] Marcos |
| FDD-FLUXO-20 | docs/FDD.md | Fluxo Detalhado | X-Event-Id permanece idêntico em todas as tentativas de reenvio | TRANSCRICAO | [09:25] Diego |
| FDD-FLUXO-21 | docs/FDD.md | Fluxo Detalhado | Ao esgotar a 5ª tentativa, insere linha em webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-22-T | docs/FDD.md | Fluxo Detalhado | Reprocessamento exclusivamente manual via replay, restrito a role ADMIN | TRANSCRICAO | [09:35] Diego |
| FDD-FLUXO-22-C | docs/FDD.md | Fluxo Detalhado | Replay reaproveita requireRole já existente | CODIGO | src/middlewares/auth.middleware.ts:49-61 |
| FDD-FLUXO-23 | docs/FDD.md | Fluxo Detalhado | Replay recria linha em outbox com status PENDING, elegível ao worker | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-24 | docs/FDD.md | Fluxo Detalhado | Toda execução de replay registrada em log de auditoria com id do admin | TRANSCRICAO | [09:36] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato Público | POST /api/v1/webhooks para cadastro de webhook | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato Público | GET /api/v1/webhooks listagem paginada por customerId | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-02-C | docs/FDD.md | Contrato Público | Limite de pageSize máximo de 100, por analogia ao padrão existente | CODIGO | src/modules/orders/order.schemas.ts:25 |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato Público | PATCH /api/v1/webhooks/:id para edição de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato Público | DELETE /api/v1/webhooks/:id para remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04-C | docs/FDD.md | Contrato Público | Resposta 204 No Content sem corpo, mesmo padrão de OrderController.delete | CODIGO | src/modules/orders/order.controller.ts:48-55 |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato Público | GET /api/v1/webhooks/:id/deliveries para histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato Público | Endpoint de rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato Público | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-07-C | docs/FDD.md | Contrato Público | Endpoint de replay exige role ADMIN via requireRole | CODIGO | src/middlewares/auth.middleware.ts:49-61 |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato Público | Limite de corpo de requisição de 1mb já cobre os endpoints propostos | CODIGO | src/app.ts:59 |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND para configuração de webhook não encontrada | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL para URL não HTTPS ou inválida | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED para operação sem secret ativa | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE para payload acima de 64KB | TRANSCRICAO | [09:24] Diego |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED para falha de entrega, timeout ou status não-2xx | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND para id de DLQ inexistente no replay | TRANSCRICAO | [09:18] Diego |
| FDD-RESILIENCIA-01 | docs/FDD.md | Estratégia de Resiliência | Timeout de 10s por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| FDD-RESILIENCIA-02 | docs/FDD.md | Estratégia de Resiliência | 5 tentativas de entrega por evento, incluindo inicial mais 4 reenvios | TRANSCRICAO | [09:15] Diego |
| FDD-RESILIENCIA-03 | docs/FDD.md | Estratégia de Resiliência | Backoff fixo 1m/5m/30m/2h/12h, janela de ~15h | TRANSCRICAO | [09:17] Diego |
| FDD-RESILIENCIA-04 | docs/FDD.md | Estratégia de Resiliência | Fallback para DLQ ao esgotar tentativas; sem fallback automático adicional | TRANSCRICAO | [09:18] Diego |
| FDD-RESILIENCIA-05 | docs/FDD.md | Estratégia de Resiliência | Invariantes críticos: atomicidade, X-Event-Id imutável, ordering, replay ADMIN | TRANSCRICAO | [09:06] Diego |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Projeto não possui biblioteca de métricas instrumentada hoje | CODIGO | package.json |
| FDD-OBS-02 | docs/FDD.md | Requisito Não Funcional | Módulo reaproveita logger Pino já configurado, sem novo mecanismo | CODIGO | src/shared/logger/index.ts:13-30 |
| FDD-OBS-03 | docs/FDD.md | Requisito Não Funcional | Campos estruturados recomendados seguem padrão já usado por requestLogger | CODIGO | src/middlewares/request-logger.middleware.ts:14-24 |
| FDD-OBS-04 | docs/FDD.md | Requisito Não Funcional | Replay administrativo deve ser logado com id do administrador responsável | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-05 | docs/FDD.md | Requisito Não Funcional | Projeto não possui biblioteca de tracing distribuído hoje | CODIGO | package.json |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node.js >=20 já exigido pelo projeto | CODIGO | package.json:7-9 |
| FDD-DEP-02 | docs/FDD.md | Dependência | @prisma/client e prisma 5.22.0 já usados no projeto | CODIGO | package.json:26,48 |
| FDD-DEP-03 | docs/FDD.md | Dependência | MySQL como datasource único do Prisma | CODIGO | prisma/schema.prisma:5-9 |
| FDD-DEP-04 | docs/FDD.md | Dependência | jsonwebtoken 9.0.2 reaproveitado sem alteração | CODIGO | package.json:29 |
| FDD-DEP-05 | docs/FDD.md | Dependência | uuid 11.0.3 já disponível para gerar event_id e IDs de novas entidades | CODIGO | package.json:32 |
| FDD-DEP-06 | docs/FDD.md | Dependência | zod 3.23.8 reaproveitado para novos schemas de validação | CODIGO | package.json:33 |
| FDD-DEP-07 | docs/FDD.md | Dependência | pino/pino-http reaproveitados sem alteração | CODIGO | package.json:30-31 |
| FDD-DEP-08 | docs/FDD.md | Dependência | express 4.21.1 como único framework HTTP do projeto | CODIGO | package.json:28 |
| FDD-DEP-09 | docs/FDD.md | Dependência | Nenhuma rota/schema/comportamento externo dos módulos existentes é alterado | CODIGO | src/routes/index.ts |
| FDD-DEP-10 | docs/FDD.md | Dependência | Contrato de resposta de PATCH /orders/:id/status não muda de formato | CODIGO | src/modules/orders/order.controller.ts:38-46 |
| FDD-DEP-11 | docs/FDD.md | Dependência | Singleton de PrismaClient por processo é estendido, não alterado | CODIGO | src/config/database.ts:4-10 |
| FDD-DEP-12 | docs/FDD.md | Dependência | Convenção de versionamento /api/v1 mantida para novos endpoints | CODIGO | src/app.ts:67 |
| FDD-CA-01 | docs/FDD.md | Critério de Aceitação | Exatamente uma linha em webhook_outbox por transição com webhook interessado | TRANSCRICAO | [09:41] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério de Aceitação | Latência de entrega em caso comum <10s, pior caso 2s | TRANSCRICAO | [09:09] Diego |
| FDD-CA-03 | docs/FDD.md | Critério de Aceitação | Toda chamada de entrega inclui os 4 headers de identificação/assinatura | TRANSCRICAO | [09:44] Diego |
| FDD-CA-04 | docs/FDD.md | Critério de Aceitação | X-Event-Id idêntico em todas as tentativas de reenvio de um evento | TRANSCRICAO | [09:25] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de Aceitação | X-Signature verificável como HMAC-SHA256 do corpo exato do request | TRANSCRICAO | [09:20] Sofia |
| FDD-CA-06 | docs/FDD.md | Critério de Aceitação | Falha de entrega segue exatamente a progressão de backoff da ADR-007 | TRANSCRICAO | [09:17] Diego |
| FDD-CA-07-T | docs/FDD.md | Critério de Aceitação | Endpoint de replay retorna 403 para usuário sem role ADMIN, com log de auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-07-C | docs/FDD.md | Critério de Aceitação | Verificação de role reaproveita requireRole já existente | CODIGO | src/middlewares/auth.middleware.ts:49-61 |
| FDD-CA-08 | docs/FDD.md | Critério de Aceitação | Rotação de secret mantém secret anterior válida por exatamente 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-09 | docs/FDD.md | Critério de Aceitação | Cadastro com URL não HTTPS rejeitado com WEBHOOK_INVALID_URL e 400 | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-10 | docs/FDD.md | Critério de Aceitação | Todo erro do módulo é instância de AppError tratada pelo error.middleware existente | CODIGO | src/middlewares/error.middleware.ts:14-24 |
| FDD-RISCO-01 | docs/FDD.md | Risco | Worker sem mecanismo de supervisão/restart definido | TRANSCRICAO | [09:11] Diego |
| FDD-RISCO-02 | docs/FDD.md | Risco | Aumento de contenção de locks na transação já pesada de changeStatus | TRANSCRICAO | [09:04] Bruno |
| FDD-RISCO-03 | docs/FDD.md | Risco | Comportamento indefinido para evento acima de 64KB dentro da transação | TRANSCRICAO | [09:24] Sofia |
| FDD-RISCO-04 | docs/FDD.md | Risco | Vazamento de secret de webhook comprometendo autenticidade de eventos | TRANSCRICAO | [09:22] Diego |
| FDD-RISCO-05 | docs/FDD.md | Risco | DLQ sem processo formal de monitoramento | TRANSCRICAO | [09:18] Diego |
| FDD-INTEGRACAO-01 | docs/FDD.md | Integração com Sistema Existente | OrderService.changeStatus estendido para invocar publishWebhookEvent | CODIGO | src/modules/orders/order.service.ts:126-179 |
| FDD-INTEGRACAO-02 | docs/FDD.md | Integração com Sistema Existente | Hierarquia AppError/http-errors estendida com novas subclasses WEBHOOK_* | CODIGO | src/shared/errors/http-errors.ts:1-63 |
| FDD-INTEGRACAO-03 | docs/FDD.md | Integração com Sistema Existente | authenticate e requireRole reaproveitados sem alteração nas novas rotas | CODIGO | src/middlewares/auth.middleware.ts:27-61 |
| FDD-INTEGRACAO-04 | docs/FDD.md | Integração com Sistema Existente | Novo processo worker reaproveita createPrismaClient() | CODIGO | src/config/database.ts:1-10 |
| FDD-INTEGRACAO-05 | docs/FDD.md | Integração com Sistema Existente | error.middleware.ts trata genericamente qualquer subclasse WEBHOOK_* de AppError | CODIGO | src/middlewares/error.middleware.ts:14-24 |
| FDD-INTEGRACAO-06 | docs/FDD.md | Integração com Sistema Existente | Controllers e buildControllers/buildApiRouter estendidos com entrada webhooks | CODIGO | src/routes/index.ts:13-31 |
| FDD-INTEGRACAO-07 | docs/FDD.md | Integração com Sistema Existente | Logger Pino reaproveitado; redactPaths como ponto de extensão recomendado | CODIGO | src/shared/logger/index.ts:4-11 |
| FDD-INTEGRACAO-08 | docs/FDD.md | Integração com Sistema Existente | src/server.ts serve de referência estrutural direta para src/worker.ts | CODIGO | src/server.ts:1-27 |

---

## Itens Sem Rastreabilidade Direta

Itens que os documentos-fonte já registram como hipótese, suposição ou questão em aberto, sem evidência real disponível em `TRANSCRICAO.md` ou no código. Não entram no denominador de cobertura.

| ID | Documento | Tipo | Conteúdo (resumo) | Motivo |
| --- | --- | --- | --- | --- |
| PRD-OBJ-05 | docs/PRD.md | Objetivo/Métrica | Meta numérica hipotética de taxa de sucesso de entrega (ex.: 95%) | O próprio PRD marca como "Hipótese (meta não sustentada explicitamente pelas fontes)" |
| PRD-NFR-LACUNA-01 | docs/PRD.md | Requisito Não Funcional | Meta de disponibilidade de 99,5% mensal para o worker | PRD marca explicitamente como "Hipótese (não quantificada nas fontes)" |
| PRD-NFR-LACUNA-02 | docs/PRD.md | Requisito Não Funcional | Política de armazenamento da secret em repouso (texto plano/hash/KMS) | PRD marca como "Lacuna explícita nas fontes"; ADR-004 confirma ausência de decisão |
| PRD-NFR-LACUNA-03 | docs/PRD.md | Requisito Não Funcional | Fluxo de revogação de emergência de secret durante o grace period | PRD marca como "Lacuna explícita nas fontes"; ADR-004 confirma ausência de decisão |
| PRD-TESTE-LACUNA-01 | docs/PRD.md | Estratégia de Testes | Validação funcional simulando o fluxo completo de um cliente real antes do go-live | PRD marca como "Hipótese (não sustentada por TRANSCRICAO.md...)"; a reunião não registra decisão sobre essa etapa formal, apenas os testes de HMAC, retry/DLQ e permissão |
| PRD-NFR-LACUNA-04 | docs/PRD.md | Requisito Não Funcional | Métricas numéricas (taxa de sucesso, latência p95, volume/hora) | PRD marca como "Lacuna explícita nas fontes" |
| PRD-NFR-LACUNA-05 | docs/PRD.md | Requisito Não Funcional | Requisitos regulatórios/compliance específicos (ex.: LGPD) | PRD marca como "Lacuna: não há... menção... em nenhuma das cinco fontes" |
| RFC-RNF-LACUNA-01 | docs/RFC.md | Requisito Não Funcional | Volume de eventos, número de clientes simultâneos e throughput-alvo | RFC marca explicitamente como "TBD... nenhuma métrica ou projeção numérica está disponível" |
| RFC-RESTR-LACUNA-01 | docs/RFC.md | Restrição | Requisitos regulatórios/compliance (ex.: LGPD) | RFC marca como "TBD: não mencionados em nenhuma fonte disponível" |
| RFC-RESTR-LACUNA-02 | docs/RFC.md | Restrição | Limitações de orçamento de infraestrutura | RFC marca como "TBD: não quantificadas nas fontes" |
| RFC-QA-LACUNA-01 | docs/RFC.md | Questão em Aberto | Armazenamento da secret em repouso (texto plano, hash ou KMS) | Não discutido na reunião; lacuna explícita registrada apenas pela ADR-004, sem evidência real em TRANSCRICAO ou código |
| RFC-QA-LACUNA-02 | docs/RFC.md | Questão em Aberto | Fluxo de revogação de emergência de secret comprometida no grace period | Não discutido na reunião; lacuna explícita da ADR-004, sem evidência real |
| RFC-QA-LACUNA-03 | docs/RFC.md | Questão em Aberto | Processo/responsável formal de monitoramento periódico da DLQ | Não discutido na reunião; lacuna explícita da ADR-007, sem evidência real |
| RFC-QA-LACUNA-04 | docs/RFC.md | Questão em Aberto | Política de retenção de registros em DLQ já reprocessados | RFC marca como "não discutida na reunião nem coberta pelo código" |
| RFC-QA-LACUNA-05 | docs/RFC.md | Questão em Aberto | Mecanismo de supervisão/restart do processo worker em caso de crash | Não fechado na reunião; ADR-008 registra como "[PRECISA DE INFORMAÇÃO]", sem evidência real |
| RFC-QA-LACUNA-06 | docs/RFC.md | Questão em Aberto | Estratégia formal de rollout gradual e plano de rollback em produção | RFC marca como "não mencionados em nenhum momento da reunião nem cobertos pelas ADRs" |
| RFC-QA-LACUNA-07 | docs/RFC.md | Questão em Aberto | Volume de tráfego esperado e throughput-alvo do worker sob carga real | RFC marca como "TBD... nenhuma métrica ou projeção numérica está disponível" |
| ADR-001-ALT-LACUNA-01 | docs/adrs/ADR-001-estrategia-autenticacao-jwt-stateless-auth.md | Alternativa Considerada | Sessão server-side com store compartilhado (ex.: Redis) | ADR marca com "[NEEDS INPUT: não há menção na transcrição... inferência técnica plausível, não debatida em reunião]" |
| ADR-001-ALT-LACUNA-02 | docs/adrs/ADR-001-estrategia-autenticacao-jwt-stateless-auth.md | Alternativa Considerada | Provedor de identidade externo (IdP), ex. Auth0/Cognito | ADR marca com o mesmo "[NEEDS INPUT]" de ausência de debate real |
| ADR-002-ALT-LACUNA-01 | docs/adrs/ADR-002-prisma-orm-camada-acesso-dados-config.md | Alternativa Considerada | Adotar ORM/query builder alternativo (TypeORM, Sequelize, Drizzle, Knex) | ADR marca com "[NEEDS INPUT: nenhuma alternativa de ORM é mencionada em TRANSCRICAO.md]" |
| ADR-002-ALT-LACUNA-02 | docs/adrs/ADR-002-prisma-orm-camada-acesso-dados-config.md | Alternativa Considerada | Gerenciar singleton via container de DI (Awilix, InversifyJS) | ADR marca que "nenhuma fonte disponível indica que essa alternativa tenha sido de fato discutida" |
| ADR-004-ALT-LACUNA-01 | docs/adrs/ADR-004-autenticacao-hmac-secret-por-endpoint-webhooks.md | Alternativa Considerada | Assinatura assimétrica (RSA/ECDSA) em vez de HMAC simétrico | ADR marca com "[NEEDS INPUT: não há evidência de que tenha sido de fato avaliada e descartada]" |
| ADR-004-ALT-LACUNA-02 | docs/adrs/ADR-004-autenticacao-hmac-secret-por-endpoint-webhooks.md | Alternativa Considerada | Rotação com invalidação imediata da secret antiga (sem grace period) | ADR marca com "[NEEDS INPUT: validar se foi de fato avaliada e descartada]" |
| ADR-004-CONSEQ-LACUNA-01 | docs/adrs/ADR-004-autenticacao-hmac-secret-por-endpoint-webhooks.md | Consequência | Ausência de procedimento de revogação de emergência de secret comprometida | ADR marca com "[NEEDS INPUT: falta definição de fluxo de revogação imediata]" |
| ADR-007-CONSEQ-LACUNA-01 | docs/adrs/ADR-007-politica-retry-backoff-dlq-webhooks.md | Consequência | Ausência de processo/responsável formal de monitoramento da DLQ | ADR marca com "[NEEDS INPUT: não há definição, nem na transcrição nem no código]" |
| ADR-007-CONSEQ-LACUNA-02 | docs/adrs/ADR-007-politica-retry-backoff-dlq-webhooks.md | Consequência | Ausência de política de retenção de registros de DLQ já reprocessados | ADR marca com "[NEEDS INPUT: não foi discutido na reunião nem há evidência no código]" |
| ADR-008-CONSEQ-LACUNA-01 | docs/adrs/ADR-008-worker-entrega-processo-separado-polling-webhooks.md | Consequência | Ausência de mecanismo de supervisão/restart do worker em caso de crash | ADR marca com "[PRECISA DE INFORMAÇÃO: não há definição, nem na transcrição nem no código]" |
| FDD-FORA-LACUNA-01 | docs/FDD.md | Item Fora de Escopo | Mecanismo formal de supervisão/restart do processo worker | FDD marca como "reconhecido como necessidade... mas sem definição operacional nas fontes (ADR-008, [PRECISA DE INFORMAÇÃO])" |
| FDD-FORA-LACUNA-02 | docs/FDD.md | Item Fora de Escopo | Política de armazenamento da secret em repouso e revogação de emergência | FDD marca como "lacunas explícitas da ADR-004, não resolvidas por nenhuma fonte disponível" |
| FDD-FLUXO-LACUNA-01 | docs/FDD.md | Fluxo Detalhado | Mecanismo de leitura concorrente-segura da outbox (ex.: SELECT FOR UPDATE SKIP LOCKED) | FDD trata explicitamente "como hipótese com três opções tecnicamente plausíveis" |
| FDD-FLUXO-LACUNA-02 | docs/FDD.md | Fluxo Detalhado | Tratamento do registro de origem em webhook_outbox após envio à DLQ | FDD trata como "hipótese com duas opções plausíveis" |
| FDD-FLUXO-LACUNA-03 | docs/FDD.md | Fluxo Detalhado | Tamanho de lote (batch) de leitura do worker por ciclo de polling | FDD marca como "TBD, sem número fechado na reunião"; oferece "hipótese: 10, 50 ou 100" |
| FDD-ERRO-LACUNA-01 | docs/FDD.md | Erro | WEBHOOK_SECRET_ROTATION_CONFLICT para rotação sobreposta ao grace period | FDD marca explicitamente como "hipótese; comportamento exato não definido pela ADR-004" |
| FDD-ERRO-LACUNA-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_STATUS_FILTER para status fora do enum OrderStatus | FDD marca explicitamente como "hipótese, seguindo o padrão já usado" |
| FDD-RESILIENCIA-LACUNA-01 | docs/FDD.md | Estratégia de Resiliência | Mecanismo de supervisão/restart do worker em caso de crash | FDD marca como lacuna herdada da ADR-008, "[PRECISA DE INFORMAÇÃO]" |
| FDD-RESILIENCIA-LACUNA-02 | docs/FDD.md | Estratégia de Resiliência | Fallback de circuito (circuit breaker) para cliente cronicamente indisponível | FDD declara "não foi discutido em nenhuma fonte; não está incluído nesta entrega" |
| FDD-OBS-LACUNA-01 | docs/FDD.md | Requisito Não Funcional | Métricas de negócio (taxa de sucesso, latência p95, volume/hora) | FDD cita a própria RFC declarando "TBD" |
| FDD-OBS-LACUNA-02 | docs/FDD.md | Requisito Não Funcional | Indicadores agregados sobre webhook_outbox/webhook_dead_letter como recomendação | FDD marca como "recomendação técnica deste FDD, não uma decisão já tomada pelas fontes" |
| FDD-OBS-LACUNA-03 | docs/FDD.md | Requisito Não Funcional | Extensão de redactPaths do logger para incluir `*.secret` | FDD marca como "proposta por analogia direta... não está decidida por nenhuma fonte disponível" |
| FDD-OBS-LACUNA-04 | docs/FDD.md | Requisito Não Funcional | Dashboards e alertas mínimos sugeridos (crescimento de DLQ, inatividade do worker) | FDD marca como "propostas deste FDD, não decisões já fechadas pelas fontes" |
| FDD-DEP-LACUNA-01 | docs/FDD.md | Dependência | Uso do módulo nativo `crypto` do Node para HMAC-SHA256 | FDD declara que "essa via não foi especificada explicitamente pela equipe... é uma inferência deste FDD" |
| FDD-CA-LACUNA-01 | docs/FDD.md | Critério de Aceitação | Critério de aceite de performance sob carga (volume/throughput) | FDD marca como "TBD, não incluído nesta lista até definição futura" |

---

## Divergências Identificadas

Itens em que o conteúdo do documento não corresponde ao que foi efetivamente encontrado em `TRANSCRICAO.md` ou no código.

| ID | Documento | O que o documento afirma | O que as fontes mostram |
| --- | --- | --- | --- |

Nenhuma divergência identificada entre os documentos e as fontes analisadas. A única divergência apontada na versão anterior desta análise (`docs/PRD.md`, item de "validação funcional guiada por cenário real de cliente" na Estratégia de Testes, sem lastro em `TRANSCRICAO.md`) foi corrigida no PRD: o item passou a ser apresentado explicitamente como hipótese/sugestão do próprio documento (não mais como parte decidida da estratégia), e agora consta em "Itens Sem Rastreabilidade Direta" (`PRD-TESTE-LACUNA-01`).

Os demais itens verificados nos quatro documentos (PRD, RFC, FDD, ADR-001 a ADR-008) mantiveram correspondência fiel com os trechos de `TRANSCRICAO.md` citados e com os caminhos de código verificados por leitura direta (`src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/modules/orders/order.routes.ts`, `src/modules/orders/order.controller.ts`, `src/modules/orders/order.schemas.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`, `src/config/database.ts`, `src/config/env.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`, `src/modules/auth/auth.service.ts`, `src/modules/auth/auth.schemas.ts`, `prisma/schema.prisma`, `package.json`, `tests/setup.ts`). As referências a números de linha citadas nos quatro documentos e nas ADRs foram conferidas linha a linha e correspondem ao conteúdo real dos arquivos, com pequenas variações de +/-1 a 2 linhas em alguns casos (ex.: `order.service.ts:131-177` vs. bloco real 131-178), consideradas dentro da margem normal de precisão de citação e não tratadas como divergência.

---

## Notas sobre a cobertura

Foram inventariados 339 itens ao todo nos quatro documentos (297 na tabela principal, 42 em "Itens Sem Rastreabilidade Direta", 0 em "Divergências Identificadas"). Os 42 itens não rastreados correspondem, em todos os casos, a lacunas que os próprios documentos-fonte (PRD, RFC, FDD ou as ADRs) já assumem explicitamente como hipótese, "TBD", "QUESTÃO EM ABERTO" ou `[NEEDS INPUT]`/`[PRECISA DE INFORMAÇÃO]` — nenhum deles foi forçado na tabela principal, e por isso são excluídos do denominador de cobertura. Descontando esses 42 itens, a cobertura é de 297 de 297 itens (100%), muito acima do mínimo de 80% exigido. A divergência originalmente identificada nesta análise (uma recomendação de teste do PRD apresentada como decidida, sem lastro direto na transcrição) foi corrigida no PRD e reclassificada como hipótese explícita (`PRD-TESTE-LACUNA-01`). Esse resultado não compromete a rastreabilidade geral do pacote de documentos, que se mostrou consistentemente fiel a `TRANSCRICAO.md` e ao código-fonte real do projeto.
