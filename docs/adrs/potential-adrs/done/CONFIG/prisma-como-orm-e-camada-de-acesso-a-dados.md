# ADR em Potencial: Prisma como ORM e Camada Única de Acesso a Dados

**Módulo**: CONFIG
**Categoria**: Arquitetura (ORM / Camada de Acesso a Dados) — Etapa 0, Categoria 3
**Prioridade**: Obrigatório Documentar (Pontuação: 140/150)
**Data de Identificação**: 2026-09-01

---

## Contexto de ADR Existente

ℹ️ **DECISÕES RELACIONADAS** (similaridade média, não duplicata)

Esta decisão está relacionada a duas ADRs potenciais já criadas em módulos analisados anteriormente, mas trata de um objeto diferente — a *escolha do ORM e o padrão de instanciação* (singleton por processo), não o *uso* desse ORM em um fluxo de negócio específico:

- **ORDERS / "Extensão Transacional de `OrderService.changeStatus` para Publicação Atômica de Eventos (Outbox)"** (135/150) — consome a API `prisma.$transaction` / `Prisma.TransactionClient` que só existe porque o projeto adotou Prisma como ORM. Aquela ADR assume o Prisma como dado; esta ADR documenta a escolha em si.
- **WEBHOOKS / "Worker de Entrega em Processo Separado com Polling de 2 Segundos"** (130/150) — já registra, como evidência de suporte (não como decisão própria), que o worker precisará de sua própria instância de `PrismaClient` por ser um processo Node distinto (`[09:29]-[09:30]`), citando exatamente `src/config/database.ts` como a origem do padrão "singleton por processo". Esta ADR de CONFIG é o local estrutural correto para a decisão de origem; a ADR de WEBHOOKS trata apenas da sua *consequência* para o worker, e assim permanece — ver seção "ADRs Potenciais Relacionadas" abaixo para a divisão de responsabilidade entre as duas.

**Linha do tempo**: não há data de introdução distinta — `src/config/database.ts` e as ADRs de ORDERS/WEBHOOKS são, segundo o Git, do mesmo commit único de inicialização (`7ef4317`, 2026-06-24). A reunião de `TRANSCRICAO.md` (webhooks) é posterior e não questiona a escolha do Prisma; apenas herda e estende o padrão de instância única por processo.

**Nota de escopo**: as análises dos módulos **MIDDLEWARES** (linha 102 de `potential-adrs-index.md`) e **PRODUCTS** (linha 125) já haviam identificado que a escolha do Prisma é de projeto inteiro, estruturalmente ligada a `src/config/database.ts`, e recomendaram explicitamente avaliá-la aqui, em CONFIG. Esta ADR potencial resolve esse encaminhamento.

---

## O Que Foi Identificado

O projeto usa **Prisma Client 5.22.0** como única camada de acesso ao MySQL em todo o sistema. A instanciação é centralizada em `src/config/database.ts`: uma função `createPrismaClient()` configura o nível de log conforme `NODE_ENV` e o resultado é exportado como **singleton** (`export const prisma: PrismaClient = createPrismaClient()`), instanciado uma única vez por processo Node. Esse singleton é então injetado manualmente — sem container de DI — em `buildControllers(prisma)` (`src/app.ts`), que o repassa para cada `*Repository` de cada módulo (`UserRepository`, `CustomerRepository`, `ProductRepository`, `OrderRepository`) e para o próprio `OrderService`, que o usa diretamente para abrir transações (`prisma.$transaction`).

Não há indício, em `TRANSCRICAO.md`, de que a escolha do Prisma como ORM tenha sido debatida nesta reunião — a reunião trata exclusivamente da feature de webhooks, e todas as menções a Prisma nela (`[09:07]`, `[09:11]`, `[09:29]-[09:30]`) tratam o ORM como infraestrutura **já estabelecida e não questionada**, a ser reaproveitada pelo novo worker. Isso é classificado com honestidade: não existe uma "decisão confirmada de adotar Prisma" nesta transcrição — o que existe é uma **decisão confirmada e estruturalmente relevante que estende o padrão do ORM**: em `[09:29]-[09:30]`, Diego pergunta se o worker abre "o mesmo PrismaClient ou um separado", e Bruno responde e a equipe confirma: *"Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node"*. Essa é uma decisão **confirmada** (não apenas proposta) sobre como o padrão de singleton de `src/config/database.ts` se aplica à nova topologia de dois processos que a feature de webhooks introduz — a primeira vez que o projeto terá mais de um processo de longa duração usando Prisma.

A pontuação e a prioridade desta ADR se apoiam primariamente em **evidência de código** (padrão maduro, usado por 100% dos módulos, categoria universal da Etapa 0), com a extensão confirmada em `[09:29]-[09:30]` como reforço de que a equipe entende e aplica deliberadamente as implicações operacionais dessa escolha ao planejar a nova feature — não como se a escolha do Prisma em si tivesse sido decidida nesta reunião.

## Por Que Isso Pode Merecer uma ADR

- **Impacto**: É a única via de acesso a dados de todo o sistema — todos os 5 módulos de negócio existentes (`AUTH` indiretamente via `UserRepository`, `USERS`, `CUSTOMERS`, `PRODUCTS`, `ORDERS`) dependem do mesmo singleton `prisma`. A feature de webhooks, ao introduzir um segundo processo (`src/worker.ts`), estende esse padrão pela primeira vez para uma topologia multi-processo.
- **Trade-offs**: Prisma acopla o schema de dados a um DSL próprio (`schema.prisma`) e a migrations geradas por ferramenta, em troca de tipagem forte end-to-end e da API de transação (`$transaction`) usada como base para a extensão transacional do outbox de webhooks (ADR relacionada de ORDERS). O padrão de singleton por processo (em vez de um pool gerenciado externamente ou de instâncias por requisição) é simples, mas exige disciplina manual quando a topologia de processos muda — como ficou explícito na decisão do worker.
- **Complexidade**: Baixa complexidade de uso (Client gerado, tipado), mas a decisão de instanciação (singleton por processo, sem container de DI, injeção manual via `buildControllers`) é um padrão que qualquer novo processo de longa duração (como o worker) precisa replicar corretamente — o que a reunião de fato faz ao decidir explicitamente por uma segunda instância separada.
- **Conhecimento da Equipe**: Essencial para qualquer trabalho de camada de dados — todo repository do projeto e todo `*.service.ts` que abre transação depende de entender o singleton e a API do Prisma Client.
- **Implicações Futuras**: Migrar de Prisma para outro ORM/query builder seria uma mudança de altíssimo custo (reescrita de todos os `*Repository`, do `schema.prisma`, das migrations, e de todo código que usa `Prisma.TransactionClient` como tipo, incluindo a futura função `publishWebhookEvent(tx, ...)` do outbox). A decisão de manter um `PrismaClient` por processo (em vez de, por exemplo, compartilhar conexão entre processos por algum outro mecanismo) também define o modelo mental de "cada processo Node gerencia seu próprio pool de conexão com o mesmo banco" — relevante para qualquer processo adicional que o projeto venha a introduzir no futuro além da API e do worker.
- **Contexto Temporal**: Estável desde a inicialização do repositório (commit único); sem histórico de mudança de ORM ou de padrão de instanciação.

## Evidências Encontradas na Base de Código

### Arquivos Principais
- [`src/config/database.ts`](../../../../../src/config/database.ts) — `createPrismaClient()` e export do singleton `prisma`.
- [`src/config/env.ts`](../../../../../src/config/env.ts) — schema Zod que valida `DATABASE_URL` (consumida indiretamente pelo Prisma Client via variável de ambiente, não passada explicitamente ao construtor).
- [`src/app.ts`](../../../../../src/app.ts) — `buildControllers(prisma: PrismaClient)` injeta manualmente o singleton em cada `*Repository`/`*Service` de cada módulo (linhas 26-43 aproximadamente).
- [`prisma/schema.prisma`](../../../../../prisma/schema.prisma) — define o `provider = "mysql"` e todos os models (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`) que o Prisma Client tipa automaticamente.
- [`src/modules/orders/order.service.ts`](../../../../../src/modules/orders/order.service.ts) — maior consumidor da API estendida do Prisma (`prisma.$transaction`, `Prisma.TransactionClient`), ponto de integração central da futura feature de webhooks.

### Evidência no Código
```typescript
// src/config/database.ts
import { PrismaClient } from '@prisma/client';
import { env } from './env.js';

export function createPrismaClient(): PrismaClient {
  return new PrismaClient({
    log: env.NODE_ENV === 'development' ? ['warn', 'error'] : ['error'],
  });
}

export const prisma: PrismaClient = createPrismaClient();
```
```typescript
// src/app.ts (injeção manual, sem container de DI)
export function buildControllers(prisma: PrismaClient): Controllers {
  const userRepository = new UserRepository(prisma);
  // ... um repository por módulo, todos recebendo a mesma instância singleton
  const orderService = new OrderService(orderRepository, prisma);
  // ...
}
```
A ausência de um container de injeção de dependências (Awilix, InversifyJS, NestJS DI, etc.) é uma característica direta desta mesma decisão — o singleton é passado manualmente "à mão" por toda a árvore de construção de objetos. Não é tratada aqui como uma ADR separada (ver Sinal de Alerta 5, seção "Observações Adicionais"): é um detalhe de como o singleton de Prisma é distribuído, não uma decisão de arquitetura de injeção de dependências independente.

### Evidência na Transcrição
- `[09:07]` Bruno (pergunta sobre performance): menciona indiretamente a dependência do Prisma para o volume de eventos na outbox.
- `[09:11]` Bruno: "Pode ser, mas vai precisar conectar no mesmo banco e usar o mesmo Prisma client" — ao discutir onde rodaria o worker, trata o Prisma como infraestrutura compartilhada já dada.
- `[09:29]` Diego: "Sobre infraestrutura compartilhada: o pool de conexão do Prisma já tá lá. O worker abre o mesmo PrismaClient ou um separado?" — pergunta que força a decisão explícita sobre a extensão do padrão de singleton para dois processos.
- `[09:30]` Bruno: "Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node." — **decisão confirmada** sobre a aplicação do padrão de CONFIG à nova topologia multi-processo.
- `[09:30]` Larissa: "Decisão: reuso máximo do que já existe" — fecha o bloco confirmando reuso deliberado da infraestrutura de dados existente (Prisma incluso) para o módulo de webhooks.

### Análise de Impacto (Git)
- `git log --follow --format='%ai|%s' -- src/config/` retorna apenas `2026-06-24 15:41:54 -0300|init repository` — commit único de inicialização do repositório (`7ef4317`), sem granularidade temporal incremental. Consistente com o padrão já registrado em todas as demais análises desta Fase 2 (AUTH, ORDERS, SHARED, MIDDLEWARES, PRODUCTS, USERS).
- Não há, portanto, evidência de git sobre motivação histórica (ex.: migração de outro ORM) — a escolha do Prisma é anterior a qualquer histórico versionado disponível neste repositório.

### Alternativas (se observáveis)
Nenhuma alternativa de ORM (TypeORM, Sequelize, Knex, Drizzle, `mysql2` cru) é mencionada em comentários de código, configuração ou em `TRANSCRICAO.md`. A única "alternativa" implícita e explicitamente descartada na transcrição é sobre a **instanciação**, não sobre o ORM em si: usar o **mesmo** `PrismaClient` do processo HTTP dentro do worker — descartada em `[09:30]` em favor de uma instância separada por processo.

## Questões a abordar na ADR (se criada)

- Qual foi o racional original para adotar Prisma como ORM único (produtividade, tipagem end-to-end, developer experience, migrations declarativas) — não documentado em nenhuma fonte disponível a este projeto?
- Por que um singleton por processo (em vez de um pool de conexão gerenciado externamente, ou instâncias por requisição)?
- Quais são os limites desse padrão quando o número de processos de longa duração cresce além de API + worker (ex.: se o projeto vier a ter mais processos no futuro)?
- Como o padrão `Prisma.TransactionClient` (usado em `OrderService.changeStatus` e proposto para `publishWebhookEvent(tx, ...)`) deve ser tratado como contrato interno estável entre módulos?

## ADRs Potenciais Relacionadas
- ORDERS / Extensão Transacional de `OrderService.changeStatus` para Publicação Atômica de Eventos (Outbox) — consumidor direto da API de transação do Prisma.
- WEBHOOKS / Worker de Entrega em Processo Separado com Polling de 2 Segundos — aplica a extensão confirmada em `[09:29]-[09:30]` (instância própria de `PrismaClient` para o novo processo); esta ADR de CONFIG é a origem estrutural do padrão, aquela é sua consequência aplicada à nova topologia. Recomenda-se, na ADR formal, que a ADR de CONFIG seja referenciada como "ADR base" pela ADR do worker, evitando duplicar a explicação do padrão singleton-por-processo em ambos os documentos.

## Observações Adicionais

- **Ausência de container de DI (injeção manual em `buildControllers`)**: avaliada e **não elevada a ADR separada** (Sinal de Alerta 5 — granularidade excessiva / acoplada à mesma decisão maior). É a forma como o singleton de Prisma é distribuído pela árvore de objetos do projeto, não uma decisão de arquitetura de injeção de dependências independente e debatida em nenhuma fonte. Consolidada como evidência de suporte dentro desta ADR (seção "Evidência no Código").
- **`src/config/env.ts` (validação Zod fail-fast de variáveis de ambiente, `process.exit(1)` em config inválida)**: avaliada separadamente e **descartada** como ADR própria de CONFIG — ver "Notas do Processo (CONFIG)" em `potential-adrs-index.md` para o racional completo (falha parcialmente o critério "Evidente" dos 3 E's; implementação trivial em arquivo único; zero menção em `TRANSCRICAO.md`; a única evidência de transcrição tocando `env.ts` — validação de `JWT_SECRET` — já foi consolidada dentro da ADR potencial de AUTH como detalhe de configuração, não como decisão de CONFIG).
- A decisão confirmada em `[09:29]-[09:30]` sobre o `PrismaClient` separado do worker é mencionada nesta ADR como *contexto e extensão direta* do padrão de CONFIG, mas a decisão "Worker em Processo Separado" em si — incluindo o polling de 2s e a razão para dois processos — permanece integralmente na ADR potencial de WEBHOOKS, evitando duplicação de conteúdo entre os dois arquivos.
