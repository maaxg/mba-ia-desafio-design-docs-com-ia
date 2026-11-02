# RFC-001 — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead), redigido a partir da reunião técnica de quinta-feira |
| **Status** | Em revisão |
| **Data** | 2026-11-02 |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |
| **Documentos irmãos** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](#7-decisões-relacionadas) · [Tracker](TRACKER.md) |

---

## 1. TL;DR

Propomos notificar sistemas externos de clientes B2B, via **webhooks HTTP outbound**, sempre que um pedido muda de status. O desenho em uma frase:

> `OrderService.changeStatus` grava o evento numa tabela **outbox no próprio MySQL**, na mesma transação da mudança de status. Um **worker separado** lê a outbox por **polling de 2s** e entrega o evento **assinado com HMAC-SHA256** (secret por endpoint). A entrega é **at-least-once** (dedup pelo cliente via `X-Event-Id`), com **5 retentativas** em backoff exponencial e, se tudo falhar, **DLQ** com replay manual por ADMIN.

Não entra infraestrutura nova: mesmo banco, mesma stack, mesmos padrões de código. A estimativa é de **3 sprints**, incluindo a revisão de segurança.

## 2. Contexto e problema

- Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram formalmente para ser notificados quando o status dos seus pedidos muda. Hoje eles fazem polling em `GET /orders`, o que é lento e caro para eles. A Atlas sinalizou que pode migrar para um concorrente se a entrega não acontecer até o fim do trimestre ([09:00] Marcos).
- "Tempo real", para esses clientes, é **qualquer coisa abaixo de 10 segundos** ([09:02] Marcos).
- O OMS **não tem nenhum mecanismo de evento, fila ou notificação externa**. A mudança de status é uma transação Prisma em `src/modules/orders/order.service.ts` que já atualiza `orders`, `order_status_history` e o estoque ([09:04] Bruno).
- O fluxo é **só outbound**: os clientes recebem, não enviam ([09:02] Marcos, [09:03] Sofia).

O problema técnico central é **publicar um efeito colateral externo (HTTP) a partir de uma transação de banco**, sem acoplar a disponibilidade do cliente à operação e sem perder nem inventar eventos.

## 3. Proposta técnica

### 3.1 Visão geral

```
 API (src/server.ts)                          Worker (src/worker.ts)
 ┌──────────────────────────────┐             ┌──────────────────────────────┐
 │ PATCH /orders/:id/status     │             │ loop a cada 2s               │
 │  └─ $transaction             │             │  ├─ lê lote de PENDING       │
 │      ├─ update orders        │   MySQL     │  ├─ assina (HMAC-SHA256)     │
 │      ├─ insert status_history│ ──────────► │  ├─ POST https://cliente     │
 │      ├─ stock (se aplicável) │ webhook_    │  ├─ 2xx → DELIVERED          │
 │      └─ publishWebhookEvent  │ outbox      │  ├─ falha → retry c/ backoff │
 │         (mesmo tx)           │             │  └─ esgotou → dead letter    │
 │ CRUD /webhooks, deliveries,  │             └──────────────────────────────┘
 │ rotação de secret, replay DLQ│
 └──────────────────────────────┘
```

### 3.2 Componentes

| Componente | Responsabilidade | Decisão |
|---|---|---|
| **Outbox transacional** | O evento é gravado na mesma transação da mudança de status. Se a outbox falhar, a mudança de status faz rollback. | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **Payload snapshot** | O JSON é renderizado na inserção, sem `items`, com limite de 64 KB. | [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) |
| **Worker separado** | Processo próprio, polling de 2s, uma única instância, ordem por `order_id`. | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| **Retry + DLQ** | 1m/5m/30m/2h/12h, depois a tabela `webhook_dead_letter` e replay manual por ADMIN. | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| **Autenticidade** | HMAC-SHA256 do corpo, secret por endpoint, rotação com grace period de 24h, apenas `https`. | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| **Ordem e isolamento** | Ordem por pedido mantida também durante o retry; um cliente lento não atrasa os outros. | [ADR-008](adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md) |
| **Semântica de entrega** | At-least-once. O `X-Event-Id` é estável entre retentativas. | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| **Encaixe no código** | Módulo `src/modules/webhooks`, `AppError` com prefixo `WEBHOOK_`, Pino, `requireRole`. | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |

### 3.3 Superfície de API (visão de alto nível)

- **Configuração de webhooks** (qualquer role autenticada): cadastrar, listar por customer, editar, remover e rotacionar a secret. O `customer_id` vem no body (cadastro) ou no path (listagem), **não do JWT**, porque o JWT é do usuário operador ([09:32] Bruno, Larissa).
- **Histórico de entregas** por webhook ([09:34] Marcos).
- **Replay da DLQ** (apenas ADMIN, com auditoria) ([09:35]–[09:36] Diego, Sofia).

Contratos, payloads, headers, modelo de dados e matriz de erros estão no [FDD](FDD.md).

### 3.4 Garantias e limites explícitos

| Garantimos | Não garantimos |
|---|---|
| Evento registrado ⇔ mudança de status commitada | Exactly-once |
| Latência típica bem abaixo de 10s (polling de 2s) | Ordem global entre pedidos |
| Ordem por `order_id` com um único worker, inclusive durante o retry | Ordem de um evento reenviado manualmente da DLQ |
| Até ~15h de tolerância a cliente fora do ar | Entrega após esgotar as retentativas (vai para a DLQ) |

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte | Origem |
|---|---|---|---|
| ALT-1 | **Disparo HTTP síncrono dentro de `changeStatus`** | Um cliente lento trava mudanças de status de outros pedidos. Um cliente fora do ar força a escolha entre rollback da mudança (inaceitável) e perda do evento. | [09:04] Bruno, [09:06] Diego |
| ALT-2 | **Redis Streams / fila dedicada** | Exige subir e operar infraestrutura nova (Redis Cluster) para um time pequeno, o que é overengineering. A outbox no MySQL existente resolve com atomicidade nativa. | [09:07] Larissa, Diego |
| ALT-3 | **Trigger do MySQL para acordar o worker** | O MySQL não tem `LISTEN/NOTIFY`. A trigger só executa SQL, então avisar o processo exigiria gambiarras (arquivo, endpoint). O polling de 2s já atende aos 10s. | [09:09] Bruno, Diego |
| ALT-4 | **Retry indefinido** ou **apenas 3 tentativas** | Retry indefinido deixa eventos pendurados para sempre. Com 3 tentativas a janela é de ~30 min, menor que indisponibilidades reais (2h). | [09:15]–[09:16] Diego, Bruno |
| ALT-5 | **Secret global da plataforma** | "Se vaza uma, vaza tudo." | [09:21] Sofia |
| ALT-6 | **Exactly-once** | Exige coordenação dos dois lados e fica muito mais complexo. At-least-once + `event_id` é o padrão de mercado (Stripe, GitHub). | [09:25] Diego |

## 5. Questões em aberto

Só entram aqui pontos **levantados na reunião** e deixados sem decisão ou adiados. Dúvidas que surgiram no detalhamento técnico foram decididas e registradas em ADR ou no FDD: a ordem durante retries no [ADR-008](adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md), a contagem de tentativas no [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) e a secret cifrada em repouso no [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md).

| # | Questão | Situação na reunião | Dono sugerido |
|---|---|---|---|
| Q1 | **Rate limiting de saída por cliente.** Se 50 pedidos de um cliente mudam de status em um minuto, enviamos 50 chamadas? | Fora do escopo. "Observar e decidir depois." | Diego ([09:38]–[09:39]) |
| Q2 | **Escalar além de um worker** (particionar por `order_id` ou lock pessimista). | Adiado: "problema do futuro". Registrado como limitação conhecida. | Diego ([09:13]) |
| Q3 | **Endurecer a autorização do CRUD de webhooks** (hoje, qualquer role autenticada). | "Por enquanto sim. Mais pra frente a gente pode endurecer." | Sofia ([09:37]) |
| Q4 | **Aviso proativo ao cliente** (e-mail após falhas seguidas). | Adiado para a próxima fase, depois de medir o impacto. | Marcos / Larissa ([09:37]) |
| Q5 | **Arquivamento da outbox** (≈30 dias). | Citado e declarado fora do escopo desta feature. | Diego ([09:08]) |

## 6. Impacto e riscos

**Impacto no sistema existente**
- `changeStatus` ganha uma leitura (endpoints do customer) e N inserções dentro da transação. É uma alteração pequena e localizada, mas toca o caminho mais crítico do OMS ([09:40] Bruno).
- Há um novo processo para implantar e monitorar (`npm run worker`) e um novo consumidor de conexões do MySQL.
- Entram quatro tabelas novas no schema Prisma (configuração, outbox, entregas, dead letter), via migration.

**Riscos principais** (a tabela completa, com probabilidade e impacto, está no [PRD](PRD.md#10-riscos-e-mitigação) e no [FDD](FDD.md#12-riscos-e-mitigação))

| Risco | Mitigação proposta |
|---|---|
| Uma falha ao inserir na outbox derruba a mudança de status | É intencional: consistência acima de disponibilidade ([09:40] Bruno, [09:41] Diego). Inserção simples, coberta por testes de integração. |
| Cliente não deduplica e processa um evento duas vezes | Documentação destacada no portal ([09:26] Marcos). `event_id` estável. |
| Vazamento de secret pelo cliente | Secret por endpoint e rotação com grace period de 24h ([09:21]–[09:22] Sofia, Diego). Do nosso lado, redaction da secret nos logs ([FDD §9](FDD.md#9-observabilidade)). |
| Prazo: a Atlas quer a entrega no fim de novembro | 3 sprints, com a revisão de segurança (≥ 2 dias úteis) reservada no fim ([09:46]–[09:47] Larissa, Sofia). |

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling de 2s](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Payload snapshot na inserção](adrs/ADR-007-payload-snapshot-na-insercao.md)
- [ADR-008 — Ordem por pedido e isolamento por endpoint](adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md)

## 8. Próximos passos

1. Sessão de revisão deste RFC com Bruno e Diego antes de começar a codar ([09:50] Larissa).
2. As questões Q1 a Q5 ficam registradas para decisão em fases futuras, conforme combinado na reunião.
3. Agendar a revisão de segurança da Sofia antes do deploy ([09:49] Sofia).
