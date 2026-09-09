# ADR-008: Worker de Entrega em Processo Separado com Polling

**Status:** Proposta
**Data:** 31-08-2026

## Status

Proposta. A decisão foi debatida e fechada de forma consensual pela equipe durante a reunião de refinamento técnico (`[09:09]`-`[09:13]`, `[09:29]`-`[09:30]`), mas ainda não há implementação no código: `src/worker.ts` e o script `npm run worker` não existem no repositório até o momento, e o único processo de longa duração hoje é `src/server.ts`.

## Contexto

O sistema de pedidos precisa notificar sistemas externos sobre mudanças de status via webhook, com um requisito de negócio comunicado pelos clientes de latência de entrega abaixo de 10 segundos. A equipe já havia decidido registrar os eventos pendentes em uma tabela de outbox no MySQL (ver ADR relacionada sobre o padrão Outbox); restava definir quem e como consome essa tabela para efetivamente disparar as chamadas HTTP.

Tecnicamente, o projeto hoje opera com um único processo Node.js de longa duração (`src/server.ts`) e um `PrismaClient` singleton por processo (`src/config/database.ts`). O MySQL, ao contrário do PostgreSQL, não oferece um mecanismo nativo de notificação assíncrona a processos externos (sem equivalente a `LISTEN/NOTIFY`); triggers de banco só executam SQL, não conseguem acionar um worker fora do banco. Isso restringe as opções técnicas viáveis para o mecanismo de leitura da outbox.

Do ponto de vista operacional, a equipe também levantou a preocupação de que, se o consumo dos eventos rodasse dentro do próprio processo da API, um reinício ou deploy da API interromperia a entrega de webhooks pendentes — algo considerado inaceitável dado o requisito de confiabilidade da feature.

## Decisão

A leitura e o envio dos eventos da outbox serão feitos por um processo Node.js **separado** do processo HTTP principal, com entry-point próprio (`src/worker.ts`) e script dedicado (`npm run worker`), seguindo o mesmo padrão estrutural já usado por `src/server.ts`. O mecanismo de leitura será **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos, processando-os e marcando-os como entregues. O worker mantém sua própria instância de `PrismaClient`, conectada à mesma `DATABASE_URL` do processo principal, pois a convenção do projeto é um `PrismaClient` por processo, não compartilhável entre processos Node distintos.

A escolha de polling em vez de um mecanismo reativo foi motivada pela limitação técnica do MySQL (ausência de notificação nativa a processos externos) combinada com o fato de que um intervalo de 2 segundos já atende com folga o requisito de negócio de latência abaixo de 10 segundos, tornando desnecessária a complexidade adicional de um mecanismo reativo. A separação em processo distinto do da API elimina o acoplamento entre o ciclo de vida do worker e o ciclo de vida/deploy da API.

## Alternativas Consideradas

### 1. Mecanismo reativo via trigger de banco de dados

- **Prós:**
  - Potencial de latência de entrega mais próxima de zero, sem espera de intervalo de polling.
  - Evita execução periódica de queries em tabelas eventualmente vazias.
- **Contras:**
  - MySQL não possui mecanismo nativo equivalente ao `LISTEN/NOTIFY` do PostgreSQL para notificar processos externos.
  - Triggers de banco só conseguem executar SQL, não acionar um processo fora do banco.
  - Exigiria soluções paliativas (escrever em arquivo, chamar um endpoint HTTP a partir do trigger) consideradas frágeis e fora do padrão pela equipe.
  - Complexidade adicional não se justifica frente a um requisito de latência (<10s) já atendido com folga pelo polling de 2s.

### 2. Worker executado dentro do mesmo processo da API

- **Prós:**
  - Elimina a necessidade de uma segunda instância de `PrismaClient` e de um segundo processo a operar.
  - Simplifica o pipeline de deploy no curto prazo (um único artefato/processo).
- **Contras:**
  - Qualquer reinício ou deploy da API interrompe a entrega de webhooks pendentes, quebrando a garantia de resiliência esperada.
  - Acopla a disponibilidade da entrega de webhooks à disponibilidade e ao ciclo de release da API.
  - Impede escalar ou operar (start/stop, monitorar) os dois workloads de forma independente.

## Consequências

A decisão introduz, pela primeira vez no projeto, uma topologia de dois processos coordenados apenas pelo banco de dados compartilhado, em vez de um processo único. Isso traz resiliência: uma reinicialização ou deploy da API não interrompe a entrega de eventos pendentes, e o pior caso de latência (2 segundos) permanece confortavelmente dentro do requisito de negócio de menos de 10 segundos. Também preserva o reuso de padrões já estabelecidos no projeto (estrutura de entry-point de processo, `PrismaClient`, tratamento de erros), reduzindo a curva de aprendizado da equipe.

Em contrapartida, a equipe aceita a complexidade operacional de administrar um segundo processo de longa duração — com seu próprio ciclo de vida, reconexão de banco e necessidade de supervisão em caso de falha —, algo inédito no projeto até então. A decisão também aceita explicitamente uma limitação de ordering: a garantia de entrega na ordem correta só existe por `order_id` e apenas enquanto houver um único worker em execução; evoluir para múltiplos workers em paralelo exigirá resolver particionamento ou lock pessimista, problema declarado como futuro e deliberadamente fora do escopo desta decisão.

[PRECISA DE INFORMAÇÃO: Não há definição, nem na transcrição nem no código, do mecanismo de supervisão/restart do processo do worker em caso de crash (ex.: gerenciador de processos, orquestrador, health check) — a equipe reconheceu a necessidade de tratar "erros não capturados" no ciclo de vida do worker, mas não fechou como isso será operacionalizado.]

## Referências

- `TRANSCRICAO.md` `[09:09]`-`[09:13]` — debate e fechamento da decisão de polling de 2s versus mecanismo reativo, e da limitação de ordering em cenário single-worker.
- `TRANSCRICAO.md` `[09:11]` — exigência de processo separado da API e proposta concreta de `src/worker.ts` / `npm run worker`.
- `TRANSCRICAO.md` `[09:29]`-`[09:30]` — decisão sobre instância própria de `PrismaClient` por processo, mesma `DATABASE_URL`.
- `src/config/database.ts:4-9` — padrão atual de `PrismaClient` singleton por processo, base técnica da necessidade de uma instância própria no worker.
- `src/server.ts` — único entry-point de processo existente hoje, referência estrutural para o novo `src/worker.ts`.
