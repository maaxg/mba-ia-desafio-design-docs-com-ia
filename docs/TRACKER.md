# Tracker de Rastreabilidade

Cada item registrado nos documentos, com sua origem na transcrição (`TRANSCRICAO.md`) ou no código. Quando um item tem várias origens, a linha aponta a **principal**; as demais aparecem no próprio documento.

**Convenções**
- **Fonte:** `TRANSCRICAO` (localização `[hh:mm] Falante`) ou `CODIGO` (caminho do arquivo no repositório).
- **Decisões de implementação do FDD (🔧)** que não foram discutidas na reunião aparecem com o tipo **"Decisão de implementação (derivada)"** e apontam a fala que motivou a dedução. A lista completa está em [FDD §14](FDD.md#14-decisões-de-implementação-tomadas-neste-fdd).

## Legenda de documentos

| Sigla | Arquivo |
|---|---|
| PRD | `docs/PRD.md` |
| RFC | `docs/RFC.md` |
| FDD | `docs/FDD.md` |
| ADR-001 … ADR-008 | `docs/adrs/ADR-00N-*.md` |

## Tabela

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em `GET /orders`, o que é lento e caro | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco de negócio | Atlas ameaça migrar para concorrente se não houver entrega no trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | OMS não tem nenhum mecanismo de notificação externa | CODIGO | package.json |
| PRD-PER-01 | docs/PRD.md | Público-alvo | Usuários do sistema que representam o cliente operam a API | TRANSCRICAO | [09:32] Marcos |
| PRD-PER-02 | docs/PRD.md | Público-alvo | Administrador (role ADMIN) reprocessa falhas | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-03 | docs/PRD.md | Cenário | Cliente em manutenção planejada de 2h | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-04 | docs/PRD.md | Cenário | Secret vazada em log do cliente; rotação | TRANSCRICAO | [09:22] Diego |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência p95 < 10s ("tempo real" para o cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | 3 de 3 clientes integrados até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Zero mudanças de status sem evento registrado | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Métrica | Janela de ~15h de retentativas | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Métrica | Revisão de segurança ≥ 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail ao cliente em falhas repetidas (adiado) | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Dashboard visual (projeto do time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída ("observar e decidir depois") | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Ordem global entre pedidos | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Itens do pedido no payload | TRANSCRICAO | [09:43] Diego |
| PRD-OOS-08 | docs/PRD.md | Fora de escopo | Arquivamento de linhas após ~30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Notificar via HTTP POST a cada mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (customerId no body, url, status) — não vem do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Secret gerada pelo servidor e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Vários webhooks por customer, identificados no envio | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | 5 retentativas 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Falha permanente vai para DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Replay de DLQ por ADMIN com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Conteúdo do payload (event_id, tipo, ids, status, valores; sem itens) | TRANSCRICAO | [09:43] Diego |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Payload reflete o momento da transição (snapshot) | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Assinatura verificável e identificador para dedup | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10s; verificação a cada 2s | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Consistência status ⇔ evento | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não afeta mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | At-least-once com dedup no cliente | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Restrição | Ordem só por pedido e com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 com secret por webhook | TRANSCRICAO | [09:22] Sofia |
| PRD-NFR-07 | docs/PRD.md | Restrição | Só URLs https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-08 | docs/PRD.md | Restrição | Limite de 64 KB; erro em vez de truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por envio | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-10 | docs/PRD.md | Restrição | CRUD com qualquer role autenticada; replay só ADMIN | TRANSCRICAO | [09:37] Sofia |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Envio em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-12 | docs/PRD.md | Restrição | Sem infra nova; padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-02 | docs/PRD.md | Dependência | Confirmação do prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-RSK-01 | docs/PRD.md | Risco | Atraso na entrega / perda da Atlas | TRANSCRICAO | [09:00] Marcos |
| PRD-RSK-02 | docs/PRD.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:25] Sofia |
| PRD-RSK-03 | docs/PRD.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-04 | docs/PRD.md | Risco | Rajada de notificações sobrecarrega o cliente | TRANSCRICAO | [09:38] Diego |
| PRD-AC-10 | docs/PRD.md | Critério de aceitação | Documentação de integração publicada antes do go-live | TRANSCRICAO | [09:40] Marcos |
| PRD-TST-01 | docs/PRD.md | Estratégia de teste | Testes de integração no padrão de Vitest + Supertest existente | CODIGO | tests/orders.test.ts |
| RFC-META-01 | docs/RFC.md | Metadado | Larissa autora; revisores são os participantes da reunião | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Transação de status já atualiza orders, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| RFC-CTX-02 | docs/RFC.md | Escopo | Só outbound | TRANSCRICAO | [09:03] Sofia |
| RFC-PROP-01 | docs/RFC.md | Decisão | Visão geral: outbox + worker + HMAC + at-least-once + retry + DLQ | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-02 | docs/RFC.md | Estimativa | 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo HTTP síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila dedicada | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger do MySQL para acordar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry indefinido / só 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers | TRANSCRICAO | [09:13] Diego |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | Endurecer autorização do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | E-mail proativo ao cliente (próxima fase) | TRANSCRICAO | [09:37] Larissa |
| RFC-Q-05 | docs/RFC.md | Questão em aberto | Arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-01 | docs/RFC.md | Impacto | Novas tabelas via migration Prisma | CODIGO | prisma/schema.prisma |
| RFC-RSK-01 | docs/RFC.md | Risco | Falha na outbox derruba mudança de status (intencional) | TRANSCRICAO | [09:41] Diego |
| RFC-NEXT-01 | docs/RFC.md | Próximo passo | Sessão de revisão com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL, mesma transação de changeStatus | TRANSCRICAO | [09:08] Larissa |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | `changeStatus` em `prisma.$transaction` com update, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-D2 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Índices em status e created_at; estados pendente/processando/falhou/entregue | TRANSCRICAO | [09:08] Diego |
| ADR-001-D3 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Filtro de eventos aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-001-D4 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | PK UUID | TRANSCRICAO | [09:51] Larissa |
| ADR-001-TO | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Redis descartado por overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em polling de 2s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-D2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Processo separado, `src/worker.ts`, `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D3 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient próprio por processo, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-D3b | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | `prisma` singleton instanciado ao carregar o módulo | CODIGO | src/config/database.ts |
| ADR-002-D4 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Lógica no módulo (`webhook.processor.ts`) | TRANSCRICAO | [09:28] Bruno |
| ADR-002-D5 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Worker único; ordem só por order_id | TRANSCRICAO | [09:12] Diego |
| ADR-002-TO | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Latência mínima de 2s aceita | TRANSCRICAO | [09:10] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Backoff 1m/5m/30m/2h/12h, 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D3 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela `webhook_dead_letter` separada | TRANSCRICAO | [09:18] Diego |
| ADR-003-D4 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Replay via `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:18] Diego |
| ADR-003-ALT | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | 3 tentativas (cobre só ~30 min) | TRANSCRICAO | [09:16] Bruno |
| ADR-003-TO | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | 15h de cliente fora é aceitável | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo em `X-Signature` | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D2 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret por endpoint (url + secret + customer_id + ativo) | TRANSCRICAO | [09:21] Bruno |
| ADR-004-D4 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-D5 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | URL https validada no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-TO | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Secret não pode ser hash como `passwordHash` | CODIGO | prisma/schema.prisma |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com `X-Event-Id` (UUID gerado na outbox) | TRANSCRICAO | [09:25] Diego |
| ADR-005-D4 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Dedup é responsabilidade do cliente, documentada no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-005-TO | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Joga responsabilidade para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod | TRANSCRICAO | [09:30] Larissa |
| ADR-006-C1 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | Módulo com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-C2 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | `AppError` com statusCode + errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-C3 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | `InvalidStatusTransitionError` / `InsufficientStockError` | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-C4 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | errorMiddleware trata AppError, Zod, Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-C5 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | `requireRole` para ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-D2 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Prefixo `WEBHOOK_` nos códigos de erro | TRANSCRICAO | [09:29] Larissa |
| ADR-006-D3 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Pino, nenhuma lib nova | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D8 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | `publishWebhookEvent(tx, order, fromStatus, toStatus)` | TRANSCRICAO | [09:41] Bruno |
| ADR-006-TO | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Trade-off | ZodError sempre vira `VALIDATION_ERROR` | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Decisão | Payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Bruno |
| ADR-007-ALT | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Alternativa descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-C1 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Consequência | Pedido pode ser apagado em PENDING/CANCELLED; outbox sem FK para orders | CODIGO | src/modules/orders/order.service.ts |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Atomicidade status ⇔ evento | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-06 | docs/FDD.md | Objetivo técnico | Zero infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| FDD-EXC-01 | docs/FDD.md | Exclusão | Evento na criação do pedido não entra (só changeStatus) | TRANSCRICAO | [09:40] Bruno |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Tabela de configuração: url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Convenções de schema (uuid Char(36), @@map snake_case) | CODIGO | prisma/schema.prisma |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | Enum de status da outbox (pendente/processando/falhou/entregue) | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-04 | docs/FDD.md | Decisão de implementação (derivada) | Uma linha de outbox por evento × endpoint | TRANSCRICAO | [09:44] Sofia |
| FDD-DADOS-05 | docs/FDD.md | Decisão de implementação (derivada) | `onDelete: Cascade` para não bloquear `CustomerService.delete` | CODIGO | src/modules/customers/customer.service.ts |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro do tx de changeStatus; erro → rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Filtra endpoints inscritos; nenhum → não insere | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Decisão de implementação (derivada) | Limite de 64 KB verificado no worker | TRANSCRICAO | [09:23] Sofia |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Worker em loop de 2s lendo pendentes em lote | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Decisão de implementação (derivada) | Envio isolado por endpoint, sequencial dentro de cada um | TRANSCRICAO | [09:04] Bruno |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Retry com BACKOFF_SCHEDULE e dead letter no esgotamento | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | Replay grava replayedById e loga quem fez | TRANSCRICAO | [09:36] Sofia |
| FDD-FLUXO-09 | docs/FDD.md | Decisão de implementação (derivada) | Replay mantém o mesmo eventId | TRANSCRICAO | [09:25] Diego |
| FDD-FLUXO-10 | docs/FDD.md | Fluxo | Rotação: previousSecret válida por 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-11 | docs/FDD.md | Decisão de implementação (derivada) | Duas assinaturas em X-Signature durante o grace period | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /api/v1/webhooks` (cadastro, retorna secret) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /api/v1/customers/:customerId/webhooks` (customer no path) | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /api/v1/webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /api/v1/webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `POST /api/v1/webhooks/:id/rotate-secret` | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /api/v1/webhooks/:id/deliveries` (últimas 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (ADMIN) | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Prefixo `/api/v1` e formato de erro `{ error: { code, message } }` | CODIGO | src/app.ts |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Listagem com `paginated()` | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Corpo: event_id, event_type, timestamp, order_id, order_number, status, customer_id, *_cents | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato | Nomes dos campos *_cents e formato `ORD-NNNNNN` | CODIGO | src/modules/orders/order.service.ts |
| FDD-ERR-01 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` como detalhe de VALIDATION_ERROR | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Erro | `WEBHOOK_SECRET_REQUIRED` (invariante do worker) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` → DLQ | TRANSCRICAO | [09:24] Diego |
| FDD-ERR-05 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_TIMEOUT` → retry | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-06 | docs/FDD.md | Erro | `WEBHOOK_OUTBOX_ENQUEUE_FAILED` → rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-ERR-07 | docs/FDD.md | Erro | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` → DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-ERR-08 | docs/FDD.md | Restrição | `NotFoundError` fixa `NOT_FOUND`, por isso exige classe nova | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout HTTP de 10s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Recuperação de PROCESSING presos (gera reenvio, at-least-once) | TRANSCRICAO | [09:24] Diego |
| FDD-RES-03 | docs/FDD.md | Decisão de implementação (derivada) | Shutdown gracioso como em `src/server.ts` | CODIGO | src/server.ts |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados com Pino, sem lib nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Redaction de secret em `redactPaths` | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Log de auditoria do replay com `replayedBy` | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Alerta de idade do pendente > 10s (SLO) | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Tracing por requestId gerado no request-logger | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node ≥ 20 (fetch e AbortSignal nativos); uuid já presente | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL 8.0 | CODIGO | docker-compose.yml |
| FDD-INT-01 | docs/FDD.md | Integração | Chamada a `publishWebhookEvent` após `orderStatusHistory.create` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Filtro `events` validado com `z.nativeEnum(OrderStatus)` | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Registro em `buildControllers` | CODIGO | src/app.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Montagem das rotas em `buildApiRouter` | CODIGO | src/routes/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | `router.use(authenticate)` como em customers | CODIGO | src/modules/customers/customer.routes.ts |
| FDD-INT-06 | docs/FDD.md | Integração | `requireRole('ADMIN')` como em users | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Limpeza das novas tabelas no `beforeEach` | CODIGO | tests/setup.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Nova factory `createTestWebhook` | CODIGO | tests/helpers/factories.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Scripts `worker` / `start:worker` | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-10 | docs/FDD.md | Integração | Estrutura de `src/modules/webhooks/` | TRANSCRICAO | [09:27] Bruno |
| FDD-AC-01 | docs/FDD.md | Critério de aceite técnico | Teste de atomicidade com falha forçada | TRANSCRICAO | [09:41] Diego |
| FDD-AC-08 | docs/FDD.md | Critério de aceite técnico | OPERATOR recebe 403 no replay; ADMIN 202 | TRANSCRICAO | [09:36] Larissa |
| FDD-RSK-01 | docs/FDD.md | Risco | Outbox cresce sem arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-RSK-02 | docs/FDD.md | Risco | X-Timestamp fora da assinatura; replay detection opcional | TRANSCRICAO | [09:44] Diego |
| FDD-PLAN-01 | docs/FDD.md | Plano | Divisão por sprint (outbox/DLQ, worker/retry, CRUD + integração) | TRANSCRICAO | [09:46] Larissa |
| PRD-PER-03 | docs/PRD.md | Público-alvo | Integrador técnico dos clientes B2B | TRANSCRICAO | [09:00] Marcos |
| PRD-CEN-01 | docs/PRD.md | Cenário | Assinar só SHIPPED/DELIVERED (ilustrativo) | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário | Atualização em poucos segundos sem polling | TRANSCRICAO | [09:02] Marcos |
| PRD-CEN-05 | docs/PRD.md | Cenário | Diagnóstico pelo histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-06 | docs/PRD.md | Cenário | Reprocessamento por ADMIN após falhas | TRANSCRICAO | [09:18] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão | Registro no próprio banco (outbox) | TRANSCRICAO | [09:08] Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão | Processo separado verificando a cada 2s | TRANSCRICAO | [09:10] Larissa |
| PRD-DEC-03 | docs/PRD.md | Decisão | 5 retentativas em ~15h e fila de falhas | TRANSCRICAO | [09:17] Larissa |
| PRD-DEC-04 | docs/PRD.md | Decisão | Secret por webhook e rotação de 24h | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-05 | docs/PRD.md | Decisão | Pelo menos uma vez em vez de exatamente uma vez | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-06 | docs/PRD.md | Decisão | Reaproveitar padrões do código | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-07 | docs/PRD.md | Decisão | Conteúdo congelado no momento da mudança | TRANSCRICAO | [09:52] Bruno |
| PRD-DEC-08 | docs/PRD.md | Decisão | Ordem por pedido também nos reenvios | TRANSCRICAO | [09:13] Larissa |
| PRD-DEP-03 | docs/PRD.md | Dependência | Clientes com endpoint https e capazes de validar HMAC | TRANSCRICAO | [09:23] Sofia |
| PRD-DEP-04 | docs/PRD.md | Dependência | Banco MySQL e stack atuais | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-05 | docs/PRD.md | Dependência | Usuários internos que representam os clientes | TRANSCRICAO | [09:32] Marcos |
| PRD-DEP-06 | docs/PRD.md | Dependência | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-RSK-05 | docs/PRD.md | Risco | Cliente fora por mais de 15h | TRANSCRICAO | [09:18] Diego |
| PRD-RSK-06 | docs/PRD.md | Risco | Mudança de status falha por falha no registro da notificação | TRANSCRICAO | [09:40] Bruno |
| PRD-AC-01 | docs/PRD.md | Critério de aceitação | Webhook de SHIPPED recebe em < 10s; outros status não | TRANSCRICAO | [09:33] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de aceitação | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-03 | docs/PRD.md | Critério de aceitação | Secret só na criação e rotação | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-04 | docs/PRD.md | Critério de aceitação | Rotação: antiga e nova válidas por 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-05 | docs/PRD.md | Critério de aceitação | Reenvios 1m/5m/30m/2h/12h e fila de falhas | TRANSCRICAO | [09:17] Diego |
| PRD-AC-06 | docs/PRD.md | Critério de aceitação | OPERATOR não reprocessa; ADMIN sim, com registro | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de aceitação | Reenvio mantém o identificador do evento | TRANSCRICAO | [09:25] Diego |
| PRD-AC-08 | docs/PRD.md | Critério de aceitação | Histórico mostra resultado, status, conteúdo, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-09 | docs/PRD.md | Critério de aceitação | Atomicidade nos dois sentidos | TRANSCRICAO | [09:40] Bruno |
| PRD-TST-02 | docs/PRD.md | Estratégia de teste | Revisão de segurança de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| PRD-TST-03 | docs/PRD.md | Estratégia de teste | Worker testado com timeout e 5xx simulados | TRANSCRICAO | [09:42] Diego |
| PRD-TST-04 | docs/PRD.md | Estratégia de teste | Teste de consistência com falha forçada | TRANSCRICAO | [09:41] Diego |
| RFC-COMP-01 | docs/RFC.md | Decisão | Componente de ordem e isolamento (ADR-008) | TRANSCRICAO | [09:13] Larissa |
| RFC-API-01 | docs/RFC.md | Requisito Funcional | Configuração de webhooks por qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| RFC-API-02 | docs/RFC.md | Requisito Funcional | Histórico de entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| RFC-API-03 | docs/RFC.md | Requisito Funcional | Replay de DLQ só ADMIN, com auditoria | TRANSCRICAO | [09:36] Sofia |
| RFC-GAR-01 | docs/RFC.md | Garantia | Evento registrado ⇔ mudança commitada | TRANSCRICAO | [09:06] Diego |
| RFC-GAR-02 | docs/RFC.md | Garantia | Tolerância de ~15h ao cliente fora | TRANSCRICAO | [09:17] Marcos |
| RFC-GAR-03 | docs/RFC.md | Limite | Sem ordem global | TRANSCRICAO | [09:14] Marcos |
| RFC-RSK-02 | docs/RFC.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:26] Marcos |
| RFC-RSK-03 | docs/RFC.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| RFC-RSK-04 | docs/RFC.md | Risco | Prazo do fim de novembro | TRANSCRICAO | [09:45] Marcos |
| RFC-IMP-02 | docs/RFC.md | Impacto | Novo processo `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| RFC-NEXT-02 | docs/RFC.md | Próximo passo | Agendar a revisão de segurança | TRANSCRICAO | [09:49] Sofia |
| ADR-001-ALT1 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Disparo síncrono ("fora de questão") | TRANSCRICAO | [09:06] Diego |
| ADR-002-ALT1 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Trigger do MySQL | TRANSCRICAO | [09:09] Bruno |
| ADR-002-ALT2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Worker dentro do processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-003-D2 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Timeout de 10s conta como falha e agenda retry | TRANSCRICAO | [09:42] Diego |
| ADR-003-D5 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Contagem: 1 envio + 5 retentativas (5 intervalos, ~15h) | TRANSCRICAO | [09:17] Diego |
| ADR-003-ALT2 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT3 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | DLQ como status failed na própria outbox | TRANSCRICAO | [09:18] Diego |
| ADR-004-D3 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret gerada pelo servidor e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| ADR-004-D6 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Revisão de segurança ≥ 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| ADR-004-D7 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret cifrada em repouso (AES-256-GCM) | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D7b | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Chave validada no schema de ambiente | CODIGO | src/config/env.ts |
| ADR-004-ALT1 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa descartada | Rotação com corte imediato | TRANSCRICAO | [09:21] Sofia |
| ADR-005-ALT1 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-006-D6 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | `requireRole('ADMIN')` no replay | TRANSCRICAO | [09:36] Larissa |
| ADR-006-D7 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | UUID como PK | TRANSCRICAO | [09:51] Larissa |
| ADR-006-ALT1 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Alternativa descartada | Injetar repository no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-006-C6 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | Logger Pino com redaction | CODIGO | src/shared/logger/index.ts |
| ADR-006-C7 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | Composição manual em `buildControllers` | CODIGO | src/app.ts |
| ADR-007-D2 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Decisão | Payload sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-ALT2 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Alternativa descartada | Truncar payload acima do limite | TRANSCRICAO | [09:23] Sofia |
| ADR-008 | docs/adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md | Decisão | Ordem por (endpoint, pedido) mantida durante retries | TRANSCRICAO | [09:13] Larissa |
| ADR-008-CTX | docs/adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md | Contexto | Ordem implícita por order_id com worker único | TRANSCRICAO | [09:12] Diego |
| ADR-008-D2 | docs/adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md | Decisão | Envio isolado por endpoint (cliente lento não trava outros) | TRANSCRICAO | [09:04] Bruno |
| ADR-008-ALT1 | docs/adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md | Alternativa descartada | Bloquear o endpoint inteiro durante retry | TRANSCRICAO | [09:14] Marcos |
| ADR-008-ALT2 | docs/adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md | Alternativa descartada | Múltiplos workers particionados | TRANSCRICAO | [09:13] Diego |
| ADR-008-TO | docs/adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md | Trade-off | Lacuna detectável via from_status no payload | TRANSCRICAO | [09:43] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Latência p95 < 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Resiliência: 5 retentativas | TRANSCRICAO | [09:17] Larissa |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | HMAC-SHA256 em toda requisição | TRANSCRICAO | [09:20] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | X-Event-Id estável | TRANSCRICAO | [09:25] Diego |
| FDD-EXC-02 | docs/FDD.md | Exclusão | Webhooks inbound | TRANSCRICAO | [09:03] Sofia |
| FDD-EXC-03 | docs/FDD.md | Exclusão | E-mail ao cliente | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-04 | docs/FDD.md | Exclusão | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| FDD-EXC-05 | docs/FDD.md | Exclusão | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-06 | docs/FDD.md | Exclusão | Múltiplos workers / ordem global | TRANSCRICAO | [09:13] Diego |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados | Tabela de entregas (resultado, payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-07 | docs/FDD.md | Modelo de dados | Tabela de dead letter (payload, motivo, timestamp) | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-08 | docs/FDD.md | Modelo de dados | replayedById para auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-DADOS-09 | docs/FDD.md | Modelo de dados | Índice (webhook_id, order_id, created_at) para ordem | TRANSCRICAO | [09:13] Larissa |
| FDD-DADOS-10 | docs/FDD.md | Modelo de dados | Colunas de secret cifradas | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-12 | docs/FDD.md | Fluxo | Payload renderizado dentro do tx | TRANSCRICAO | [09:52] Larissa |
| FDD-FLUXO-13 | docs/FDD.md | Decisão de implementação (derivada) | Endpoint inativo: pendentes descartados sem DLQ | TRANSCRICAO | [09:21] Bruno |
| FDD-FLUXO-14 | docs/FDD.md | Decisão de implementação (derivada) | Redirects não seguidos | TRANSCRICAO | [09:23] Sofia |
| FDD-FLUXO-15 | docs/FDD.md | Decisão de implementação (derivada) | Replay duplicado recusado (409) | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-14 | docs/FDD.md | Contrato | Secret só nas respostas de criação e rotação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-15 | docs/FDD.md | Contrato | Exemplo de verificação para o portal | TRANSCRICAO | [09:26] Marcos |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_INVALID_EVENT_FILTER | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-10 | docs/FDD.md | Erro | WEBHOOK_ENDPOINT_INACTIVE | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-11 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-12 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED | TRANSCRICAO | [09:36] Sofia |
| FDD-ERR-13 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_HTTP_ERROR / NETWORK_ERROR → retry | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-14 | docs/FDD.md | Erro | Reuso de UNAUTHORIZED/FORBIDDEN | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-RES-04 | docs/FDD.md | Resiliência | Backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-RES-05 | docs/FDD.md | Resiliência | Fallback final: DLQ + replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | Isolamento de processo | TRANSCRICAO | [09:11] Diego |
| FDD-RES-07 | docs/FDD.md | Resiliência | Lote pequeno (50) | TRANSCRICAO | [09:08] Diego |
| FDD-OBS-07 | docs/FDD.md | Observabilidade | eventId correlaciona com o cliente | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-08 | docs/FDD.md | Observabilidade | webhookId identifica o cadastro | TRANSCRICAO | [09:44] Sofia |
| FDD-OBS-09 | docs/FDD.md | Observabilidade | `logger.child` no worker | CODIGO | src/shared/logger/index.ts |
| FDD-DEP-03 | docs/FDD.md | Dependência | HMAC via node:crypto | TRANSCRICAO | [09:20] Sofia |
| FDD-DEP-04 | docs/FDD.md | Dependência | Migration aditiva sobre o schema atual | CODIGO | prisma/migrations/20260519182739_init/migration.sql |
| FDD-DEP-05 | docs/FDD.md | Dependência | Nova variável WEBHOOK_SECRET_ENCRYPTION_KEY | CODIGO | src/config/env.ts |
| FDD-AC-02 | docs/FDD.md | Critério de aceite técnico | Filtro na inserção; 2 endpoints = 2 linhas | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-03 | docs/FDD.md | Critério de aceite técnico | Snapshot não muda após inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-AC-04 | docs/FDD.md | Critério de aceite técnico | Latência ≤ 2s + processamento | TRANSCRICAO | [09:09] Diego |
| FDD-AC-05 | docs/FDD.md | Critério de aceite técnico | Assinatura dupla no grace period | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-06 | docs/FDD.md | Critério de aceite técnico | Retry com relógio controlado; 6º envio → DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-AC-07 | docs/FDD.md | Critério de aceite técnico | > 64 KB → DLQ sem HTTP | TRANSCRICAO | [09:24] Larissa |
| FDD-AC-09 | docs/FDD.md | Critério de aceite técnico | http:// → 400 | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-10 | docs/FDD.md | Critério de aceite técnico | Secret nunca em listagens ou logs | TRANSCRICAO | [09:22] Diego |
| FDD-AC-11 | docs/FDD.md | Critério de aceite técnico | API e worker independentes | TRANSCRICAO | [09:11] Diego |
| FDD-RSK-03 | docs/FDD.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:26] Marcos |
| FDD-RSK-04 | docs/FDD.md | Risco | Retenção por pedido durante retry | TRANSCRICAO | [09:13] Larissa |
| FDD-RSK-05 | docs/FDD.md | Risco | Worker único vira gargalo | TRANSCRICAO | [09:13] Diego |
| FDD-RSK-06 | docs/FDD.md | Risco | Cliente lento atrasa os próprios eventos | TRANSCRICAO | [09:42] Diego |
| FDD-RSK-07 | docs/FDD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| FDD-RSK-08 | docs/FDD.md | Risco | Perda da chave de cifra | TRANSCRICAO | [09:21] Sofia |
| FDD-RSK-09 | docs/FDD.md | Risco | Atraso na entrega | TRANSCRICAO | [09:45] Marcos |
| FDD-INT-11 | docs/FDD.md | Integração | requestId repassado pelo controller | CODIGO | src/modules/orders/order.controller.ts |
| FDD-INT-12 | docs/FDD.md | Integração | errorMiddleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Worker usa o singleton `prisma` | CODIGO | src/config/database.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Scripts novos em package.json | CODIGO | package.json |
| FDD-INT-15 | docs/FDD.md | Integração | Schemas aplicados via `validate` | CODIGO | src/middlewares/validate.middleware.ts |

## Cobertura

| Métrica | Valor |
|---|---|
| Total de linhas | 306 |
| Fonte = TRANSCRICAO | 263 (≈ 86%), todas com `[hh:mm] Falante` conferido contra `TRANSCRICAO.md` |
| Fonte = CODIGO | 43 (≈ 14%), todos os caminhos existentes no repositório |

A lista de itens rastreados foi montada varrendo cada seção dos documentos: requisitos, decisões, alternativas, questões, contratos, erros, riscos, critérios, dependências e integrações. Itens não rastreados de propósito: exemplos de valores em payloads (UUIDs, URLs fictícias) e textos de navegação entre documentos.
