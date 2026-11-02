# ADR-006 — Webhooks como módulo padrão do projeto, reutilizando AppError, Pino, error middleware, Zod e requireRole

- **Data:** reunião técnica de quinta-feira
- **Decisores:** Bruno (proposta), Larissa (decisão em [09:30]), Diego
- **Relacionados:** [FDD — Integração com o sistema existente](../FDD.md#13-integração-com-o-sistema-existente)

## Status

Aceito.

## Contexto

A codebase tem convenções claras ([09:27] Bruno):

| Padrão existente | Onde está |
|---|---|
| Módulo por domínio com `controller`, `service`, `repository`, `routes`, `schemas` | `src/modules/orders/`, `src/modules/customers/` etc. |
| Erros de domínio herdando de `AppError` com `statusCode` + `errorCode` em SCREAMING_SNAKE_CASE | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`, `InsufficientStockError` → `INSUFFICIENT_STOCK`) |
| Tratamento centralizado de `AppError`, `ZodError` e erros conhecidos do Prisma | `src/middlewares/error.middleware.ts` |
| Validação de body/query/params com Zod | `src/middlewares/validate.middleware.ts` + `*.schemas.ts` |
| Autenticação JWT e autorização por role | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) |
| Logger estruturado Pino com redaction | `src/shared/logger/index.ts` |
| Composição manual de dependências e roteamento | `src/app.ts` (`buildControllers`), `src/routes/index.ts` (`buildApiRouter`) |
| IDs UUID em todas as tabelas | `prisma/schema.prisma` (`@default(uuid()) @db.Char(36)`) |

O time é pequeno ([09:07] Diego), e a reunião rejeitou introduzir peças novas: "Não vamos botar nada novo" ([09:29] Bruno).

## Decisão

**Reuso máximo do que já existe** ([09:30] Larissa):

1. Novo módulo **`src/modules/webhooks/`** (a criar) com a mesma estrutura dos demais ([09:27] Bruno), mais o `webhook.processor.ts` do worker ([09:28] Bruno).
2. **Erros do módulo herdam de `AppError`** com o **prefixo `WEBHOOK_`** (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, ...) ([09:28] Bruno, [09:29] Larissa).
3. **Pino** (`logger` de `src/shared/logger/index.ts`) em todo o módulo e no worker. Nenhuma biblioteca de log nova ([09:29] Bruno).
4. **`errorMiddleware` sem alterações:** ele já trata qualquer `AppError` ([09:29] Bruno).
5. **Schemas Zod** no padrão de `*.schemas.ts`, inclusive a regra de URL `https` ([09:23] Sofia).
6. **`requireRole('ADMIN')`**, já existente, protege o replay da DLQ ([09:36] Larissa).
7. **UUID** como PK das tabelas novas ([09:51] Larissa).
8. A integração com pedidos é feita por uma **função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`** que recebe o `Prisma.TransactionClient`, em vez de injetar um repository inteiro no `OrderService` ([09:41] Bruno, Diego).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Serviço/repositório separado (microserviço de notificações)** (não discutida na reunião; registrada para completude) | Contraria a decisão de manter "mesmo banco, mesma stack" ([09:11] Diego) e de que o webhook "fica como módulo igual aos outros" ([09:30] Larissa). Exigiria infraestrutura de deploy e comunicação nova. |
| **Injetar `WebhookRepository` no construtor de `OrderService`** | Acopla mais do que o necessário. A função pura que recebe `tx` foi preferida ([09:41] Bruno propôs; Diego: "Não precisa injetar repository inteiro"). |
| **Nova hierarquia de erros/códigos própria do módulo** (não discutida na reunião; plausível) | Quebraria o contrato de erro `{ error: { code, message, details } }` que os clientes da API já consomem via `errorMiddleware`. |

## Consequências

**Positivas**
- Curva de aprendizado zero para o time. Code review e testes seguem o mesmo molde.
- Contrato de erro HTTP idêntico ao do resto da API.
- Nenhuma dependência nova para revisar ou atualizar.

**Negativas / trade-offs**
- **Limitações herdadas:**
  - `validate.middleware.ts` converte qualquer `ZodError` em `VALIDATION_ERROR`. Por isso a recusa de URL `http` validada no schema sai como `VALIDATION_ERROR`, não como `WEBHOOK_INVALID_URL`. O FDD documenta como os dois convivem.
  - `NotFoundError` fixa o código `NOT_FOUND`. Para emitir `WEBHOOK_NOT_FOUND` é preciso criar uma classe nova em vez de reutilizar a existente.
- A observabilidade fica restrita ao que o Pino oferece (logs estruturados). Não há stack de métricas ou tracing distribuído no projeto (`package.json`).
