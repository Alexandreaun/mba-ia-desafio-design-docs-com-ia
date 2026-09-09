# ADR-002: Prisma como ORM Único e PrismaClient em Singleton por Processo

**Status:** Aceita
**Date:** 2026-06-24

## Status

Aceita. O padrão está implementado e ativo em 100% dos módulos de negócio do sistema (`AUTH`,
`USERS`, `CUSTOMERS`, `PRODUCTS`, `ORDERS`) desde a inicialização do repositório, sem histórico de
mudança de ORM. A reunião de refinamento da feature de Webhooks (`TRANSCRICAO.md`) não reabre essa
escolha — trata o Prisma como infraestrutura já estabelecida — mas confirma explicitamente, em
`[09:29]-[09:30]`, a extensão do padrão de instanciação (singleton por processo) para a nova
topologia de dois processos (API + worker), reforçando o consenso da equipe sobre a decisão.

## Contexto

O sistema é uma API de gestão de pedidos B2B que depende de acesso transacional e fortemente tipado
ao MySQL para orquestrar operações críticas, como a máquina de estados de pedidos em
`OrderService.changeStatus`, que precisa debitar estoque, atualizar status e gravar auditoria dentro
de uma única transação atômica. Não há, em nenhuma fonte disponível a este projeto — nem em
`TRANSCRICAO.md`, nem em histórico de Git anterior à inicialização do repositório —, registro do
racional original que levou à escolha do Prisma em vez de outro ORM ou query builder; a decisão é
anterior a qualquer histórico versionado e é tratada por toda a equipe como infraestrutura já dada e
não questionada. `[NEEDS INPUT: Não há registro do racional original de negócio/técnico para a
escolha do Prisma como ORM — nem TRANSCRICAO.md nem o histórico de Git anterior à inicialização do
repositório documentam essa motivação]`.

O que a reunião de refinamento efetivamente decide e confirma é a forma como esse padrão já
estabelecido se estende: a feature de Webhooks introduz, pela primeira vez no projeto, um segundo
processo Node de longa duração (`src/worker.ts`, ao lado da API em `src/server.ts`). Isso força a
equipe a decidir explicitamente como o singleton de `PrismaClient` de `src/config/database.ts` se
comporta nessa nova topologia multi-processo, o que é resolvido em `[09:29]-[09:30]`: cada processo
Node mantém sua própria instância de `PrismaClient`, ainda que apontando para o mesmo banco e a
mesma `DATABASE_URL`.

## Decisão

O projeto adota o Prisma Client como única camada de acesso a dados ao MySQL, com instanciação
centralizada em `src/config/database.ts` (`createPrismaClient()`) e exportação como singleton por
processo (`export const prisma`), injetado manualmente — sem container de injeção de dependência —
em cada `*Repository` e em `OrderService` via `buildControllers(prisma)` (`src/app.ts`). A
justificativa técnica direta é a tipagem forte end-to-end gerada a partir de `prisma/schema.prisma`
e a disponibilidade da API `prisma.$transaction`/`Prisma.TransactionClient`, que é a base sobre a
qual `OrderService.changeStatus` implementa suas transações atômicas e sobre a qual a futura
extensão de outbox de webhooks pretende se apoiar.

A decisão de manter uma instância de `PrismaClient` por processo — em vez de compartilhar uma única
instância entre processos por algum mecanismo externo — é reafirmada deliberadamente pela equipe ao
planejar o worker de webhooks: cada processo Node de longa duração deve gerenciar seu próprio pool
de conexão contra o mesmo banco, um modelo mental simples que dispensa infraestrutura de coordenação
adicional, alinhado à decisão mais ampla da reunião de "reuso máximo do que já existe" (`[09:30]`
Larissa).

## Alternativas Consideradas

### Compartilhar uma única instância de PrismaClient entre processos (API e worker)
Alternativa discutida e ativamente descartada em `[09:29]-[09:30]`: em vez de cada processo Node
instanciar seu próprio `PrismaClient`, o novo worker poderia tentar reutilizar a mesma instância já
criada pelo processo da API.
- **Prós**: evitaria uma segunda inicialização de pool de conexão; conceitualmente mais simples de explicar como "um único cliente para todo o sistema".
- **Prós**: reduziria (marginalmente) o número total de conexões abertas contra o MySQL.
- **Contras**: tecnicamente inviável entre processos Node distintos sem um mecanismo externo de IPC ou pool compartilhado, que o projeto não possui.
- **Contras**: quebraria o isolamento de falhas — reinício ou travamento de um processo afetaria o outro, contrariando a própria motivação de rodar o worker separado da API (`[09:11]` Diego).

### Adotar um ORM ou query builder alternativo (TypeORM, Sequelize, Drizzle, Knex/mysql2 cru)
Nenhuma alternativa de ORM é mencionada em `TRANSCRICAO.md` ou em comentários de código; esta é uma
alternativa tecnicamente plausível para o cenário de uma API TypeScript sobre MySQL, incluída aqui
por ausência de registro da decisão original.
- **Prós**: um query builder mais leve (Knex, mysql2 cru) reduziria o acoplamento a um DSL proprietário (`schema.prisma`) e ao pipeline de geração de código do Prisma.
- **Prós**: TypeORM ou Sequelize ofereceriam padrões de Active Record/Data Mapper mais familiares a parte das equipes.
- **Contras**: perderia a tipagem end-to-end gerada automaticamente a partir do schema, hoje usada em todos os `*Repository` e na assinatura de `Prisma.TransactionClient`.
- **Contras**: migração teria custo altíssimo hoje, exigindo reescrita de todos os repositories, do schema e das migrations existentes.
- `[NEEDS INPUT: Validar se esta alternativa inferida (outro ORM/query builder) foi de fato avaliada pela equipe antes da inicialização do repositório, já que não há evidência disso em nenhuma fonte disponível]`

### Gerenciar o singleton via container de injeção de dependência (Awilix, InversifyJS)
Alternativa ao mecanismo de distribuição do singleton: em vez de injeção manual em
`buildControllers`, um container de DI poderia gerenciar o ciclo de vida do `PrismaClient` e
resolvê-lo automaticamente para cada `*Repository`/`*Service`.
- **Prós**: reduziria a repetição manual de passagem do singleton por toda a árvore de construção de objetos em `src/app.ts`.
- **Prós**: facilitaria a substituição do `PrismaClient` por mocks em testes de unidade isolados.
- **Contras**: adicionaria uma dependência e uma camada de abstração extra a um time pequeno, sem ganho funcional imediato dado o baixo número de módulos atuais.
- **Contras**: nenhuma fonte disponível (código ou transcrição) indica que essa alternativa tenha sido de fato discutida como decisão independente.

## Consequências

A escolha de Prisma como camada única de acesso a dados garante tipagem forte e consistente em todos
os módulos de negócio e fornece a base transacional (`$transaction`, `Prisma.TransactionClient`) que
sustenta tanto a máquina de estados de pedidos hoje quanto a futura extensão atômica de outbox para
webhooks. O padrão de singleton por processo, com injeção manual e sem container de DI, mantém o
projeto simples de entender para um time pequeno e foi validado como suficientemente flexível para
se estender, sem retrabalho estrutural, à primeira topologia multi-processo do sistema (API +
worker), conforme confirmado em `[09:29]-[09:30]`.

Em contrapartida, a equipe aceita um acoplamento forte ao DSL proprietário do Prisma
(`schema.prisma`) e ao seu pipeline de geração de código e migrations, o que torna uma eventual
migração para outro ORM ou query builder uma mudança de altíssimo custo, exigindo reescrita de todos
os `*Repository`, do schema e de todo código que depende de `Prisma.TransactionClient` como tipo.
Adicionalmente, o padrão de singleton por processo exige disciplina manual sempre que a topologia de
processos do sistema mudar — cada novo processo de longa duração introduzido no futuro (além de API
e worker) precisa replicar corretamente a criação de sua própria instância, sem um mecanismo
automatizado ou um container de DI que garanta isso estruturalmente.

## Referências

- `src/config/database.ts` — `createPrismaClient()` e export do singleton `prisma`.
- `src/app.ts` — `buildControllers(prisma: PrismaClient)`, injeção manual do singleton em cada módulo (linhas ~26-49).
- `src/modules/orders/order.service.ts:24,58,131` — uso de `Prisma.TransactionClient` e `prisma.$transaction` como base transacional.
- `prisma/schema.prisma` — definição do `provider = "mysql"` e dos models tipados pelo Prisma Client.
- `TRANSCRICAO.md [09:11], [09:29]-[09:30]` — debate e decisão confirmada sobre instância separada de `PrismaClient` por processo para o worker de webhooks.
