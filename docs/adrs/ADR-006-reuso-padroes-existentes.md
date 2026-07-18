# ADR-006 — Reuso máximo dos padrões existentes do projeto

## Status

Aceito — decisão fechada na reunião técnica ([09:30] Larissa: "Decisão: reuso máximo do que já existe").

## Contexto

A codebase do OMS tem um padrão claro e consistente: cada domínio é um módulo em `src/modules/` com `controller`, `service`, `repository`, `routes` e `schemas`; erros derivam de uma classe base com código simbólico; logging, validação e tratamento de erro são centralizados. O módulo de webhooks é o primeiro a incluir também um worker assíncrono — a questão é se ele segue o padrão vigente ou introduz estrutura própria.

## Decisão

O módulo de webhooks **reutiliza integralmente os padrões existentes**, sem introduzir nada novo:

- **Estrutura de módulo:** pasta `src/modules/webhooks` com controller, service, repository, routes e schemas, igual a `src/modules/orders`, `src/modules/customers` etc. A lógica do worker vive dentro do módulo (ex.: `src/modules/webhooks/webhook.processor.ts`), com entry point separada em `src/worker.ts` espelhando `src/server.ts`.
- **Erros:** classes derivadas de `AppError` (`src/shared/errors/app-error.ts`), no estilo de `InsufficientStockError` e `InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`), com códigos simbólicos prefixados **`WEBHOOK_`** (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, …).
- **Error handling:** o middleware centralizado `src/middlewares/error.middleware.ts` já trata `AppError`, `ZodError` e erros do Prisma — pega os erros do módulo novo sem nenhuma mudança.
- **Logging:** Pino, já configurado em `src/shared/logger/index.ts` (com redação de campos sensíveis) e usado no projeto inteiro. Nada novo.
- **Validação:** schemas Zod por módulo aplicados via `src/middlewares/validate.middleware.ts` (ex.: validação de URL https no cadastro).
- **Autenticação/autorização:** `authenticate` e `requireRole` de `src/middlewares/auth.middleware.ts` (CRUD autenticado normal; replay de DLQ com `requireRole('ADMIN')`).
- **Banco:** mesmo MySQL/Prisma; IDs UUID como em todos os modelos de `prisma/schema.prisma`; o worker abre **instância própria de `PrismaClient`** (client é por processo), mesma `DATABASE_URL`.

## Alternativas Consideradas

1. **Estrutura própria para o módulo de webhooks** (ex.: subprojeto do worker com stack/convenções próprias, biblioteca nova de logging ou de erro) — descartada implicitamente na reunião: a discussão partiu de "a gente tem um padrão claro na codebase […] Webhook vai seguir igual" e cada item (erros, logger, middleware, Prisma) foi confirmado como reuso, sem nada novo a adicionar.

## Consequências

**Positivas:**

- Módulo indistinguível dos existentes: onboarding trivial para o time, revisão de código mais rápida.
- Zero dependências ou infraestrutura novas; o middleware de erro e o logger funcionam sem alteração.
- Códigos `WEBHOOK_*` seguem o contrato de erro já consumido pelos clientes da API.

**Negativas / trade-offs:**

- O padrão atual foi desenhado para módulos HTTP request/response; o worker (processo de longa duração) é o primeiro caso fora desse molde e exige adaptação pontual (entry point própria, loop de polling, shutdown gracioso) mantida dentro das mesmas convenções.
- Duas instâncias de `PrismaClient` (API e worker) contra o mesmo banco: o pool de conexões total passa a ser a soma dos dois processos.
