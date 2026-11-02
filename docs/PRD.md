# PRD — Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Produto** | Order Management System (OMS) |
| **Owner de produto** | Marcos (PM) |
| **Tech Lead** | Larissa |
| **Status** | Aprovado na reunião técnica; em detalhamento ([RFC](RFC.md), [FDD](FDD.md)) |
| **Prazo-alvo** | Fim de novembro (pedido da Atlas Comercial), estimado em 3 sprints |

---

## 1. Resumo e contexto

O OMS vai **avisar os sistemas dos clientes B2B, em menos de 10 segundos, sempre que um pedido deles mudar de status**. O aviso é uma chamada HTTP (webhook) para uma URL que o próprio cliente cadastra. Hoje esse aviso não existe: o OMS não emite nenhuma notificação externa, e os clientes descobrem mudanças consultando a API periodicamente.

## 2. Problema e motivação

- **Pedido formal de três clientes B2B:** Atlas Comercial, MaxDistribuição e Nova Cargo querem ser notificados em tempo real quando o status dos seus pedidos muda ([09:00] Marcos).
- **A integração atual é lenta e cara para eles:** os clientes fazem polling em `GET /orders` "de tempos em tempos" ([09:00] Marcos).
- **Risco comercial:** a Atlas sinalizou que pode migrar para um concorrente se a funcionalidade não sair até o fim do trimestre ([09:00] Marcos). O prazo combinado depois foi o fim de novembro ([09:45] Marcos).

## 3. Público-alvo e cenários de uso

| Persona | Quem é | O que precisa |
|---|---|---|
| **Integrador do cliente B2B** | Time técnico da Atlas, MaxDistribuição, Nova Cargo | Receber eventos de status com confiança (autenticidade, sem perda) e sem polling |
| **Usuário autenticado do OMS** (qualquer role) | Usuários do sistema que representam o cliente na API ([09:32] Marcos) | Cadastrar e manter os webhooks dos clientes via API |
| **Administrador do OMS** (role `ADMIN`) | Time interno de suporte/plataforma | Reprocessar entregas que falharam de vez, com rastro de auditoria |

**Cenários**

1. **Assinar só o que interessa** (exemplo ilustrativo). Um cliente só quer saber quando o pedido vira `SHIPPED` ou `DELIVERED`. Um operador cadastra o webhook com esse filtro, e o cliente não recebe os outros status ([09:33] Marcos).
2. **Atualização quase imediata.** O pedido `ORD-000123` passa de `PROCESSING` para `SHIPPED`. Em poucos segundos o sistema da Nova Cargo recebe o evento e atualiza o rastreio, sem consultar `GET /orders`.
3. **Cliente em manutenção.** O endpoint da MaxDistribuição fica 2h fora do ar em manutenção planejada. Os eventos são reenviados automaticamente quando ele volta ([09:16] Diego).
4. **Secret vazada.** O integrador percebe que a secret foi parar num log. Ele pede uma nova, e a antiga continua aceita por 24h enquanto ele migra ([09:21] Sofia, [09:22] Diego).
5. **Diagnóstico.** O cliente reclama que "não recebeu". O operador consulta o histórico de entregas do webhook e vê as tentativas, o status HTTP e o tempo de resposta ([09:34] Marcos).
6. **Reprocessamento.** Depois de ~15h de falhas, o evento vai para a fila de falhas permanentes. Um ADMIN dispara o reenvio quando o cliente confirma que corrigiu o endpoint ([09:18] Diego, [09:36] Sofia).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| O1 | Notificação percebida como "tempo real" | Tempo entre a mudança de status e o recebimento pelo cliente (p95, cliente saudável) | **< 10 segundos** | [09:02] Marcos |
| O2 | Eliminar o polling dos clientes solicitantes | Clientes solicitantes com webhook ativo e integrado | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até o **fim de novembro** | [09:00], [09:45] Marcos |
| O3 | Nenhum evento perdido | Mudanças de status commitadas sem evento registrado (para endpoints inscritos) | **0** | [09:40] Bruno |
| O4 | Tolerar indisponibilidade do cliente | Janela de retentativa automática antes de declarar falha | **≈ 15 horas** | [09:17] Diego, Marcos |
| O5 | Entregar no prazo com segurança | Revisão de segurança concluída antes do deploy | **≥ 2 dias úteis** reservados; revisão aprovada | [09:46] Sofia |

## 5. Escopo

### 5.1 Incluso

- Webhooks **outbound** (OMS → cliente) para mudanças de status de pedido.
- Cadastro, listagem, edição, remoção e rotação de secret dos webhooks via API.
- Filtro, por webhook, dos status que o cliente quer receber.
- Entrega assinada, com retentativas automáticas e fila de falhas permanentes.
- Histórico de entregas por webhook.
- Reprocessamento manual de falhas permanentes por ADMIN.

### 5.2 Fora de escopo

| Item | Situação | Origem |
|---|---|---|
| **Aviso por e-mail ao cliente** quando o webhook falha repetidamente | **Adiado** para a próxima fase, depois de medir o impacto | [09:37] Larissa |
| **Dashboard/painel visual** para o cliente ver seus webhooks | **Descartado** nesta fase; é projeto separado do time de frontend | [09:40] Larissa |
| **Rate limiting** de envio por cliente | **Adiado**: "observar e decidir depois" | [09:38]–[09:39] Diego, Larissa |
| **Webhooks inbound** (cliente envia para nós) | **Descartado**: só saída | [09:02] Marcos |
| **Garantia de entrega exatamente uma vez** | **Descartado**: o cliente deduplica | [09:25] Diego |
| **Ordem global** dos eventos entre pedidos | **Descartado**: os clientes nunca pediram | [09:14] Marcos |
| **Itens do pedido** no conteúdo da notificação | **Descartado**: o cliente consulta `GET /orders/:id` | [09:43] Diego |
| **Arquivamento** de notificações antigas (~30 dias) | **Fora desta feature** | [09:08] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| FR-01 | O sistema deve **notificar o cliente via HTTP POST** na URL cadastrada sempre que um pedido dele mudar de status. | [09:00] Marcos, [09:40] Bruno |
| FR-02 | Um usuário autenticado deve poder **cadastrar um webhook** informando `customerId`, URL e lista de status de interesse. O `customerId` vem no body, **não** do JWT. | [09:31] Marcos, [09:32] Larissa |
| FR-03 | Na criação, o sistema deve **gerar a secret** do webhook e **devolvê-la na resposta**. | [09:31] Marcos |
| FR-04 | Deve ser possível **listar** os webhooks de um customer (customer identificado no path). | [09:33] Bruno, [09:32] Larissa |
| FR-05 | Deve ser possível **editar** um webhook (URL, status de interesse, ativo/inativo). | [09:33] Bruno, [09:21] Bruno |
| FR-06 | Deve ser possível **remover** um webhook. | [09:33] Bruno |
| FR-07 | Cada webhook deve permitir **escolher quais status** disparam notificação. Status fora da lista não geram notificação para aquele webhook. | [09:33] Marcos, [09:34] Bruno |
| FR-08 | Um customer pode ter **vários webhooks**, e cada notificação deve identificar qual cadastro a originou. | [09:44] Sofia |
| FR-09 | O cliente deve poder **rotacionar a secret** pela API. A secret anterior continua válida por **24 horas** e depois expira. | [09:21] Sofia |
| FR-10 | Deve ser possível consultar o **histórico das últimas 100 entregas** de um webhook, com sucesso/falha, conteúdo enviado, resposta e tempo de resposta. | [09:34] Marcos |
| FR-11 | Entregas que falharem devem ser **reenviadas automaticamente** 5 vezes, com intervalos crescentes de 1 min, 5 min, 30 min, 2 h e 12 h. | [09:17] Diego, Larissa |
| FR-12 | Esgotadas as retentativas, a notificação deve ir para uma **fila de falhas permanentes**, guardando o conteúdo, o motivo e o momento da falha. | [09:18] Diego |
| FR-13 | Um **ADMIN** deve poder **reprocessar** uma falha permanente. O sistema registra quem fez o reprocessamento. | [09:18] Diego, [09:36] Sofia |
| FR-14 | Cada notificação deve conter: identificador único do evento, tipo `order.status_changed`, data/hora, id e número do pedido, status anterior e novo, customer e valores do pedido. **Sem itens.** | [09:43] Diego |
| FR-15 | O conteúdo da notificação deve refletir o pedido **no momento da mudança de status**, mesmo se reenviado horas depois. | [09:52] Larissa |
| FR-16 | Cada notificação deve permitir ao cliente **verificar a autenticidade** (assinatura) e **detectar duplicatas** (identificador do evento). | [09:20] Sofia, [09:25] Diego |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| NFR-01 | **Latência:** notificação entregue em menos de 10s (verificação de novos eventos a cada 2s). | [09:02] Marcos, [09:09] Diego |
| NFR-02 | **Consistência:** se o status mudou, a notificação foi registrada; se a mudança falhou, nenhuma notificação existe. | [09:06] Diego, [09:40] Bruno |
| NFR-03 | **Isolamento:** a lentidão ou indisponibilidade de um cliente não pode afetar a mudança de status de nenhum pedido. | [09:04] Bruno |
| NFR-04 | **Entrega pelo menos uma vez:** o cliente pode receber a mesma notificação mais de uma vez e deve deduplicar pelo identificador do evento. | [09:24]–[09:26] Diego, Larissa |
| NFR-05 | **Ordem:** garantida apenas por pedido (inclusive quando há reenvio), enquanto houver um único processo de envio. Não há ordem global. | [09:12]–[09:13] Diego, Larissa |
| NFR-06 | **Segurança:** assinatura HMAC-SHA256 com secret exclusiva por webhook, guardada cifrada ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)). | [09:20]–[09:22] Sofia |
| NFR-07 | **Segurança:** só URLs `https`; cadastro com `http` é recusado com erro de validação. | [09:23] Sofia |
| NFR-08 | **Limite de tamanho:** notificações acima de 64 KB não são enviadas (erro, sem truncar). | [09:23] Sofia, [09:24] Diego, Larissa |
| NFR-09 | **Timeout:** o cliente tem 10s para responder; acima disso, conta como falha e entra em retentativa. | [09:42] Diego |
| NFR-10 | **Autorização:** a configuração de webhooks aceita qualquer role autenticada; o reprocessamento exige ADMIN. | [09:36]–[09:37] Sofia |
| NFR-11 | **Operação:** o envio roda num processo separado da API, para que reinícios da API não interrompam as notificações. | [09:11] Diego |
| NFR-12 | **Manutenibilidade:** nenhuma infraestrutura nova; seguir os padrões de código já existentes. | [09:07] Diego, [09:30] Larissa |

## 8. Decisões e trade-offs principais

| Decisão | Ganho | Custo aceito | ADR |
|---|---|---|---|
| Registrar o evento no próprio banco, junto com a mudança de status (outbox) | Nenhum evento perdido, sem infra nova | Tabela cresce; latência limitada pelo polling | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Processo de envio separado, verificando a cada 2s | Simples; atende os 10s | Até 2s de atraso; um único processo | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15h e depois fila de falhas | Cobre manutenções reais (2h) sem eventos eternos | Reprocessamento final é manual | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| Assinatura com secret por webhook e rotação de 24h | Vazamento afeta um só webhook | Cliente precisa validar assinatura | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| "Pelo menos uma vez" em vez de "exatamente uma vez" | Muito mais simples; padrão de mercado | Cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reaproveitar os padrões do código | Time pequeno entrega mais rápido | Herda limitações (ex.: formato de erro de validação) | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Conteúdo congelado no momento da mudança | Reenvios coerentes | Mais armazenamento | [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) |
| Ordem por pedido também nos reenvios; cliente lento isolado | Cliente recebe cada pedido em ordem; um cliente não atrasa outro | Próximos eventos de um pedido esperam o reenvio do anterior | [ADR-008](adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md) |

## 9. Dependências

| Dependência | Tipo | Origem |
|---|---|---|
| Revisão de segurança da Sofia (≥ 2 dias úteis) antes do deploy | Pessoa/processo | [09:46] Sofia |
| Documentação no portal do desenvolvedor: integração, validação de assinatura, deduplicação | Produto (Marcos) | [09:26], [09:40] Marcos |
| Confirmação do prazo com a Atlas | Comercial (Marcos) | [09:47] Marcos |
| Clientes com endpoint `https` público e capacidade de validar HMAC-SHA256 e deduplicar | Externa | [09:20], [09:23], [09:24] |
| Banco MySQL e stack atuais (Prisma, Node) | Técnica | [09:11] Diego |
| Usuários internos que representam os clientes na API | Operação | [09:32] Marcos |

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| **Atraso na entrega e perda da Atlas para o concorrente** | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limit fora); 3 sprints com revisão de segurança já planejada; Marcos alinha o prazo com a Atlas ([09:37]–[09:47]). |
| **Cliente processa notificação duplicada** (não deduplica) | Média | Médio | Identificador único em toda notificação; documentação em destaque no portal ([09:26] Marcos). |
| **Vazamento de secret** do lado do cliente | Média | Alto | Secret por webhook; rotação self-service com 24h de convivência ([09:21]–[09:22] Sofia, Diego). |
| **Cliente sobrecarregado** por rajadas de notificações | Baixa | Médio | Monitorar volume por cliente; rate limiting como próxima decisão ([09:38]–[09:39] Diego). |
| **Cliente fora do ar por mais de 15h** perde notificações | Baixa | Médio | Fila de falhas permanentes + reprocessamento por ADMIN ([09:18] Diego). |
| **Mudança de status falha por problema no registro da notificação** | Baixa | Alto | Comportamento intencional (consistência), coberto por testes ([09:40]–[09:41] Bruno, Diego). |

## 11. Critérios de aceitação

1. Um webhook cadastrado para `SHIPPED` recebe uma notificação assinada em **menos de 10s** quando um pedido do customer vai para `SHIPPED`. Mudanças para outros status não geram notificação para ele.
2. A tentativa de cadastrar URL `http://...` é recusada com erro de validação.
3. A secret aparece na criação ([09:31] Marcos) e na rotação, e não volta a aparecer em listagens. Este critério é derivado do risco de vazamento de [09:22] Diego.
4. Depois de rotacionar a secret, as notificações das próximas 24h são verificáveis com a secret antiga **ou** a nova. Passadas as 24h, só com a nova.
5. Com o endpoint do cliente fora do ar, há reenvios nos intervalos 1m/5m/30m/2h/12h. Esgotados, a notificação aparece na fila de falhas.
6. Um OPERATOR **não** consegue reprocessar uma falha (erro de permissão). Um ADMIN consegue, e o registro mostra quem reprocessou.
7. O reenvio de uma mesma notificação mantém o mesmo identificador de evento.
8. O histórico de entregas mostra, por tentativa: resultado, status HTTP, conteúdo enviado, resposta e tempo.
9. Se a mudança de status falha, nenhuma notificação é enviada. Se o registro da notificação falha, a mudança de status não acontece.
10. A documentação de integração (validação de assinatura e deduplicação por `X-Event-Id`) está publicada no portal do desenvolvedor antes do go-live ([09:26], [09:40] Marcos).

## 12. Estratégia de testes e validação

| Nível | O que valida | Como |
|---|---|---|
| Unitário | Cálculo do backoff, assinatura HMAC (com e sem secret anterior), montagem do conteúdo, regra `https` | Vitest |
| Integração (API) | CRUD, rotação, histórico, permissão ADMIN no reprocessamento, erros `WEBHOOK_*` | Vitest + Supertest, padrão de `tests/orders.test.ts` |
| Integração (consistência) | Mudança de status + registro da notificação na mesma transação; rollback nos dois sentidos | Teste sobre `PATCH /orders/:id/status` com falha forçada |
| Worker | Entrega OK, timeout de 10s, erro 5xx, retentativas, fila de falhas, limite de 64 KB | Servidor HTTP local simulando o cliente; relógio controlado |
| Segurança | Revisão de código de HMAC e geração de secret | Revisão da Sofia, ≥ 2 dias úteis antes do deploy ([09:46]) |
| Documentação para o cliente | Instruções de verificação de assinatura e deduplicação | Revisão do conteúdo do portal por Marcos ([09:26], [09:40] Marcos) |

Detalhes técnicos de cada teste: [FDD §11](FDD.md#11-critérios-de-aceite-técnicos).
