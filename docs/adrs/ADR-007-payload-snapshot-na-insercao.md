# ADR-007 — Payload do evento renderizado (snapshot) no momento da inserção na outbox

- **Data:** reunião técnica de quinta-feira, fim da call ([09:51]–[09:52])
- **Decisores:** Larissa, Diego, Bruno
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Status

Aceito.

## Contexto

Ao modelar a outbox, Bruno perguntou se a linha guarda o **payload já renderizado** ou só o `order_id`, para renderizar na hora do envio ([09:51]). Com retry de até ~15h ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), o pedido pode mudar várias vezes entre a inserção e a entrega.

O formato do payload foi definido em [09:43] Diego: JSON com `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order, como `total_cents`. **Sem `items`**, para não inflar. Quem quiser detalhes consulta `GET /orders/:id`.

## Decisão

**Renderizar o payload JSON completo dentro da transação de `changeStatus` e gravá-lo na linha da outbox.** O worker envia exatamente esse conteúdo, sem consultar `orders` ([09:52] Larissa, Diego; "Beleza, snapshot. Decidido." [09:52] Bruno).

O payload fica enxuto (sem `items`) e limitado a **64 KB**. Acima disso, o envio é recusado com erro em vez de truncado ([09:23] Sofia, [09:24] Diego, Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Guardar só `order_id` e renderizar no envio** | Um retry tardio enviaria o estado atual do pedido, não o da transição, o que gera eventos incoerentes ("caso esquisito", [09:52] Larissa). |
| **Incluir os `items` do pedido no payload** | Infla o payload sem necessidade. O cliente pode buscar detalhes em `GET /orders/:id` ([09:43] Diego, [09:44] Bruno). |
| **Truncar payloads acima do limite** | Um payload desse tamanho indica que algo está errado. Recusar é mais seguro ([09:23] Sofia). |

## Consequências

**Positivas**
- Cada evento é **fiel ao momento da transição**, mesmo em retries de horas depois.
- Retry e replay são **determinísticos**: o mesmo corpo com o mesmo `event_id` (necessário para o [ADR-005](ADR-005-at-least-once-com-x-event-id.md)).
- O worker não precisa ler `orders`, o que reduz acoplamento e queries.
- Permite apagar o pedido (`OrderService.delete`, permitido em `PENDING`/`CANCELLED`) sem quebrar eventos pendentes, desde que a outbox não tenha FK com cascade para `orders`.

**Negativas / trade-offs**
- **Mais armazenamento** por linha, porque o JSON é duplicado em cada evento e em cada endpoint.
- Mudanças no formato do payload não afetam eventos já enfileirados. Isso é bom para consistência, mas exige cuidado se o formato evoluir.
- A renderização acontece dentro da transação de status, que já é pesada ([09:04] Bruno). O custo é pequeno porque o payload não inclui `items`.
