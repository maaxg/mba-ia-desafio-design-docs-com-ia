# ADR-001 — Padrão Outbox no MySQL existente para eventos de webhook

- **Data:** reunião técnica de quinta-feira (09:00–09:53)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-snapshot-na-insercao.md)

## Status

Aceito.

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados quando o status dos seus pedidos muda, em vez de fazer polling em `GET /orders` ([09:00] Marcos).

A mudança de status hoje acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que atualiza `orders`, insere em `order_status_history` e debita/repõe `stock_quantity` dos produtos. Bruno descreveu essa transação como "já pesada" ([09:04]).

Duas forças precisam ser conciliadas:

1. **Não acoplar a disponibilidade do cliente à mudança de status.** Uma chamada HTTP dentro da transação faria um cliente lento travar mudanças de status de outros pedidos, e um cliente fora do ar não justifica rollback da mudança de status ([09:04] Bruno).
2. **Não perder eventos.** Se o status mudou, o evento tem que existir; se a transação foi desfeita, o evento não pode existir ([09:06] Diego, [09:40] Bruno).

O projeto não tem nenhuma infraestrutura de mensageria hoje: só MySQL via Prisma (`prisma/schema.prisma`, `docker-compose.yml`).

## Decisão

Adotar o **padrão Transactional Outbox** usando o MySQL que a aplicação já usa:

- Criar a tabela `webhook_outbox`.
- Dentro da **mesma transação** de `changeStatus`, inserir uma linha por evento a entregar. Se a inserção falhar, a transação inteira faz rollback ([09:40] Bruno).
- Um worker separado lê a tabela e faz as chamadas HTTP (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).
- A tabela tem um campo de status (pendente, processando, falhou, entregue) com índices em `status` e `created_at` ([09:08] Diego).
- O filtro de eventos por endpoint é aplicado **na inserção**: se nenhum webhook ativo do customer assina o `to_status`, nenhuma linha é criada ([09:34] Bruno).
- Chave primária UUID, como no resto do schema ([09:51] Larissa).

Decisão registrada por Larissa em [09:08] ("Tá decidido então: outbox em MySQL") e confirmada no resumo de [09:48].

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo HTTP síncrono dentro de `changeStatus`** | Acopla a latência e a disponibilidade do cliente à transação de status. Um cliente lento trava mudanças de outros pedidos, e uma falha do cliente forçaria a escolha entre rollback (inaceitável) e perda do evento ([09:04] Bruno, [09:06] Diego: "Síncrono está fora de questão"). |
| **Redis Streams (ou fila similar)** | Exige infraestrutura nova (Redis Cluster) para um time pequeno, o que Diego chamou de overengineering ([09:07] Larissa, Diego). Argumento complementar, não citado na reunião: também não resolve a atomicidade entre o commit no MySQL e a publicação na fila (dual write). |

## Consequências

**Positivas**
- Atomicidade: o evento é registrado se e somente se a mudança de status é commitada ([09:06] Diego).
- Nenhuma infraestrutura nova: mesmo MySQL, mesmo Prisma, mesmo backup.
- A falha ou lentidão de um cliente não afeta a API de pedidos.
- A tabela vira trilha de auditoria natural do que foi enfileirado.

**Negativas / trade-offs**
- **Mais trabalho na transação crítica:** `changeStatus` passa a ler os endpoints do customer e inserir N linhas. Aceitamos esse custo extra em troca da garantia de consistência.
- **Crescimento da tabela:** linhas entregues se acumulam. O arquivamento (≈30 dias) foi citado, mas está **fora do escopo** desta feature ([09:08] Diego).
- **Latência mínima limitada pelo polling** (ver ADR-002), e não pelo push.
- **Carga extra no banco principal:** o polling do worker concorre com a API pelo mesmo MySQL.
