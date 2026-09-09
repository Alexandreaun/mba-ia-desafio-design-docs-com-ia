# ADR em Potencial: Worker de Entrega em Processo Separado com Polling de 2 Segundos

**Módulo**: WEBHOOKS
**Categoria**: Arquitetura (Topologia de Processos / Infraestrutura)
**Prioridade**: Obrigatório Documentar (Pontuação: 130/150)
**Data de Identificação**: 2026-08-31

---

## O Que Foi Identificado

A equipe decidiu que a leitura e o envio dos eventos registrados na `webhook_outbox` (ver ADR potencial do padrão Outbox) serão feitos por um **processo Node.js separado do processo HTTP principal**, com um novo entry-point `src/worker.ts` e script `npm run worker`, em vez de rodar dentro da mesma instância da API. O mecanismo de leitura será **polling em loop, a cada 2 segundos**, buscando os eventos pendentes mais antigos, processando-os e marcando-os como entregues.

Esta é uma decisão de topologia de infraestrutura inédita no projeto: hoje existe apenas um processo de longa duração (`src/server.ts`); a feature introduz o segundo. O worker precisará de sua própria instância de `PrismaClient` (mesmo banco, mesma `DATABASE_URL`, mas processo distinto), já que a convenção atual do projeto é um `PrismaClient` singleton por processo (`src/config/database.ts`). Como o módulo webhooks ainda não existe, não há `src/worker.ts` no repositório hoje — toda a evidência vem de `TRANSCRICAO.md`.

## Por Que Isso Pode Merecer uma ADR

- **Impacto**: Muda a topologia de deployment/operação do sistema pela primeira vez (de um processo único para dois processos coordenados pelo mesmo banco). Afeta pipeline de deploy, monitoramento e runbooks operacionais, não apenas o código do módulo.
- **Trade-offs**: Polling foi escolhido explicitamente em vez de um mecanismo reativo (trigger de banco) por limitação técnica do MySQL (sem `LISTEN/NOTIFY`); o intervalo de 2s foi calibrado contra o requisito de negócio de latência "abaixo de 10 segundos" comunicado pelos clientes.
- **Complexidade**: Introduz a necessidade de gerenciar ciclo de vida de um processo adicional (start/stop, reconexão de banco, tratamento de erros não capturados) — algo que não existe hoje em nenhuma parte do projeto.
- **Conhecimento da Equipe**: Times de operações/infra e qualquer engenheiro depurando atraso ou falha de entrega precisam saber que existe um segundo processo rodando separadamente da API, com seu próprio ciclo de vida e necessidade de `PrismaClient` próprio.
- **Implicações Futuras**: A decisão de manter um único worker (não escalado horizontalmente por ora) implica a limitação de ordering documentada na reunião; qualquer evolução futura para múltiplos workers exigirá revisitar tanto esta ADR quanto a de particionamento/lock mencionada como problema em aberto.
- **Contexto Temporal**: Decisão fechada nesta única reunião; sem histórico de código anterior (worker ainda não implementado).

## Evidências Encontradas na Base de Código

### Arquivos Principais
- `src/server.ts` — único entry-point de processo existente hoje no projeto; referência do padrão que `src/worker.ts` deverá seguir (mencionado explicitamente por Larissa em `[09:11]`: "Tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em src/server.ts").
- [`src/config/database.ts`](../../../../../src/config/database.ts) — `createPrismaClient()` e singleton `prisma` por processo; confirma a premissa técnica discutida em `[09:29]-[09:30]` de que o worker precisará de sua própria instância (`PrismaClient` é por processo).
- `package.json` — hoje contém apenas scripts `dev`/`build`/`start` apontando para `src/server.ts`; não existe ainda um script `worker`.

### Evidência no Código
```typescript
// src/config/database.ts (padrão atual — um PrismaClient por processo)
export function createPrismaClient(): PrismaClient { /* ... */ }
export const prisma = createPrismaClient();
```
Este padrão de singleton por processo é a razão técnica concreta pela qual o worker, sendo um processo Node distinto, precisa instanciar seu próprio `PrismaClient` (mesmo `DATABASE_URL`), conforme decidido em `[09:29]-[09:30]`.

### Evidência na Transcrição
- `[09:09]` Diego: escolha do mecanismo de polling em vez de reativo — "Polling em loop. A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca."
- `[09:09]` Diego: justificativa técnica para descartar trigger de banco — "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres... Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito."
- `[09:10]` Marcos/Larissa: validação do intervalo contra o requisito de negócio e fechamento formal — "Vamos registrar isso como uma decisão. Worker em polling, 2s."
- `[09:11]` Diego: exigência de processo separado — "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker."
- `[09:11]` Larissa/Bruno/Diego: proposta concreta de entry-point (`src/worker.ts`, `npm run worker`) e confirmação de que compartilha banco mas não processo.
- `[09:29]-[09:30]` Diego/Bruno/Larissa: decisão sobre `PrismaClient` separado por processo, mesma `DATABASE_URL`.
- `[09:12]-[09:13]` Diego/Bruno/Larissa: consequência aceita — garantia de ordering apenas por `order_id`, apenas em cenário single-worker; escalar para múltiplos workers é problema declarado como futuro, sem solução especificada.

### Análise de Impacto (Git)
- Histórico do Git não aplicável — `src/worker.ts` ainda não existe no repositório. `git log` do repositório mostra apenas o commit único `init repository`, sem evolução incremental de `src/server.ts` que pudesse ser usada como precedente temporal.

### Alternativas (explicitamente discutidas na transcrição)
- **Trigger de banco reativo** — rejeitada (`[09:09]`): MySQL não tem mecanismo nativo de notificação a processos externos.
- **Worker rodando dentro do mesmo processo da API** — rejeitada implicitamente (`[09:11]`): risco de perda do worker em caso de reinício da API.

## Questões a abordar na ADR (se criada)

- Qual problema estava sendo resolvido (necessidade de um consumidor assíncrono e resiliente a reinícios da API)?
- Por que polling de 2s foi escolhido em vez de mecanismos reativos, dado o requisito de negócio de latência abaixo de 10s?
- Como o worker gerencia seu próprio ciclo de vida, conexão de banco e tratamento de erros?
- Quais são as consequências de longo prazo (limitação de ordering em cenário single-worker; caminho de evolução para múltiplos workers via particionamento por `order_id` ou lock pessimista, mencionado como problema futuro em `[09:13]`)?

## ADRs Potenciais Relacionadas
- Padrão Outbox no MySQL para Entrega de Eventos de Webhook (a tabela que o worker consome)
- Política de Retry com Backoff Exponencial e Dead Letter Queue (executada pelo próprio worker durante o ciclo de polling)

## Observações Adicionais

- O intervalo exato de polling (2 segundos) é um parâmetro de configuração desta decisão maior, não uma ADR isolada (Sinal de Alerta 5).
- A garantia de ordering apenas por `order_id` em cenário single-worker (`[09:12]-[09:13]`) é uma limitação/consequência desta decisão — deve constar na seção de Consequências da ADR formal e também ser registrada como Questão em Aberto no RFC (conforme já indicado em `docs/adrs/mapping.md`), não como decisão arquitetural separada.
- Item explicitamente reforçado pela reunião mas não uma decisão nova em si: reuso do `PrismaClient` e do padrão de conexão já estabelecidos no projeto (`[09:29]-[09:30]`) — tratado aqui como evidência de suporte, não como ADR própria (ver decisão descartada "Reuso de Padrões Existentes" no relatório de identificação).
