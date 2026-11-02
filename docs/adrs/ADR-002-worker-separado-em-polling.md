# ADR-002 — Worker de entrega em processo separado, lendo a outbox por polling de 2s

- **Data:** reunião técnica de quinta-feira
- **Decisores:** Larissa, Diego, Bruno; Marcos validou a latência
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-008](ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md)

## Status

Aceito.

## Contexto

Com a outbox decidida ([ADR-001](ADR-001-outbox-no-mysql.md)), falta definir **quem** lê a tabela e **como** ele percebe que há eventos novos.

- O requisito de produto é latência "abaixo de 10 segundos", que os clientes consideram tempo real ([09:02] Marcos).
- O MySQL não tem um mecanismo nativo de notificação para processos externos equivalente ao `LISTEN/NOTIFY` do Postgres. Triggers só executam SQL ([09:09] Diego).
- A API sobe por `src/server.ts`, que hoje é o único entry-point (`package.json`: scripts `dev` e `start`).

## Decisão

1. **Polling em loop a cada 2 segundos.** A cada ciclo, o worker busca os eventos pendentes mais antigos em **lote pequeno**, processa e marca o resultado ([09:08], [09:09] Diego; decisão em [09:10] Larissa).
2. **Processo separado da API.** Novo entry-point `src/worker.ts` (a criar) com script `npm run worker` ([09:11] Diego, Larissa). Se a API reiniciar, o worker não cai junto.
3. **Mesma stack e mesmo banco, com instância própria do `PrismaClient`.** Mesma `DATABASE_URL`, mas um client novo por processo ([09:11] Bruno, [09:30] Bruno). O `prisma` exportado por `src/config/database.ts` é instanciado quando o módulo é carregado, então importá-lo no worker já gera a instância daquele processo.
4. A lógica de processamento fica no módulo, no arquivo novo `src/modules/webhooks/webhook.processor.ts`. `src/worker.ts` só faz o bootstrap ([09:28] Bruno).
5. **Uma única instância do worker.** A ordem de entrega é implícita por `created_at` e, portanto, por `order_id`. Não há garantia de ordem global ([09:12] Diego, [09:13] Larissa). A forma como essa ordem é mantida durante retries, e o isolamento entre clientes dentro desse único processo, estão em [ADR-008](ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger no MySQL para tornar o worker reativo** | Trigger não notifica processo externo. Seria preciso improvisar (escrever em arquivo, chamar endpoint), o que Diego chamou de "esquisito". O polling de 2s já atende aos 10s ([09:09] Bruno, Diego). |
| **Worker dentro do processo da API** | Um restart da API derruba o worker junto ([09:11] Diego). |
| **Múltiplos workers em paralelo** | Quebra a garantia de ordem por pedido. Particionar por `order_id` ou usar lock pessimista foi adiado ("problema do futuro", [09:13] Diego). |

## Consequências

**Positivas**
- Simples de implementar, operar e depurar: é um loop com um `SELECT` indexado.
- A latência no pior caso fica bem abaixo dos 10s exigidos (≈2s de polling + tempo de envio), latência aceita explicitamente em [09:10] Larissa.
- Ciclo de vida independente da API: o deploy e o restart de um não afetam o outro.

**Negativas / trade-offs**
- **Latência mínima de até 2s**, aceita conscientemente em troca de simplicidade ([09:10] Larissa).
- **Queries contínuas no banco**, mesmo sem eventos.
- **Throughput limitado a um worker.** Escalar exige redesenho (particionamento ou lock), registrado como **limitação conhecida** ([09:13] Larissa).
- Um novo processo para implantar e monitorar.
