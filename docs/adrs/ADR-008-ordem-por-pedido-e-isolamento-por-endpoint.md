# ADR-008 — Ordem por pedido preservada durante retries e envio isolado por endpoint no worker

- **Data:** consolidado no detalhamento técnico (FDD), a partir da reunião de quinta-feira
- **Decisores:** Larissa (Tech Lead), com base nas falas de Diego, Bruno e Marcos
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [FDD §5.2](../FDD.md#52-processamento-pelo-worker)

## Status

Aceito.

## Contexto

A reunião fechou três pontos que, combinados, se chocam:

1. **Ordem por pedido é a garantia oferecida.** "Por enquanto, single-worker e ordering implícita por order_id" ([09:12] Diego). Larissa registrou: "Não é garantia de ordering global, só por order_id e enquanto for single-worker" ([09:13]). Marcos confirmou que os clientes "só querem saber se cada pedido deles mudou" ([09:14]).
2. **Retry com backoff de até 12h** ([09:17], [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)).
3. **Um cliente lento não pode travar os outros.** Bruno usou esse argumento contra o envio síncrono ([09:04]: "qualquer cliente lento vai travar mudança de status pra outros pedidos"), com timeout de 10s por chamada ([09:42] Diego).

A ordem por `created_at` de [09:12] só vale no caminho feliz. Suponha que `PAID→PROCESSING` falhe e vá para retry em 1 minuto. Se nada for feito, `PROCESSING→SHIPPED` do mesmo pedido, criado segundos depois, é entregue antes. Além disso, com um único worker processando o lote em sequência, um cliente que leva 10s por chamada atrasa todos os demais clientes.

## Decisão

1. **Ordem por (endpoint, pedido) também durante retries.** Um evento só fica elegível para envio se não existir, para o mesmo `webhook_id` e o mesmo `order_id`, um evento mais antigo em `PENDING` ou `PROCESSING`. O próximo evento do pedido só sai quando o anterior termina em `DELIVERED` ou `FAILED` (dead letter). Isso mantém literalmente a garantia de [09:12]–[09:13].
2. **Isolamento por endpoint.** Ainda em **um único processo worker** ([ADR-002](ADR-002-worker-separado-em-polling.md)), os eventos de endpoints diferentes são enviados em paralelo. Dentro de um mesmo endpoint, o envio é sequencial por `created_at`. Endpoints com envio em andamento não entram no lote seguinte.
3. **Falha interrompe o grupo no ciclo.** Quando um envio de um endpoint falha, os eventos seguintes daquele endpoint voltam para `PENDING` sem consumir tentativa e saem no próximo ciclo (2s). Assim, um endpoint fora do ar não consome o ciclo com vários timeouts seguidos.
4. **Pedidos diferentes não se bloqueiam.** O bloqueio do item 1 é por pedido, não pelo endpoint inteiro. Um pedido com evento em retry não segura os eventos de outros pedidos do mesmo cliente.

Implementação: subconsulta `NOT EXISTS` na query do worker e índice `(webhook_id, order_id, created_at)` na outbox ([FDD §4](../FDD.md#4-modelo-de-dados) e [§5.2](../FDD.md#52-processamento-pelo-worker)).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Processar o lote inteiro em sequência**, como na leitura literal de "processa em ordem de created_at" ([09:12]) | Um cliente lento (até 10s por evento) atrasaria todos os outros e estouraria a meta de < 10s ([09:02] Marcos), o mesmo problema que motivou descartar o envio síncrono ([09:04] Bruno). |
| **Não bloquear eventos posteriores durante o retry** (ordem só no caminho feliz) | Quebra a única garantia de ordem prometida na reunião ([09:13] Larissa) justamente quando o cliente está instável, que é o caso em que a ordem mais importa. |
| **Bloquear o endpoint inteiro enquanto houver qualquer retry** | Um único pedido problemático seguraria todos os pedidos do cliente por até ~15h. Os clientes só pediram ordem por pedido, não ordem global ([09:14] Marcos). |
| **Múltiplos workers com particionamento por `order_id`** | Adiado explicitamente: "problema do futuro, não agora" ([09:13] Diego). |

## Consequências

**Positivas**
- A garantia de ordem por pedido vale também com retries, não só no caminho feliz.
- A latência de um cliente não contamina a dos outros. A meta de < 10s depende só da saúde de cada cliente.
- Continua um único processo, sem lock distribuído nem infraestrutura nova.

**Negativas / trade-offs**
- **Retenção por pedido.** Enquanto um evento de um pedido está em retry, os eventos seguintes daquele pedido esperam, por até ~15h, até o anterior ir para a dead letter. Aceitamos o atraso em troca da ordem.
- **Lacuna na sequência.** Quando um evento vai para a dead letter, o seguinte é liberado e o cliente pode receber `PROCESSING→SHIPPED` sem ter recebido `PAID→PROCESSING`. O cliente percebe a lacuna pelo `from_status` do payload ([09:43] Diego). Se um ADMIN fizer o replay depois, o evento antigo chega fora de ordem e o cliente deve usar o `timestamp` do corpo.
- **Query mais cara** (subconsulta `NOT EXISTS`), mitigada pelo índice composto.
- **Estado em memória** (endpoints em envio) no worker. Num crash, ele se perde e os eventos `PROCESSING` voltam para `PENDING` no boot, o que é coerente com o at-least-once ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)).
