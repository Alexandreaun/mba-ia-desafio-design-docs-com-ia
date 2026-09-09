# ADR-001: Estratégia de Autenticação JWT Stateless com Hash de Senha via bcrypt

**Status:** Aceita
**Date:** 2026-06-24

## Status

Aceita. O mecanismo está implementado e em uso ativo em todos os módulos protegidos do sistema desde a inicialização do projeto (commit único `7ef4317`, 2026-06-24), sem histórico de commits incrementais subsequentes que indiquem revisão da abordagem. Na reunião de refinamento da feature de Webhooks, o JWT/RBAC existente foi reutilizado sem qualquer questionamento — inclusive para proteger o futuro endpoint administrativo de replay de DLQ (`[09:35]-[09:36]`). Não há registro de debate, revisão ou plano de substituição desta decisão; ela é tratada pelo time como infraestrutura consolidada.

## Contexto

O OMS precisa autenticar usuários internos (operadores e administradores) em toda a API e aplicar controle de acesso baseado em papel (RBAC) a rotas sensíveis, sem introduzir um serviço de sessão dedicado para um sistema de porte relativamente contido. A implementação atual resolve isso emitindo um JSON Web Token assinado com segredo compartilhado no login (`AuthService.login`, `src/modules/auth/auth.service.ts:31-45`), validando a senha via `bcrypt.compare` contra o hash armazenado, e embutindo o papel do usuário (`ADMIN`/`OPERATOR`) diretamente no claim do token — eliminando qualquer consulta a sessão ou permissão em tempo de execução. Esse mecanismo é consumido de forma transversal por `authenticate`/`requireRole` (`src/middlewares/auth.middleware.ts:27-61`), que hoje protegem as rotas de USERS, CUSTOMERS, PRODUCTS e ORDERS.

`TRANSCRICAO.md` não registra nenhum debate sobre a escolha do JWT, do bcrypt ou do modelo de RBAC binário — a reunião analisada trata exclusivamente da feature de Webhooks, e o JWT aparece apenas como infraestrutura pré-existente sendo reaproveitada sem questionamento (`[09:31]-[09:32]`, `[09:35]-[09:36]`). Isso evidencia uma decisão fundacional estável, mas também expõe uma lacuna: não há evidência de que o time tenha validado se o modelo binário de papéis atende necessidades futuras, como o possível escopo por `customer_id` nos endpoints de configuração de webhook mencionado na mesma reunião (`[09:31]-[09:32]`). [NEEDS INPUT: Não há registro na transcrição nem no código de que o RBAC binário ADMIN/OPERATOR tenha sido avaliado frente a necessidades futuras de permissão granular por cliente B2B; falta confirmação do time de produto/segurança.]

## Decisão

O sistema adota autenticação stateless via JWT, com verificação de senha via `bcrypt` e emissão de token assinado por um segredo único validado no boot (mínimo de 16 caracteres, fail-fast via Zod — `src/config/env.ts:8`). O papel do usuário é embutido diretamente no payload do token (`{ sub, email, role }`), dispensando consulta a store de sessão ou serviço de permissões em runtime, o que favorece simplicidade operacional e escalabilidade horizontal — qualquer instância da API valida o token localmente, sem estado compartilhado entre processos.

Como consequência direta desse design, não há refresh token nem mecanismo de revogação/blacklist: o token permanece válido até expirar naturalmente (`JWT_EXPIRES_IN`, padrão de 8 horas). O time aceita a reautenticação periódica como custo em troca da simplicidade do design, mas essa troca não está formalmente registrada em nenhuma discussão — apenas inferida pelo estado estável e não questionado do código.

## Alternativas Consideradas

### 1. Sessão server-side com store compartilhado (ex.: Redis)

Prós:
- Permite revogação imediata de sessões comprometidas ou de usuários com papel alterado.
- Centraliza o estado de autenticação, facilitando auditoria de sessões ativas.

Contras:
- Introduz um componente de infraestrutura adicional e um ponto único de falha/latência em toda validação de request.
- Quebra o modelo stateless que hoje simplifica a escalabilidade horizontal da API.
- Exigiria alterar `auth.middleware.ts` e toda a cadeia de validação de todos os módulos protegidos.

### 2. Provedor de identidade externo (IdP), ex. Auth0/Cognito

Prós:
- Delega gestão de credenciais, MFA e rotação de segredo a um serviço especializado.
- Reduz a superfície de código de autenticação mantida internamente (`bcrypt`, `auth.service.ts`).

Contras:
- Adiciona dependência de terceiro e risco de vendor lock-in a um sistema atualmente autocontido.
- Exigiria reescrever a emissão e verificação de token em todos os pontos que hoje dependem apenas de `JWT_SECRET` local.

[NEEDS INPUT: Não há menção na transcrição nem indício no código de que estas alternativas tenham sido de fato avaliadas pelo time; a listagem acima é uma inferência técnica plausível para o cenário, não uma alternativa formalmente debatida em reunião.]

## Consequências

O design stateless traz simplicidade operacional relevante: não há infraestrutura extra de sessão para manter, e a validação local do token permite escalar a API horizontalmente sem coordenação entre instâncias. O reuso consistente de `authenticate`/`requireRole` por todos os módulos protegidos minimiza a superfície de código de autenticação e facilita o entendimento do mecanismo por qualquer engenheiro que crie um novo endpoint.

O trade-off aceito é a perda de controle fino sobre sessões ativas: um token comprometido ou um usuário com papel alterado permanece válido até a expiração natural (até 8h por padrão), sem possibilidade de revogação antecipada. [NEEDS INPUT: Não está claro se a ausência de refresh token e de revogação foi uma escolha deliberada de simplicidade aceita pela liderança técnica, ou uma lacuna de segurança ainda não endereçada — nem o código nem a transcrição esclarecem esse ponto.] Além disso, diferente do mecanismo HMAC de webhooks, que decidiu por rotação de secret com grace period de 24h (`[09:21]` Sofia), não há estratégia de rotação equivalente para `JWT_SECRET` na implementação atual. [NEEDS INPUT: Falta definição de política de rotação para o `JWT_SECRET` e de plano de resposta em caso de comprometimento do segredo — tema não coberto nem no código nem na reunião analisada.]

## Referências

- `src/modules/auth/auth.service.ts:31-50` — login, verificação de senha via bcrypt, emissão do JWT
- `src/middlewares/auth.middleware.ts:27-61` — verificação do token e enforcement de RBAC (`authenticate`, `requireRole`)
- `src/config/env.ts:3-10` — validação de `JWT_SECRET`/`JWT_EXPIRES_IN` via Zod, fail-fast no boot
- `src/modules/auth/auth.schemas.ts:3-13` — modelo de papéis `ADMIN`/`OPERATOR`
- `TRANSCRICAO.md` `[09:31]-[09:36]` — reuso do JWT/RBAC existente na feature de Webhooks, sem questionamento do mecanismo
