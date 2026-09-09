# Mapeamento da Arquitetura da Base de Código

## Visão Geral do Projeto

- **Nome**: `order-management-api` (Order Management System — OMS)
- **Propósito**: API REST para gestão de pedidos B2B — clientes, produtos, pedidos com máquina de estados, controle transacional de estoque e auditoria de mudanças de status.
- **Tipo**: Backend HTTP monolítico modular (não há frontend no repositório).
- **Linguagem**: TypeScript 5.6 (ESM puro, `"type": "module"`, `target: ES2022`, `strict: true`, `noUncheckedIndexedAccess`).
- **Framework principal**: Express 4.21.
- **Estado atual da feature em análise**: o repositório **não contém nenhuma implementação de webhooks**. Este mapeamento cobre a arquitetura existente e serve de base para a Fase 2, que vai identificar decisões (algumas já tomadas em reunião, ainda não codificadas) relevantes para o novo módulo de Webhooks de Notificação de Pedidos.

## Stack Tecnológica

| Camada | Tecnologia | Versão | Observação |
| --- | --- | --- | --- |
| Runtime | Node.js | ≥20 (`engines`) | ESM nativo via `tsx --env-file` |
| Linguagem | TypeScript | 5.6.3 | `moduleResolution: Bundler`, sem `any` implícito |
| Framework HTTP | Express | 4.21.1 | Roteamento manual, sem DI container |
| ORM | Prisma Client | 5.22.0 | `PrismaClient` único por processo, provider MySQL |
| Banco de dados | MySQL | 8.0 (`docker-compose.yml`) | `utf8mb4`, `mysql_native_password` |
| Validação | Zod | 3.23.8 | Schemas por módulo, aplicados via middleware `validate()` |
| Autenticação | jsonwebtoken + bcrypt | 9.0.2 / 5.1.1 | JWT stateless, hash de senha bcrypt |
| Logging | Pino + pino-http | 9.5.0 / 10.3.0 | Logger estruturado, redaction de campos sensíveis |
| Testes | Vitest + Supertest | 2.1.4 / 7.0.0 | Rodam contra banco MySQL real (não mockado) |
| Lint/Format | ESLint + Prettier | 8.57.1 / 3.3.3 | `@typescript-eslint`, `eslint-config-prettier` |
| IDs | `uuid` (v4) | 11.0.3 | Todas as PKs do schema usam UUID (`@db.Char(36)`) |

Não há fila de mensagens, cache, nem serviços de nuvem configurados. A única infraestrutura externa é o MySQL via `docker-compose.yml`.

## Notas de Contexto e Transcrição

**Arquivos de Origem**: `TRANSCRICAO.md` (reunião técnica de ~55min, 5 participantes: Larissa/Tech Lead, Marcos/PM, Bruno/Eng. Pleno, Diego/Eng. Sênior Plataforma, Sofia/Eng. Segurança).

**Contexto de negócio da reunião**: três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram notificação de mudança de status de pedido em até 10s ("tempo real" para eles), hoje fazem polling em `GET /orders`. A feature decidida é **outbound webhook** (só a plataforma envia, não recebe).

**Padrões arquiteturais mencionados na reunião**:
- Padrão **Outbox** no MySQL existente (tabela `webhook_outbox`), evitando subir infraestrutura nova (Redis Streams foi cogitado e descartado).
- **Worker separado** em processo próprio (`src/worker.ts`), com **polling** a cada 2 segundos (MySQL não tem `LISTEN/NOTIFY` como Postgres).
- **Retry com backoff exponencial** (1m/5m/30m/2h/12h, 5 tentativas) + **Dead Letter Queue** em tabela própria (`webhook_dead_letter`).
- **HMAC-SHA256** com secret por endpoint (não global), suportando rotação com grace period de 24h.
- **At-least-once delivery** com deduplicação client-side via header `X-Event-Id` (UUID).
- Reuso extensivo dos padrões já existentes no código (ver seção "Discrepâncias" abaixo — nenhuma discrepância, é alinhamento explícito).

**Decisões chave extraídas da reunião** (classificação de evidência aplicada a cada uma):

| # | Decisão | Classificação | Timestamp |
| --- | --- | --- | --- |
| 1 | Padrão Outbox no MySQL (não síncrono, não Redis Streams) | **Confirmada** | `[09:06]-[09:08]` Diego/Larissa |
| 2 | Worker em processo separado, `src/worker.ts`, `npm run worker` | **Confirmada** | `[09:11]` Diego/Larissa |
| 3 | Polling a cada 2 segundos | **Confirmada** | `[09:09]-[09:10]` Diego/Marcos/Larissa |
| 4 | Ordering apenas por `order_id`, sem garantia global, single-worker | **Confirmada** (como limitação documentada) | `[09:12]-[09:13]` |
| 5 | Retry: 5 tentativas, backoff 1m/5m/30m/2h/12h | **Confirmada** | `[09:15]-[09:17]` Diego/Larissa |
| 6 | DLQ em tabela separada `webhook_dead_letter` | **Confirmada** | `[09:18]` Diego |
| 7 | Endpoint admin de replay manual `POST /admin/webhooks/dead-letter/:id/replay`, role `ADMIN` obrigatória | **Confirmada** | `[09:18]-[09:19]`, `[09:35]-[09:36]` |
| 8 | HMAC-SHA256, secret por endpoint, header `X-Signature` | **Confirmada** | `[09:20]-[09:22]` Sofia |
| 9 | Rotação de secret com grace period de 24h | **Confirmada** | `[09:21]` Sofia |
| 10 | TLS obrigatório (URL https) | **Confirmada**, mas classificada pela própria equipe como validação de schema, não decisão arquitetural | `[09:23]` Sofia |
| 11 | Limite de payload 64KB, erro se ultrapassar (sem truncar) | **Confirmada**, classificada como requisito não funcional, não ADR separado | `[09:23]-[09:24]` |
| 12 | At-least-once + dedup via `X-Event-Id` | **Confirmada** | `[09:24]-[09:26]` Diego/Sofia |
| 13 | Novo módulo `src/modules/webhooks` seguindo padrão routes/controller/service/repository/schemas existente | **Confirmada** | `[09:27]-[09:28]` Bruno/Diego |
| 14 | Reuso de `AppError`, códigos `WEBHOOK_*`, Pino, error middleware centralizado | **Confirmada** | `[09:28]-[09:30]` Bruno |
| 15 | Worker usa `PrismaClient` separado (novo processo), mesma `DATABASE_URL` | **Confirmada** | `[09:29]-[09:30]` Bruno/Diego |
| 16 | Filtro de eventos por webhook (lista de status), aplicado na **inserção** na outbox, não no envio | **Confirmada** | `[09:33]-[09:34]` |
| 17 | Endpoint `GET /webhooks/:id/deliveries` (histórico de entregas) | **Confirmada** | `[09:34]` Marcos |
| 18 | CRUD de configuração de webhook: qualquer role autenticada (não exige ADMIN) | **Confirmada** ("por enquanto") | `[09:36]-[09:37]` |
| 19 | Inserção na outbox dentro da **mesma transação** de `changeStatus` (atomicidade) | **Confirmada**, crítica | `[09:40]-[09:41]` Bruno/Diego |
| 20 | Função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` client | **Confirmada** (proposta técnica concreta) | `[09:41]` Bruno/Diego |
| 21 | Timeout do HTTP call do worker: 10 segundos | **Confirmada** | `[09:42]` Diego/Sofia |
| 22 | Payload enxuto (sem `items`), campos: `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` | **Confirmada** | `[09:43]` Diego |
| 23 | Headers: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` | **Confirmada** | `[09:44]-[09:45]` |
| 24 | IDs da outbox em UUID (não auto-incremento), consistente com o restante do schema | **Confirmada** | `[09:50]-[09:51]` Larissa/Diego |
| 25 | Payload do evento é **snapshot renderizado na inserção** (não recalculado no envio) | **Confirmada** | `[09:51]-[09:52]` Larissa/Bruno/Diego |
| 26 | Prazo estimado: 3 sprints, incluindo revisão de segurança da Sofia | **Confirmada** (planejamento, não arquitetural) | `[09:45]-[09:46]` |

**Propostas rejeitadas ou adiadas** (não viram requisito):
- Envio **síncrono** dentro do `OrderService`, sem outbox — **rejeitada** (`[09:03]-[09:05]`, risco de travar a transação e impossibilidade de rollback em cliente offline).
- **Redis Streams** (ou fila externa) em vez de outbox no MySQL — **rejeitada**, considerada overengineering para o tamanho do time (`[09:07]`).
- **Trigger de banco** para notificar o worker de forma reativa — **rejeitada**, MySQL não tem `LISTEN/NOTIFY` (`[09:09]`).
- **Retry indefinido** sem teto de tentativas — **rejeitada** em favor de 5 tentativas fixas (`[09:15]`).
- **3 tentativas de retry** (mais agressivo) — **rejeitada**, considerada insuficiente para cobrir indisponibilidades de clientes de algumas horas (`[09:15]-[09:16]`).
- **Notificação por e-mail em caso de falhas repetidas** — **adiada** para fase futura (`[09:37]-[09:38]`).
- **Rate limiting de envio** para clientes com muitos eventos simultâneos — **discussão inconclusiva**, registrada como "observar e decidir depois" (`[09:38]-[09:39]`).
- **Dashboard visual** para o cliente acompanhar webhooks — **rejeitada** para este escopo, fica com o time de frontend (`[09:39]-[09:40]`).
- **Garantia de ordering global** entre múltiplos workers — **adiada/não decidida**; solução futura cogitada (particionamento por `order_id` ou lock pessimista) mas não especificada (`[09:12]-[09:13]`).

**Limites de módulos documentados vs. código**: a transcrição confirma exatamente o padrão de módulos já observado no código (`controller/service/repository/routes/schemas`), sem discrepância. Bruno (autor do código de `orders`) é quem propõe a estrutura do módulo `webhooks` na reunião, citando os mesmos artefatos (`AppError`, `requireRole`, error middleware, Pino) que já existem hoje.

**Discrepâncias entre discussão e implementação**: nenhuma. A reunião é anterior à implementação — o código de webhooks ainda não existe. A "discrepância" relevante é apenas a lacuna esperada: nada do que foi decidido está implementado ainda (outbox, worker, endpoints de webhook, tabela DLQ).

## Módulos do Sistema

### Índice de Módulos
1. **AUTH** - Autenticação: login/registro, emissão e verificação de JWT.
2. **USERS** - Gestão de usuários internos (operadores/admins) e RBAC.
3. **CUSTOMERS** - Cadastro de clientes B2B.
4. **PRODUCTS** - Catálogo de produtos e controle de estoque.
5. **ORDERS** - Pedidos: criação, máquina de estados, transação de mudança de status, histórico de auditoria.
6. **SHARED** - Infraestrutura transversal: erros (`AppError`), HTTP (paginação), logger.
7. **MIDDLEWARES** - Autenticação/autorização, validação Zod, logging de requisição, tratamento central de erros.
8. **CONFIG** - Carregamento/validação de env vars, cliente Prisma singleton.
9. **WEBHOOKS** (não implementado) - Módulo alvo da feature discutida na reunião; ainda inexistente no código, mapeado aqui apenas como lacuna a ser preenchida.

---

### AUTH: Autenticação
**Objetivo**: Login e registro de usuários, emissão de JWT stateless.
**Localização**: `src/modules/auth/*`
**Componentes Principais**: `AuthService` (login/register, assinatura JWT via `jsonwebtoken`, verificação de senha via `bcrypt`), `AuthController`, `auth.routes.ts` (`POST /register`, `POST /login`, `GET /me`).
**Tecnologias**: `jsonwebtoken`, `bcrypt`, Zod (`auth.schemas.ts`).
**Dependências**: Internas — `UserRepository`, `UserService` (delega criação de usuário). Externas — nenhuma.
**Padrões**: Token JWT com payload `{ sub, email, role }`, expiração configurável (`JWT_EXPIRES_IN`), stateless (sem refresh token nem blacklist).
**Arquivos Principais**: `src/modules/auth/auth.service.ts`, `src/modules/auth/auth.routes.ts`.
**Escopo**: Pequeno — 4 arquivos.

### USERS: Usuários
**Objetivo**: CRUD de usuários internos, com papéis `ADMIN` / `OPERATOR`.
**Localização**: `src/modules/users/*`
**Componentes Principais**: `UserRepository`, `UserService` (inclui `toPublic()` para nunca vazar `passwordHash`), `UserController`.
**Tecnologias**: Prisma, bcrypt (hash na criação), Zod.
**Dependências**: Consumido por `AuthService` e por `Order.createdBy` / `OrderStatusHistory.changedBy`.
**Padrões**: Repository pattern; enum `UserRole` (`ADMIN`, `OPERATOR`) do Prisma reaproveitado no `AuthUser` de `auth.middleware.ts`.
**Arquivos Principais**: `src/modules/users/user.service.ts`, `src/modules/users/user.repository.ts`.
**Escopo**: Pequeno — 5 arquivos.

### CUSTOMERS: Clientes
**Objetivo**: Cadastro de clientes B2B (nome, e-mail, telefone, documento, endereço).
**Localização**: `src/modules/customers/*`
**Componentes Principais**: `CustomerRepository`, `CustomerService`, `CustomerController`, rotas CRUD padrão.
**Tecnologias**: Prisma (`Customer` model, `address` como `Json`), Zod.
**Dependências**: Referenciado por `Order.customerId`. É o ponto de integração de negócio mais provável para a entidade "cliente" que vai possuir configurações de webhook (a reunião não decide se webhook pertence a `Customer` ou a um novo conceito — usa `customer_id` como campo, `[09:31]-[09:32]`).
**Padrões**: Mesmo padrão modular; índice único em `document`.
**Arquivos Principais**: `src/modules/customers/customer.service.ts`.
**Escopo**: Pequeno — 5 arquivos.

### PRODUCTS: Produtos
**Objetivo**: Catálogo de produtos, preço em centavos, controle de `stockQuantity`, flag `active`.
**Localização**: `src/modules/products/*`
**Componentes Principais**: `ProductRepository`, `ProductService`, `ProductController`.
**Tecnologias**: Prisma, Zod.
**Dependências**: Consumido diretamente por `OrderService` (`debitStock`/`replenishStock` operam em `tx.product.update` com `increment`/`decrement`).
**Padrões**: Mesmo padrão modular; preços e totais sempre em centavos (inteiros), nunca float.
**Arquivos Principais**: `src/modules/products/product.service.ts`.
**Escopo**: Pequeno — 5 arquivos.

### ORDERS: Pedidos (núcleo do domínio)
**Objetivo**: Criação de pedidos, máquina de estados de status, transação atômica de mudança de status com efeitos colaterais de estoque, histórico de auditoria.
**Localização**: `src/modules/orders/*`
**Componentes Principais**:
- `order.status.ts` — máquina de estados pura: `canTransition`, `allowedTransitions`, `isTerminal`, `shouldDebitStock`, `shouldReplenishStock`. Transições válidas: `PENDING→{PAID,CANCELLED}`, `PAID→{PROCESSING,CANCELLED}`, `PROCESSING→{SHIPPED,CANCELLED}`, `SHIPPED→{DELIVERED}`; `DELIVERED`/`CANCELLED` são terminais.
- `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) — **é o ponto de integração central da feature de webhooks**. Dentro de um único `prisma.$transaction`: valida transição (`canTransition`), debita/repõe estoque, atualiza `Order.status`, insere `OrderStatusHistory`. A reunião decide (`[09:40]-[09:42]`) que a inserção na `webhook_outbox` deve ocorrer **dentro dessa mesma transação**, via uma função proposta `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
- `OrderService.create` — transação separada: valida cliente/produtos ativos, calcula subtotal/desconto/total, reserva `orderNumber` sequencial via `OrderNumberSequence.upsert`.
**Tecnologias**: Prisma `$transaction` (transaction client `Prisma.TransactionClient`), Zod.
**Dependências**: Internas — `CustomerRepository`/`ProductRepository` (leitura dentro da transação via `tx`), classes de erro de `shared/errors`. Nenhuma dependência externa hoje (é exatamente o vácuo que a feature de webhooks preenche).
**Padrões**: Máquina de estados explícita e centralizada (não espalhada em ifs); todas as mutações de estoque/status ocorrem dentro de transação; histórico append-only (`OrderStatusHistory`, nunca UPDATE/DELETE).
**Arquivos Principais**: `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`.
**Escopo**: Médio — 6 arquivos, mas concentra a lógica de negócio mais crítica do sistema.

### SHARED: Infraestrutura Compartilhada
**Objetivo**: Contratos reutilizados por todos os módulos.
**Localização**: `src/shared/*`
**Componentes Principais**:
- `shared/errors/app-error.ts` + `http-errors.ts` — hierarquia `AppError` (statusCode + errorCode string + details opcionais). Subclasses: `BadRequestError`, `ValidationError` (400), `UnauthorizedError` (401), `ForbiddenError` (403), `NotFoundError` (404), `ConflictError`/`InvalidStatusTransitionError` (409), `UnprocessableEntityError`/`InsufficientStockError` (422). Este é o padrão que a reunião decide replicar com prefixo `WEBHOOK_*` (`[09:28]`).
- `shared/http/response.ts` — `paginated()`/`buildPagination()`, formato padrão `{ data, pagination: { page, pageSize, total, totalPages } }`.
- `shared/logger/index.ts` — Pino configurado com `redact` (nunca loga senha/token), `pino-pretty` em dev.
**Tecnologias**: Pino, tipos TS puros.
**Dependências**: Consumido por todos os módulos e middlewares.
**Padrões**: Erro tipado com `errorCode` string estável (contrato de API), nunca `throw` de erro genérico dentro de services.
**Arquivos Principais**: `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`.
**Escopo**: Pequeno — 5 arquivos, mas é a espinha dorsal de consistência da API.

### MIDDLEWARES: Middlewares Transversais
**Objetivo**: Autenticação/autorização, validação de entrada, logging por requisição, tratamento central de erros.
**Localização**: `src/middlewares/*`
**Componentes Principais**:
- `auth.middleware.ts` — `authenticate` (verifica JWT via `jwt.verify`, popula `req.user: AuthUser`) e `requireRole(...roles)` (checa `req.user.role` contra lista). A reunião decide reaproveitar `requireRole('ADMIN')` no endpoint de replay de DLQ (`[09:36]`).
- `validate.middleware.ts` — `validate({ body?, query?, params? })`, aplica schemas Zod e converte `ZodError` em `ValidationError`.
- `request-logger.middleware.ts` — gera/propaga `X-Request-Id`, loga `http_request` estruturado no `finish` da resposta (método, path, status, duração, `userId`).
- `error.middleware.ts` — único ponto que serializa qualquer erro (`AppError`, `ZodError`, `Prisma.PrismaClientKnownRequestError` P2002/P2025, ou desconhecido) em JSON `{ error: { code, message, details? } }`. **Não precisa de nenhuma mudança para suportar erros `WEBHOOK_*`**, pois já trata qualquer `AppError` genericamente (confirmado em `[09:29]`: "Vai pegar nossos erros sem precisar mudar nada").
**Tecnologias**: Express `RequestHandler`/`ErrorRequestHandler`, `jsonwebtoken`, Zod, `uuid`.
**Dependências**: Usado por `routes/index.ts` e `app.ts`.
**Padrões**: Middleware chain padrão Express; nenhum middleware específico de módulo — tudo compartilhado.
**Arquivos Principais**: `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`.
**Escopo**: Pequeno — 4 arquivos.

### CONFIG: Configuração
**Objetivo**: Validação de variáveis de ambiente e instância única do Prisma Client.
**Localização**: `src/config/*`
**Componentes Principais**: `env.ts` (schema Zod para env vars, `process.exit(1)` em config inválida), `database.ts` (`createPrismaClient()`, exporta singleton `prisma`).
**Tecnologias**: Zod, `@prisma/client`.
**Dependências**: Usado por `server.ts`, `app.ts`, todos os módulos (via injeção manual do `PrismaClient` em `buildControllers`).
**Padrões**: Fail-fast na inicialização se env inválida; um único `PrismaClient` por processo (relevante para a decisão do worker precisar de sua **própria** instância, `[09:29]-[09:30]`).
**Arquivos Principais**: `src/config/env.ts`, `src/config/database.ts`.
**Escopo**: Pequeno — 2 arquivos.

### WEBHOOKS (não implementado — lacuna mapeada)
**Objetivo** (conforme decidido na reunião, não no código): CRUD de configuração de webhook por cliente, outbox de eventos de mudança de status, worker de entrega com retry/DLQ, HMAC, histórico de entregas.
**Localização esperada** (proposta na reunião, `[09:27]-[09:28]`): `src/modules/webhooks/*` seguindo o padrão `controller/service/repository/routes/schemas` + `webhook.worker.ts` (ou `webhook.processor.ts`) + entry-point separado `src/worker.ts`.
**Componentes ainda a criar**: tabelas `webhook_outbox` e `webhook_dead_letter` (Prisma), função `publishWebhookEvent(tx, order, fromStatus, toStatus)` a ser chamada de dentro de `OrderService.changeStatus`, endpoints CRUD de configuração, `GET /webhooks/:id/deliveries`, `POST /admin/webhooks/dead-letter/:id/replay`.
**Tecnologias previstas**: mesmas do restante do projeto (Prisma, Zod, Pino, `AppError`) + biblioteca de HTTP client para o worker (não especificada na reunião) + `crypto` nativo do Node para HMAC-SHA256 (não especificado explicitamente, mas é a via padrão em Node/TS).
**Dependências**: Acopla-se fortemente a `OrderService.changeStatus` (ponto de disparo) e reaproveita toda a infraestrutura de `SHARED`/`MIDDLEWARES`/`CONFIG`.
**Escopo**: Ainda não mensurável em arquivos (não existe); pela quantidade de decisões (outbox, worker, retry, DLQ, HMAC, CRUD, deliveries, admin replay) é o maior módulo em complexidade de todo o projeto quando implementado.

## Preocupações Transversais

- **Infraestrutura**: apenas MySQL 8.0 via Docker Compose hoje. A feature de webhooks vai introduzir um segundo processo Node de longa duração (`src/worker.ts`) rodando fora do processo HTTP principal, algo inédito no projeto atual (hoje só existe `src/server.ts`).
- **Autenticação/Autorização**: JWT stateless (`authenticate`) + RBAC simples de dois papéis (`requireRole`). A reunião estende o uso existente de `requireRole('ADMIN')` sem criar novo mecanismo.
- **Camada de Dados**: Prisma como único ponto de acesso ao MySQL; todas as PKs são UUID; mutações críticas (estoque + status) sempre dentro de `$transaction`. A feature de webhooks depende de estender esse mesmo padrão transacional.
- **Camada de API**: prefixo `/api/v1`, paginação padrão (`shared/http/response.ts`), erros padronizados via `AppError`/`error.middleware.ts`, validação de entrada centralizada via `validate()` + Zod.
- **Observabilidade**: logging estruturado Pino com redaction de segredos; nenhuma métrica ou tracing hoje. A reunião não decide explicitamente uma estratégia de observabilidade específica para o worker (ponto potencialmente relevante para o FDD, mas não confirmado em `TRANSCRICAO.md`).
- **Integrações externas**: nenhuma hoje. A feature de webhooks introduz a primeira integração HTTP outbound do sistema com terceiros, com todas as implicações de segurança (HMAC, TLS, rotação de secret) tratadas na reunião por Sofia.

---

## Diretrizes para a Fase 2

- Nenhum ADR pode ser extraído do módulo `WEBHOOKS` a partir de evidência de **código**, pois o módulo não existe — toda evidência para esse módulo vem exclusivamente de `TRANSCRICAO.md`. Isso é esperado e não deve ser tratado como fraqueza da análise.
- Para os módulos `AUTH`, `USERS`, `CUSTOMERS`, `PRODUCTS`, `ORDERS`, `SHARED`, `MIDDLEWARES`, `CONFIG`: o histórico de Git não oferece granularidade temporal (repositório criado em commit único de inicialização — `7ef4317 init repository`), então o enriquecimento de Git deve ser registrado como indisponível/não informativo, apoiando-se apenas na análise estática do código.
- As 6 decisões estruturais centrais da reunião (outbox, retry/backoff/DLQ, HMAC-SHA256 com secret por endpoint, at-least-once com `X-Event-Id`, worker em processo separado com polling, reuso de padrões existentes) se enquadram claramente na Categoria 1 (Infraestrutura) e Categoria 4 (Arquitetura de API/entrega) da Etapa 0 — candidatas fortes a `must-document/`.
- Decisões secundárias como TLS obrigatório, limite de payload 64KB, e o formato exato do payload/headers foram explicitamente classificadas pela própria equipe (Larissa, `[09:24]`) como não-arquiteturais — tratar como Sinal de Alerta 3/5 (detalhe de configuração / granular demais), a consolidar na ADR estratégica correspondente, não como ADR isolado.
- Pontos em aberto (rate limiting, ordering multi-worker, notificação por e-mail) não devem virar ADR "Decisão", mas podem ser registrados como "Questões em Aberto" para uso no RFC.
- Sugestão de ordem de análise da Fase 2: `WEBHOOKS` primeiro (maior densidade de decisões confirmadas), seguido por `ORDERS` (ponto de integração), depois `SHARED`/`MIDDLEWARES` (para o ADR de reuso de padrões existentes).
