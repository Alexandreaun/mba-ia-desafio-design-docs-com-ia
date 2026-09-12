### PRD: OMS (Order Management System) Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-08-31
Responsável: Larissa (Tech Lead, conduziu a reunião de refinamento e declarou intenção de abrir o documento de design da feature, `[09:50]` Larissa)

---

### Resumo

O OMS é uma API de gestão de pedidos B2B, organizada em módulos por domínio (`src/modules/{auth,users,customers,products,orders}`), que hoje não possui nenhum mecanismo de notificação assíncrona de eventos. Esta feature estende o OMS existente com um sistema de webhooks outbound: quando o status de um pedido muda, a plataforma passa a notificar automaticamente os sistemas de clientes B2B cadastrados, eliminando a necessidade de polling manual em `GET /orders`.

A solução proposta usa o padrão outbox sobre o MySQL já operado pelo time: a mudança de status de um pedido insere, dentro da mesma transação que já atualiza o pedido, um evento em uma tabela de outbox; um processo worker separado, em polling a cada 2 segundos, lê essa tabela e entrega os eventos via HTTP, com autenticação HMAC-SHA256, retry com backoff exponencial e Dead Letter Queue (DLQ) para falhas definitivas. Essa arquitetura já foi validada tecnicamente pela equipe e formalizada nas ADR-003 a ADR-008; este PRD não reabre essas escolhas, traduzindo-as para o que a feature entrega em termos de produto e negócio.

A feature é aditiva sobre o sistema existente: nenhuma rota ou comportamento hoje ativo em `orders`, `customers`, `products`, `users` ou `auth` é removido ou alterado. O único ponto de integração é `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`).

---

### Contexto e problema

Público-alvo
- Clientes B2B integradores da plataforma (identificados nominalmente na reunião: Atlas Comercial, MaxDistribuição e Nova Cargo), que hoje consomem o OMS via API para acompanhar o status de pedidos que fazem.
- Usuários internos operadores/administradores do OMS, que cadastram e gerenciam webhooks em nome de um `customer` via API autenticada (FDD, Seção 1, "Atores").
- Administradores (role `ADMIN`), únicos autorizados a reprocessar manualmente eventos presos em DLQ.

Cenários de uso chave
- Um cliente B2B (ex.: Atlas Comercial) cadastra um webhook informando a URL do seu sistema e os status de pedido que deseja acompanhar (ex.: `SHIPPED`, `DELIVERED`); a partir daí, toda vez que um pedido dele muda para um desses status, o sistema dele é notificado automaticamente em vez de precisar consultar `GET /orders` repetidamente (`[09:00]-[09:02]` Marcos).
- O time de operações do cliente consulta o histórico de entregas de um webhook para investigar se uma notificação específica chegou, com sucesso ou falha, e o tempo de resposta do seu próprio sistema (`[09:34]` Marcos).
- Um administrador do OMS identifica um evento que esgotou todas as tentativas de entrega e ficou em DLQ, investiga o motivo da falha e o reenvia manualmente após o cliente confirmar que seu sistema voltou a funcionar (`[09:18]`, `[09:35]-[09:36]` Diego/Sofia/Larissa).
- Um cliente que precisa trocar de infraestrutura de segurança solicita a rotação da secret do seu webhook pela API, mantendo a integração funcionando durante a transição de 24 horas em que ambas as secrets são aceitas (`[09:21]` Sofia).

Onde essa feature será implantada
- No próprio OMS existente (sistema já em produção), como extensão aditiva: um novo módulo `src/modules/webhooks` seguindo a convenção estrutural dos módulos já existentes, mais um novo processo Node dedicado (`src/worker.ts`), rodando ao lado do processo de API já existente (`src/server.ts`). Nenhum sistema novo é introduzido; a única infraestrutura de dados usada é o MySQL já operado via Prisma (ADR-002).

Problemas priorizados
- Custo operacional do polling para clientes B2B: os três clientes hoje ficam consultando periodicamente `GET /orders` para detectar mudanças de status, o que é lento e caro para eles de integrar e operar (`[09:00]` Marcos). Impacto: risco concreto de churn, com a Atlas Comercial sinalizando possível migração para um concorrente se a entrega não ocorrer até o fim do trimestre. Prioridade: alta.
- Ausência de mecanismo de notificação assíncrona e confiável no sistema: tecnicamente, não existe hoje nenhum ponto de extensão para publicar eventos de domínio sem acoplar a disponibilidade de sistemas externos à disponibilidade da transação de negócio que muda o status do pedido (RFC, Seção 3). Impacto: qualquer solução ingênua (chamada HTTP síncrona) arriscaria travar operações internas por causa de um cliente externo lento ou fora do ar. Prioridade: alta.

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Eliminar a dependência de polling manual para detectar mudanças de status de pedido | Latência entre a mudança de status e a entrega do evento ao cliente | Abaixo de 10 segundos no caso comum, aceito pelos clientes como "tempo real" (`[09:02]` Marcos); pior caso de 2 segundos determinado pelo intervalo de polling do worker (ADR-008) |
| Garantir que nenhuma notificação de mudança de status seja perdida | Proporção de transições de status com pelo menos um webhook interessado que geram exatamente um evento correspondente na outbox | 100 por cento, garantido pela atomicidade transacional (ADR-003, ADR-006); nenhuma tolerância a perda de evento aceita pela equipe |
| Reter clientes B2B estratégicos que condicionaram a continuidade do contrato à entrega da feature | Entrega da primeira versão dentro do prazo comunicado | Até o fim do trimestre corrente, dentro da estimativa de três sprints (incluindo revisão de segurança da Sofia), conforme `[09:00]` e `[09:45]-[09:46]` Marcos/Larissa |
| Tolerar indisponibilidade temporária de sistemas de clientes sem perda definitiva de evento | Janela de tentativas de reentrega antes de mover para DLQ | Cobertura de aproximadamente 15 horas de indisponibilidade via 5 tentativas com backoff exponencial (1m/5m/30m/2h/12h), ADR-007 |

Hipótese (meta não sustentada explicitamente pelas fontes): não há, em nenhuma das cinco fontes, uma meta numérica para taxa de sucesso de entrega global (ex.: "99 por cento dos eventos entregues com sucesso na primeira tentativa") nem para volume de eventos/throughput esperado do worker sob carga real; a RFC declara isso como TBD (RFC, Seções 6.2, 15, 16). Como meta quantitativa plausível de produto, oferecemos como hipótese, a escolher pelo time: (a) taxa de sucesso de entrega (2xx) na primeira tentativa acima de 95 por cento; (b) menos de 1 por cento dos eventos mensais terminando em DLQ; (c) não definir meta até que haja dados reais de operação nos primeiros 30 dias.

---

### Escopo

Incluso
- CRUD completo de configuração de webhook (criação, edição, remoção, listagem) por `customer`, com secret de assinatura gerada pela plataforma e devolvida apenas na criação (`[09:31]-[09:33]` Marcos/Bruno; RFC Seção 6.1; FDD Seção 3).
- Filtro de interesse por status de pedido: cada webhook escolhe quais status deseja receber, e o sistema só insere evento na outbox se houver interesse declarado (`[09:33]-[09:34]` Marcos/Bruno/Diego).
- Consulta de histórico de entregas por webhook (sucesso/falha, payload, response, tempo de resposta) (`[09:34]` Marcos).
- Rotação de secret via API, com a secret antiga permanecendo válida por 24 horas em paralelo à nova (`[09:21]` Sofia; ADR-004).
- Endpoint administrativo de reprocessamento manual de eventos em DLQ, restrito a role `ADMIN`, com log de auditoria de quem executou o replay (`[09:18]`, `[09:35]-[09:36]` Diego/Sofia/Larissa; ADR-007).
- Publicação atômica do evento de webhook dentro da transação de mudança de status de pedido, sem risco de inconsistência entre status e notificação (ADR-003, ADR-006).
- Entrega via processo worker separado, com autenticação HMAC-SHA256 por endpoint, garantia de entrega at-least-once com deduplicação por `X-Event-Id`, e retry com backoff exponencial antes de mover para DLQ (ADR-004, ADR-005, ADR-007, ADR-008).

Fora de escopo (itens descartados ou adiados na reunião)
- Notificação alternativa por e-mail em caso de falhas recorrentes de entrega ao cliente: perguntado explicitamente por Marcos e recusado por Larissa nesta fase, condicionado a uma fase futura após medição de impacto ("Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto", `[09:37]` Larissa; "Beleza, anotado como 'futuro'", `[09:38]` Marcos).
- Rate limiting de envio de webhooks para clientes com picos de eventos simultâneos: levantado por Diego como preocupação real (cenário de 50 pedidos mudando de status em um minuto), mas deliberadamente não incorporado a esta fase ("Eu acho que não. A gente observa e implementa se virar problema. Mas vale registrar como ponto em aberto", `[09:39]` Diego; "Tá. Fica como 'observar e decidir depois'", `[09:39]` Larissa).
- Dashboard visual para o cliente gerenciar seus próprios webhooks: perguntado por Marcos e recusado por Larissa, ficando a cargo de um projeto separado do time de frontend ("Não, agora não. Só endpoints. Painel é projeto separado do time de frontend", `[09:40]` Larissa).
- Webhooks inbound (clientes enviando dados para a plataforma): descartado logo no início da reunião, o fluxo é estritamente outbound ("Só saindo da gente pra eles. Eles querem receber, não mandar", `[09:02]` Marcos).
- Arquivamento/purga de eventos já entregues na outbox após 30 dias: mencionado como necessidade futura por Diego, mas explicitamente fora do escopo desta feature ("Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature", `[09:08]` Diego).
- Suporte a múltiplos workers em paralelo com garantia de ordering global: tratado como evolução futura, fora do escopo desta decisão ("Mas isso é problema do futuro, não agora", `[09:13]` Diego).

Nota de consistência: os itens "fora de escopo" acima não contradizem o "incluso": a feature entrega notificação confiável e auditável via API e DLQ manual, sem prometer canais alternativos de alerta, controle de tráfego de saída, interface visual, fluxo inbound, retenção de longo prazo ou escalabilidade horizontal do worker.

---

### Requisitos funcionais

#### FR-001 Cadastro de webhook
O cliente (via usuário autenticado do OMS) cadastra um webhook para um `customer`, informando a URL de destino e a lista de status de pedido de interesse; a plataforma gera a secret de assinatura e a devolve apenas nesta resposta.

**Fluxo principal**
- Usuário autenticado (JWT válido, qualquer role) envia `POST /api/v1/webhooks` com `customerId`, `url` e lista de status de interesse (`[09:31]` Marcos; FDD Contrato 1).
- Sistema valida que a `url` é HTTPS, gera a secret e cria o cadastro com estado ativo.
- Resposta devolve o cadastro criado incluindo a secret gerada (única vez que ela é exposta).

**Fluxos alternativos e exceções**
- `customerId` informado não existe: erro de recurso não encontrado (FDD Contrato 1).
- Lista de status de interesse contém valor fora dos status válidos de pedido: rejeitado antes de persistir (FDD, matriz de erros, `WEBHOOK_INVALID_STATUS_FILTER`, hipótese de nome de erro alinhada ao padrão do projeto).

**Erros previstos**
- URL não HTTPS ou malformada (`WEBHOOK_INVALID_URL`, `[09:23]` Sofia).
- Token ausente ou inválido (não autenticado).

**Prioridade:** alta

---

#### FR-002 Edição de webhook
O cliente edita um webhook já cadastrado, podendo alterar a lista de status de interesse e o estado ativo/inativo.

**Fluxo principal**
- Usuário autenticado envia `PATCH /api/v1/webhooks/:id` com os campos a alterar (`[09:33]` Bruno; RFC Seção 6.1).
- Sistema valida e atualiza o cadastro; a secret nunca é retornada nesta operação.

**Fluxos alternativos e exceções**
- Alteração de `url` para um valor não HTTPS: rejeitada da mesma forma que no cadastro.

**Erros previstos**
- Webhook não encontrado pelo `id` informado (`WEBHOOK_NOT_FOUND`).
- Payload inválido (erro de validação).

**Prioridade:** alta

---

#### FR-003 Remoção de webhook
O cliente remove um webhook cadastrado, interrompendo o recebimento de futuras notificações por aquele cadastro.

**Fluxo principal**
- Usuário autenticado envia `DELETE /api/v1/webhooks/:id` (`[09:33]` Bruno).
- Sistema remove o cadastro; resposta sem corpo.

**Fluxos alternativos e exceções**
- Nenhuma variação adicional sustentada pelas fontes.

**Erros previstos**
- Webhook não encontrado pelo `id` informado (`WEBHOOK_NOT_FOUND`).

**Prioridade:** media

---

#### FR-004 Listagem de webhooks por cliente
O cliente consulta os webhooks cadastrados para um `customer`.

**Fluxo principal**
- Usuário autenticado envia `GET /api/v1/webhooks` filtrando por `customerId` (`[09:33]` Bruno; `[09:32]` Larissa: "customer_id é passado no body ou no path. Não vem do JWT").
- Sistema retorna lista paginada dos cadastros (sem expor a secret).

**Fluxos alternativos e exceções**
- Nenhum webhook cadastrado para o `customerId`: retorna lista vazia.

**Erros previstos**
- Parâmetros de paginação/filtro inválidos (erro de validação).

**Prioridade:** media

---

#### FR-005 Filtragem de eventos por status de interesse na origem
O sistema só gera um evento de notificação para os webhooks de um cliente que declararam interesse explícito no status resultante da transição.

**Fluxo principal**
- Durante a mudança de status de um pedido, o sistema consulta as configurações de webhook ativas daquele `customer` e verifica quais possuem o `toStatus` na lista de interesse (`[09:33]-[09:34]` Marcos/Bruno/Diego).
- Insere um evento na outbox apenas para as configurações correspondentes; se nenhuma configuração tiver interesse naquele status, nenhum evento é criado ("Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela", `[09:34]` Bruno).

**Fluxos alternativos e exceções**
- Cliente com múltiplos webhooks cadastrados e interesses distintos: cada webhook interessado recebe seu próprio evento.

**Erros previstos**
- Falha na consulta de configuração ou na inserção do evento provoca rollback de toda a transação de mudança de status, incluindo estoque e histórico (ADR-003: "não pode ter caso de status mudar e evento não sair").

**Prioridade:** alta

---

#### FR-006 Consulta de histórico de entregas
O cliente consulta o histórico de tentativas de entrega de um webhook específico, incluindo sucesso/falha, payload, resposta recebida e tempo de resposta.

**Fluxo principal**
- Usuário autenticado envia `GET /api/v1/webhooks/:id/deliveries` (`[09:34]` Marcos).
- Sistema retorna lista paginada de tentativas de entrega, preservando o registro de todas as tentativas de um mesmo evento, não apenas a última (FDD Seção 4, "Retry").

**Fluxos alternativos e exceções**
- Webhook sem nenhuma entrega ainda: retorna lista vazia.

**Erros previstos**
- Webhook não encontrado pelo `id` informado (`WEBHOOK_NOT_FOUND`).

**Prioridade:** alta

---

#### FR-007 Rotação de secret com grace period
O cliente solicita uma nova secret para um webhook cadastrado, sem interromper a validação de eventos já em trânsito com a secret anterior.

**Fluxo principal**
- Usuário autenticado solicita rotação de secret para um webhook (`[09:21]` Sofia; ADR-004).
- Sistema gera nova secret e mantém a secret anterior válida por 24 horas em paralelo, permitindo ao cliente migrar sua verificação sem downtime.
- Após as 24 horas, apenas a nova secret é aceita.

**Fluxos alternativos e exceções**
- Nova rotação solicitada dentro do grace period de uma rotação anterior ainda ativa: comportamento exato não definido nas fontes; FDD marca como hipótese entre rejeitar com conflito ou permitir e expirar imediatamente a secret mais antiga (FDD Contrato 6, matriz de erros).

**Erros previstos**
- Webhook não encontrado pelo `id` informado (`WEBHOOK_NOT_FOUND`).
- Possível conflito de rotação sobreposta (hipótese, `WEBHOOK_SECRET_ROTATION_CONFLICT`, FDD matriz de erros).

**Prioridade:** alta

---

#### FR-008 Entrega assinada de evento ao cliente
O worker entrega cada evento pendente ao endpoint HTTPS do cliente, assinado com HMAC-SHA256, permitindo ao cliente verificar autenticidade e integridade.

**Fluxo principal**
- O worker seleciona periodicamente eventos pendentes na outbox (a cada 2 segundos, ADR-008), calcula a assinatura HMAC-SHA256 do corpo usando a secret ativa daquele endpoint (ADR-004) e envia via HTTP com timeout de 10 segundos (`[09:42]` Diego/Sofia).
- Resposta 2xx dentro do timeout: evento marcado como entregue com sucesso.

**Fluxos alternativos e exceções**
- Resposta não-2xx, erro de rede ou timeout: entra no fluxo de retry (ver FR-009).

**Erros previstos**
- Falha de entrega HTTP (`WEBHOOK_DELIVERY_FAILED`, FDD Seção 6).

**Prioridade:** alta

---

#### FR-009 Retry com backoff exponencial
Quando uma entrega falha, o sistema tenta reenviar o evento automaticamente, aumentando progressivamente o intervalo entre tentativas, até um teto de tentativas.

**Fluxo principal**
- Em caso de falha ou timeout, o sistema agenda uma nova tentativa seguindo a progressão de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, totalizando 5 tentativas (ADR-007).
- O `X-Event-Id` do evento permanece idêntico em todas as tentativas, permitindo ao cliente identificar reenvios do mesmo evento (ADR-005).

**Fluxos alternativos e exceções**
- Cliente volta a responder com sucesso em qualquer tentativa intermediária: evento marcado como entregue, tentativas seguintes canceladas.

**Erros previstos**
- Falhas sucessivas até a 5ª tentativa esgotam o teto de retry e acionam o encaminhamento para DLQ (ver FR-010).

**Prioridade:** alta

---

#### FR-010 Dead Letter Queue e reprocessamento manual
Eventos que esgotam todas as tentativas de entrega são registrados de forma auditável e podem ser reprocessados manualmente por um administrador.

**Fluxo principal**
- Ao esgotar a 5ª tentativa ainda com falha, o sistema registra o evento em uma fila de eventos definitivamente falhos (DLQ), com o payload, o motivo da última falha e o timestamp (`[09:18]` Diego; ADR-007).
- Um administrador (role `ADMIN`) consulta os eventos em DLQ e aciona `POST /api/v1/admin/webhooks/dead-letter/:id/replay` para reintroduzir o evento na fila de entrega como pendente (`[09:18]`, `[09:35]` Diego).
- Toda execução de replay é registrada em log de auditoria identificando o administrador responsável (`[09:36]` Sofia).

**Fluxos alternativos e exceções**
- Usuário sem role `ADMIN` tenta acionar o replay: acesso negado (`[09:35]-[09:36]` Sofia/Larissa).

**Erros previstos**
- Evento de DLQ não encontrado pelo `id` informado (`WEBHOOK_DEAD_LETTER_NOT_FOUND`, ADR-007).
- Acesso negado a usuário sem role `ADMIN` (erro de permissão).

**Prioridade:** alta

---

### Requisitos não funcionais

Performance
- Latência de entrega de eventos inferior a 10 segundos no caso comum, com pior caso de 2 segundos determinado pelo intervalo de polling do worker (`[09:02]`, `[09:09]-[09:10]` Marcos/Diego/Larissa; ADR-008).
- Timeout de 10 segundos por chamada HTTP de entrega ao cliente; ausência de resposta nesse intervalo é tratada como falha (`[09:42]` Diego/Sofia).

Disponibilidade
- O processo de entrega de webhooks (worker) não deve ser afetado por reinícios ou deploys do processo de API, e vice-versa; ambos operam como processos independentes coordenados apenas pelo banco de dados (`[09:11]` Diego; ADR-008).
- Hipótese (não quantificada nas fontes): meta de disponibilidade de 99.5 por cento mensal para o processo worker, alinhada ao padrão de sistemas internos B2B, já que não há definição de mecanismo de supervisão/restart automático do worker em caso de crash (lacuna explícita da ADR-008, tratada como bloqueio de produção pelo FDD Seção 11).

Segurança e autorização
- Toda chamada de entrega ao cliente é assinada com HMAC-SHA256, com secret exclusiva por endpoint cadastrado, nunca uma secret global (`[09:19]-[09:22]` Sofia; ADR-004).
- URLs de webhook devem ser obrigatoriamente HTTPS; cadastro com HTTP é recusado por validação (`[09:23]` Sofia).
- Limite de tamanho de payload de 64KB, com erro explícito em vez de truncamento silencioso (`[09:23]-[09:24]` Sofia/Diego).
- O CRUD de configuração de webhook exige apenas JWT válido de qualquer role nesta fase; o endpoint de replay de DLQ exige role `ADMIN`, reaproveitando o mecanismo `requireRole` já existente (`src/middlewares/auth.middleware.ts:49-61`), com log de auditoria obrigatório por execução (`[09:35]-[09:36]` Sofia).
- Lacuna explícita nas fontes: não há definição de como a secret será armazenada em repouso (texto plano, hash, ou criptografia com KMS/chave de aplicação), nem de fluxo de revogação de emergência de uma secret comprometida durante o grace period de rotação (ADR-004).

Observabilidade
- Histórico de entregas (sucesso/falha, payload, response, tempo de resposta) deve ser consultável pelo cliente via API (`[09:34]` Marcos).
- Falhas definitivas ficam auditáveis em DLQ com motivo registrado (`[09:18]` Diego; ADR-007).
- O módulo reaproveita o logger estruturado Pino já existente no projeto, sem novo mecanismo de logging (`[09:29]` Bruno).
- Lacuna explícita nas fontes: não há definição de métricas numéricas (ex.: taxa de sucesso de entrega, latência p95, volume de eventos por hora), nem de processo formal de monitoramento periódico da DLQ (RFC Seções 16, 21; FDD Seção 11).

Confiabilidade e integridade de dados
- A inserção do evento de notificação na outbox ocorre dentro da mesma transação SQL que já atualiza o status do pedido, o histórico e o estoque; a mudança de status nunca é persistida sem o evento correspondente, e vice-versa (ADR-003, ADR-006).
- Garantia de entrega at-least-once, com deduplicação delegada ao cliente via identificador único (`X-Event-Id`) constante em todas as tentativas de reenvio (ADR-005).
- Garantia de ordering de entrega apenas por pedido individual (`order_id`) e apenas enquanto houver um único worker ativo processando a fila; não há garantia de ordering global entre pedidos distintos, o que os clientes nunca solicitaram (`[09:12]-[09:14]` Diego/Bruno/Marcos).

Compatibilidade e portabilidade
- Os novos endpoints seguem a convenção já usada pelo restante da API, versionados sob `/api/v1`, sem necessidade de uma versão distinta (FDD Seção 9).
- Nenhuma rota ou comportamento externo dos módulos existentes (`orders`, `customers`, `products`, `users`, `auth`) é alterado; a mudança é estritamente aditiva.

Compliance
- Toda execução do endpoint administrativo de replay de DLQ gera um registro de auditoria identificando o administrador responsável (`[09:36]` Sofia).
- Lacuna: não há, em nenhuma das cinco fontes, menção a requisitos regulatórios específicos (ex.: LGPD sobre dados de pedido trafegados a terceiros) aplicáveis a esta feature (RFC Seção 7).

Acessibilidade no frontend consumidor
- Não aplicável: esta feature não inclui nenhuma interface visual; a interação é somente via API, e um eventual painel do cliente foi explicitamente definido como projeto separado do time de frontend, fora de escopo (`[09:39]-[09:40]` Larissa/Marcos).

---

### Decisões e trade-offs principais

#### Decisão: Publicação atômica do evento de webhook dentro da transação de mudança de status (ADR-003)
- **Justificativa:** garante que a mudança de status de um pedido e o registro do evento de notificação ocorram de forma atômica, eliminando a classe de bug em que um pedido muda de status mas o cliente nunca é notificado, ou é notificado sem a mudança ter ocorrido de fato (`[09:40]-[09:41]` Bruno/Diego).
- **Trade-off:** a transação de mudança de status, já descrita como "pesada" pela equipe, passa a incluir mais uma escrita, aumentando ligeiramente sua duração e a janela de contenção de locks no banco; qualquer alteração futura nesse fluxo precisa preservar essa inserção sob risco de romper a garantia de consistência.

#### Decisão: Padrão outbox sobre o MySQL existente, sem fila externa dedicada (ADR-006)
- **Justificativa:** reaproveita a infraestrutura de dados já operada pelo time (MySQL via Prisma), evitando subir e operar um componente novo (ex.: Redis Streams) considerado overengineering para o tamanho da equipe (`[09:07]` Diego).
- **Trade-off:** aceita o teto de desempenho e a ausência de notificação reativa nativa do MySQL que uma fila dedicada ofereceria, exigindo leitura por polling em vez de push em tempo real.

#### Decisão: Autenticação HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h (ADR-004)
- **Justificativa:** garante autenticidade e integridade dos eventos entregues a terceiros; a secret única por endpoint (em vez de secret global) reduz o raio de impacto de um vazamento a um único cadastro, motivada por um incidente real em que um cliente vazou uma secret em log de sua própria aplicação (`[09:22]` Diego).
- **Trade-off:** a plataforma assume uma responsabilidade nova e permanente de gestão de ciclo de vida de credenciais por cliente (geração, armazenamento, rotação, expiração), inexistente até então, já que a autenticação interna via JWT é stateless.

#### Decisão: Garantia de entrega at-least-once com deduplicação via X-Event-Id (ADR-005)
- **Justificativa:** garantir exactly-once exigiria coordenação transacional complexa entre plataforma e cliente; at-least-once com deduplicação client-side é o padrão adotado por provedores de referência de mercado como Stripe e GitHub (`[09:25]` Diego).
- **Trade-off:** desloca uma responsabilidade de engenharia real (deduplicação) para cada cliente B2B integrador, objeção explícita levantada por Sofia (`[09:25]` Sofia: "Isso joga responsabilidade pro cliente"), mitigada pela documentação destacada do contrato no portal de desenvolvedor (`[09:26]` Marcos).

#### Decisão: Retry com backoff exponencial de 5 tentativas (1m/5m/30m/2h/12h) e DLQ em tabela separada (ADR-007)
- **Justificativa:** cobre janelas de indisponibilidade de cliente da ordem de horas, como um caso real de manutenção planejada de duas horas já vivido pela equipe (`[09:16]` Diego), sem manter eventos "pendurados" indefinidamente.
- **Trade-off:** mantém eventos não confirmados "em voo" por até aproximadamente 15 horas antes de considerar falha definitiva, uma janela mais longa do que uma política mais agressiva (ex.: 3 tentativas) ofereceria; não há, nas fontes, processo formal de monitoramento periódico da DLQ.

#### Decisão: Worker de entrega em processo Node separado, com polling a cada 2 segundos (ADR-008)
- **Justificativa:** evita que um reinício ou deploy da API interrompa a entrega de eventos pendentes; MySQL não oferece mecanismo nativo de notificação reativa a processos externos, tornando o polling a opção tecnicamente viável (`[09:09]-[09:11]` Diego).
- **Trade-off:** introduz uma latência mínima inerente de até 2 segundos (folgada frente ao requisito de 10 segundos) e a operação de um segundo processo de longa duração, cujo mecanismo de supervisão/restart em caso de crash não está definido em nenhuma fonte disponível (lacuna explícita da ADR-008).

#### Decisão: Reuso da autenticação JWT/RBAC existente sem alterações (ADR-001)
- **Justificativa:** o mecanismo `authenticate`/`requireRole` já estabelecido no projeto é reaproveitado sem alteração para proteger os endpoints de configuração de webhook e o endpoint administrativo de replay, evitando introduzir um novo modelo de autenticação interna (`[09:35]-[09:36]`).
- **Trade-off:** o modelo binário de papéis (`ADMIN`/`OPERATOR`) permanece sem granularidade por `customer`; qualquer usuário autenticado de qualquer role pode operar o CRUD de configuração de webhook nesta fase, com endurecimento futuro do RBAC explicitamente não decidido (`[09:36]-[09:37]` Marcos/Sofia).

#### Decisão: Prisma como camada de acesso a dados única, com PrismaClient próprio por processo (ADR-002)
- **Justificativa:** mantém a tipagem forte e a base transacional (`prisma.$transaction`) já usada por `OrderService.changeStatus`, estendendo o padrão de singleton por processo para a nova topologia de dois processos (API e worker) sem necessidade de mecanismo de coordenação adicional (`[09:29]-[09:30]`).
- **Trade-off:** cada novo processo de longa duração introduzido no futuro precisa replicar corretamente a criação de sua própria instância de `PrismaClient`, sem um mecanismo automatizado que garanta isso estruturalmente.

---

### Dependências

#### technical: Ponto de integração único em OrderService.changeStatus
Toda a garantia de consistência da feature depende de uma única extensão ao método `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que hoje já executa, dentro de `prisma.$transaction`, a validação de transição (`canTransition`, `src/modules/orders/order.status.ts:12-14`), o ajuste de estoque (`shouldDebitStock`/`shouldReplenishStock`, `order.status.ts:29-37`) e a gravação de `OrderStatusHistory`. A equipe de Pedidos (Bruno) precisa entregar essa extensão antes que qualquer evento possa ser gerado.

#### technical: Hierarquia de erros AppError e middleware de erro centralizado
Todos os novos erros do módulo de webhooks devem seguir a hierarquia `AppError` já existente (`src/shared/errors/app-error.ts:3-16`, `src/shared/errors/http-errors.ts:1-63`), com códigos prefixados `WEBHOOK_*`, sem exigir alteração no middleware de erro centralizado (`src/middlewares/error.middleware.ts`), que já trata genericamente qualquer instância de `AppError` (`[09:28]-[09:29]` Bruno).

#### technical: Autenticação e RBAC existentes
Os endpoints de configuração de webhook e o endpoint administrativo de replay dependem do mecanismo `authenticate`/`requireRole` já implementado (`src/middlewares/auth.middleware.ts:27-61`, ADR-001), sem nenhuma alteração nesse middleware.

#### technical: Novas tabelas em prisma/schema.prisma
A feature depende da criação de pelo menos três novos modelos Prisma (configuração de webhook, outbox e DLQ), inexistentes hoje no schema (que só contém `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`), seguindo a convenção de UUID como chave primária já usada em todo o schema (`[09:50]-[09:51]` Larissa/Diego).

#### technical: Novo processo worker independente do processo de API
A entrega de eventos depende da criação de um novo entry-point de processo (`src/worker.ts`), espelhando a estrutura de `src/server.ts`, com script dedicado ainda a ser adicionado ao projeto, e instância própria de `PrismaClient` via `createPrismaClient()` (`src/config/database.ts`) (ADR-008).

#### organizational: Revisão de segurança dedicada da Sofia antes do deploy
A engenheira de segurança reservou pelo menos dois dias úteis para revisar especificamente o mecanismo de HMAC e a geração/armazenamento de secret antes de qualquer deploy em produção (`[09:46]` Sofia/Larissa). A feature não deve ir a produção sem essa revisão.

#### organizational: Comunicação de prazo aos clientes B2B
Marcos (Product Manager) é responsável por atualizar os três clientes (Atlas Comercial, MaxDistribuição, Nova Cargo) sobre o prazo estimado de entrega, condição citada pela própria Atlas como fator de continuidade do contrato (`[09:47]` Marcos).

#### organizational: Sessão de revisão técnica de design antes da implementação
Larissa declarou a intenção de agendar uma sessão de revisão do documento de design da feature com Bruno e Diego antes do início da codificação (`[09:50]` Larissa), o que é uma dependência de processo para o início seguro da implementação.

---

### Riscos e mitigação

#### Processo worker sem mecanismo de supervisão/restart definido pode interromper silenciosamente toda a entrega
- **Probabilidade:** media
- **Impacto:** um crash não tratado no processo worker interrompe toda a entrega de webhooks até intervenção manual, sem nenhum sinal automático, já que não há mecanismo de alerta de processo definido (lacuna explícita da ADR-008, tratada como bloqueio de produção pelo FDD).
- **Mitigação:**
  - Implementar tratamento de erros não capturados no worker antes de decidir entre reiniciar ou encerrar o processo.
  - Definir, antes do deploy, um mecanismo de restart automático (gerenciador de processo ou orquestrador); item que nenhuma fonte disponível resolve e que precisa ser fechado como decisão de produto/infraestrutura antes de produção.
- **Plano de contingência:** monitoramento manual periódico de atividade recente na outbox (eventos marcados como entregues) até que um mecanismo automatizado seja formalizado.

#### DLQ pode acumular falhas definitivas sem que ninguém perceba, por falta de processo formal de monitoramento
- **Probabilidade:** alta
- **Impacto:** falhas definitivas de entrega ficam registradas em DLQ, mas sem notificação automática de novo item nem revisão periódica definida, uma falha real de entrega para um cliente estratégico pode passar despercebida por tempo indeterminado (ADR-007).
- **Mitigação:**
  - Definir um responsável e uma cadência mínima de revisão manual da DLQ até que um mecanismo de alerta automatizado seja formalizado (recomendação do FDD, ainda não decidida pelas fontes).
  - Expor contagem de falhas no histórico de entregas já consultável pelo cliente, ainda que sem visibilidade direta da tabela de DLQ.
- **Plano de contingência:** revisão manual periódica por operação/suporte até que um processo formal seja definido em decisão futura.

#### Risco de churn de cliente estratégico caso o prazo de três sprints não seja cumprido
- **Probabilidade:** media
- **Impacto:** a Atlas Comercial sinalizou explicitamente risco de migração para um concorrente caso a feature não seja entregue até o fim do trimestre (`[09:00]` Marcos); atraso na entrega tem impacto comercial direto, não apenas técnico.
- **Mitigação:**
  - Escopo desta primeira entrega mantido estritamente ao que foi fechado na reunião, sem incorporar itens adiados (e-mail de falha, rate limiting, dashboard) que estenderiam o prazo.
  - Comunicação proativa de status por Marcos aos três clientes ao longo da implementação (`[09:47]` Marcos).
- **Plano de contingência:** nenhum plano de contingência formal de renegociação de prazo está registrado nas fontes; esta é uma lacuna de produto a ser tratada por Marcos caso o prazo de três sprints não se confirme.

#### Vazamento de secret de webhook compromete a autenticidade de eventos para um cliente
- **Probabilidade:** baixa a media (considerando que já houve um incidente real de vazamento de secret antes desta decisão)
- **Impacto:** um vazamento compromete a autenticidade de eventos para o endpoint cadastrado correspondente, com risco de o cliente aceitar eventos forjados (ADR-004).
- **Mitigação:**
  - Isolamento por secret única por endpoint, limitando o raio de impacto a um único cadastro.
  - Suporte a rotação via API com grace period de 24h.
- **Plano de contingência:** não há, em nenhuma fonte disponível, um fluxo de revogação de emergência definido para um comprometimento identificado durante o próprio grace period; esta lacuna deve ser resolvida antes do deploy em produção (ADR-004).

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Um cliente consegue cadastrar, editar, remover e listar webhooks para um `customer`, recebendo a secret apenas na criação.
- Uma mudança de status de pedido gera exatamente um evento de notificação para cada webhook ativo daquele cliente que declarou interesse no status resultante, e nenhum evento para os que não declararam.
- Se a inserção do evento de notificação falhar por qualquer motivo, a mudança de status correspondente também não é persistida (rollback completo).
- Cada chamada de entrega ao cliente inclui os headers de identificação de evento, assinatura HMAC e identificação do cadastro de webhook, permitindo ao cliente validar autenticidade e deduplicar reenvios.
- Um evento reenviado após falha preserva o mesmo identificador único em todas as tentativas.
- Uma falha de entrega segue a progressão de tentativas acordada (1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas) e só é movida para DLQ após esgotar as 5 tentativas.
- Um administrador consegue reprocessar manualmente um evento em DLQ, e essa ação fica registrada em log de auditoria identificando quem a executou.
- Um usuário sem role `ADMIN` recebe acesso negado ao tentar reprocessar um evento em DLQ.
- Um cliente consegue consultar o histórico completo de tentativas de entrega de um webhook, incluindo tentativas malsucedidas.
- Uma rotação de secret mantém a secret anterior válida por 24 horas, e a rejeita depois desse prazo.
- Um cadastro de webhook com URL não HTTPS é rejeitado, sem persistir nenhum registro.
- Nenhuma rota ou comportamento hoje existente em `orders`, `customers`, `products`, `users` ou `auth` muda de comportamento externo.
- A revisão de segurança dedicada de Sofia sobre HMAC e geração de secret ocorre e é aprovada antes do primeiro deploy em produção.

---

### Testes e validação

Tipos de teste obrigatórios
- Testes de integração cobrindo o fluxo transacional completo de `OrderService.changeStatus` com a nova inserção de evento, incluindo o cenário de rollback forçado para comprovar que nenhum evento é criado sem a transição correspondente ser commitada (FDD, Critérios de Aceite Técnicos).
- Testes unitários para a lógica de filtragem de interesse por status na inserção do evento (FR-005).
- Testes de integração ponta a ponta para o ciclo completo de retry e DLQ, simulando falhas sucessivas até o esgotamento das 5 tentativas e verificando a progressão exata de backoff.
- Teste de segurança dedicado para verificação da assinatura HMAC-SHA256 e validação de que a secret antiga permanece aceita apenas durante o grace period de 24h e é rejeitada depois.
- Teste de permissão (autorização) garantindo que o endpoint de replay de DLQ retorna acesso negado para usuários sem role `ADMIN` e sucesso, com log de auditoria, para usuários `ADMIN`.
- Teste de validação de schema para rejeição de URL não HTTPS e de payload acima do limite de 64KB.

Estratégia de validação
- Testes de integração e unitários como base de confiança técnica, seguindo o padrão de testes já usado no projeto (Vitest, banco real truncado entre testes, conforme `tests/setup.ts`).
- Revisão de segurança dedicada e presencial da Sofia (mínimo dois dias úteis), focada especificamente em HMAC e geração/armazenamento de secret, como gate obrigatório antes do deploy (`[09:46]` Sofia/Larissa).
- Sessão de revisão técnica do design entre Larissa, Bruno e Diego antes do início da implementação, como validação de arquitetura prévia à codificação (`[09:50]` Larissa).
- Validação funcional guiada por cenário real de cliente: simular o fluxo completo de um dos três clientes citados (cadastro de webhook, mudança de status de um pedido de teste, recebimento do evento, consulta de histórico de entregas) antes da liberação a qualquer cliente em produção.
