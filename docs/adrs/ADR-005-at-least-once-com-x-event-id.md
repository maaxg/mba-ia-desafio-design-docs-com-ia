# ADR-005 — Garantia de entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

- **Data:** reunião técnica de quinta-feira
- **Decisores:** Diego, Larissa; Sofia e Marcos avaliaram o impacto para o cliente
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Status

Aceito.

## Contexto

Com outbox + worker + retry, há janelas em que o mesmo evento pode ser enviado mais de uma vez. Exemplos: o cliente processou mas respondeu depois do timeout de 10s; o worker caiu depois do envio e antes de marcar como entregue; um admin fez replay de um item da DLQ. É preciso declarar a semântica de entrega e dar ao cliente uma forma de lidar com duplicatas.

## Decisão

1. A garantia oficial é **at-least-once**: o cliente **pode receber o mesmo evento mais de uma vez** e tem que estar preparado para isso ([09:24] Diego).
2. Cada evento recebe um **UUID (`event_id`) gerado quando entra na outbox**, único por evento. Ele é enviado no header **`X-Event-Id`** e também no corpo ([09:25] Diego; corpo com `event_id` em [09:43] Diego).
3. O `event_id` **não muda** entre retentativas nem no replay da DLQ. É isso que permite ao cliente deduplicar.
4. A deduplicação é **responsabilidade do cliente**, e Marcos vai documentar isso com destaque no portal do desenvolvedor ([09:26] Marcos).

Decisão registrada em [09:26] Larissa e confirmada no resumo de [09:48].

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exigiria coordenação entre os dois lados e ficaria "muito mais complexo" ([09:25] Diego). |
| **At-most-once (enviar sem retry)** (não discutida na reunião; registrada para completude) | Incompatível com a política de retry do [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md): um cliente fora do ar perderia eventos. |

## Consequências

**Positivas**
- Segue o padrão de mercado (Stripe e GitHub, citados em [09:25] Diego), que os clientes já conhecem.
- Simplifica o worker: na dúvida, reenviar é seguro.
- Torna seguro o replay manual da DLQ.

**Negativas / trade-offs**
- **Transfere responsabilidade para o cliente**, ponto levantado pela Sofia ([09:25]: "Isso joga responsabilidade pro cliente"). A mitigação é a documentação no portal.
- Um cliente que não deduplica pode processar uma transição duas vezes.
