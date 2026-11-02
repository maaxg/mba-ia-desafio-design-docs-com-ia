# ADR-003 — Retry com backoff exponencial (1m/5m/30m/2h/12h) e Dead Letter em tabela separada

- **Data:** reunião técnica de quinta-feira
- **Decisores:** Larissa, Diego, Bruno; Marcos validou a janela; Sofia definiu a regra de acesso ao replay
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Status

Aceito.

## Contexto

O endpoint do cliente pode ficar fora do ar. Diego lembrou que já houve cliente com **2 horas** de indisponibilidade em manutenção planejada ([09:16]). A política precisa:

- cobrir janelas longas de indisponibilidade;
- não deixar eventos "pendurados para sempre" ([09:15] Diego);
- dar ao time um jeito de investigar e reprocessar o que falhou de vez.

## Decisão

1. **Backoff exponencial com 5 retentativas** nos intervalos **1 min → 5 min → 30 min → 2 h → 12 h**. São quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego; decisão em [09:17] Larissa; resumo em [09:48]).
2. Um timeout de 10s sem resposta conta como falha e agenda retry ([09:42] Diego).
3. Esgotadas as tentativas, o evento é **falha permanente** e vai para a **tabela separada `webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego).
4. **Replay manual** por `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego). O endpoint exige role **ADMIN** e registra quem fez o replay, para auditoria ([09:36] Sofia, Larissa).

> **Contagem de tentativas.** A decisão é **1 envio inicial + 5 retentativas** (6 envios no total). É a única leitura que bate com as duas falas: os **5 intervalos** listados por Diego (1m/5m/30m/2h/12h) e o total de "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). Como 1m + 5m + 30m + 2h + 12h = 14h36m, os 5 intervalos só cabem depois do envio inicial. Com 5 envios no total, só caberiam 4 intervalos, ou seja, 2h36m de janela, o que contradiria os ~15h aceitos por Marcos ([09:17]).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **3 tentativas (mais agressivo)** | Cobre só ~30 minutos. Uma indisponibilidade de manhã mataria o evento, e já houve cliente fora por 2h ([09:16] Bruno propôs, Diego refutou). |
| **DLQ como status `failed` na própria outbox** | Polui a leitura da outbox principal. Uma tabela separada deixa a outbox mais limpa e serve de evidência para debug e reprocessamento ([09:17] Larissa perguntou, [09:18] Diego). |

## Consequências

**Positivas**
- Cobre indisponibilidades de até ~15h sem intervenção humana. Marcos considerou aceitável: "se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17]).
- A DLQ isolada dá um ponto único para observar falhas permanentes.
- O replay é controlado e auditável.

**Negativas / trade-offs**
- **Atraso de até ~15h** antes de um evento ser considerado perdido.
- **Reprocessamento manual:** não há replay automático nem aviso ao cliente (o e-mail ficou para fase futura, [09:37] Larissa).
- **Interação com a ordem:** enquanto um evento de um pedido está em retry, os eventos seguintes do mesmo pedido esperam, para preservar a ordem por `order_id` ([09:13] Larissa). A regra está detalhada no [ADR-008](ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md).
