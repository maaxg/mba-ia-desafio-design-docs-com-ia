# Da Reunião ao Documento: Webhooks de Notificação de Pedidos

> Enunciado original do desafio: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)

## Sobre o desafio

O ponto de partida é um Order Management System (Node.js, TypeScript, Express, Prisma/MySQL) em produção e a transcrição literal (`TRANSCRICAO.md`) de uma reunião de ~55 minutos. Nela, tech lead, PM, dois engenheiros e a engenheira de segurança decidem como construir um **sistema de webhooks que avisa clientes B2B quando o status de um pedido muda**. Nada foi registrado além da transcrição.

A tarefa foi transformar essa conversa, cruzada com o código real, num pacote de design docs acionável: PRD, RFC, FDD, 8 ADRs e um tracker de rastreabilidade. A regra que guiou o trabalho: **nada pode entrar sem origem identificável**. Cada requisito, decisão ou restrição aponta para um timestamp da reunião ou para um arquivo do repositório. O que foi descartado ou adiado na reunião aparece como fora de escopo ou questão em aberto, nunca como requisito.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code (Claude Opus 5.5)** | Agente principal. Leu o repositório (transcrição, `order.service.ts`, erros, middlewares, schema Prisma, testes), extraiu decisões e redigiu todos os documentos. |
| **Subagente revisor do Claude Code** | Revisão adversarial independente. Recebeu só os caminhos dos arquivos, sem a minha interpretação, com a missão de achar citações erradas, invenções e contradições entre documentos e com o código. |
| **Scripts Python gerados pela IA** | Validação mecânica: conferir que cada `[hh:mm] Falante` do tracker existe na transcrição, que cada caminho de código citado existe e calcular a cobertura. Também aplicaram as correções em lote. |

## Workflow adotado

1. **Contextualização.** Clone do fork e leitura integral da transcrição e dos arquivos que a feature toca: `src/modules/orders/*`, `src/shared/errors/*`, `src/middlewares/*`, `src/config/*`, `src/app.ts`, `src/routes/index.ts`, `prisma/schema.prisma`, `tests/*`, `package.json`.
2. **Triagem da transcrição em cinco baldes:** decisão fechada, requisito funcional, restrição/NFR, descartado/adiado e detalhe secundário. Também mapeei os **ganchos com o código**: `changeStatus` e seu `$transaction`, `AppError`, `requireRole` e `validate`.
3. **ADRs primeiro.** As 6 decisões principais viraram ADRs, já com as alternativas que a reunião descartou. Duas decisões secundárias também ganharam ADR: o payload snapshot, decidido nos últimos minutos da call (ADR-007), e a ordem por pedido com isolamento por endpoint (ADR-008, criado na iteração 4).
4. **RFC** como consolidação em nível de arquitetura, com links para os ADRs, alternativas descartadas e questões em aberto. A regra foi não descer a contrato nem a schema.
5. **FDD** como documento de implementação: modelo Prisma, fluxos em pseudocódigo, 7 endpoints com exemplos, matriz `WEBHOOK_*`, observabilidade e a seção obrigatória de integração cobrindo mais de 15 arquivos reais.
6. **PRD** por último entre os grandes documentos, já que é uma consolidação em linguagem de produto.
7. **Revisão adversarial** por subagente e aplicação das correções.
8. **Tracker** montado varrendo os documentos finais e validado por script.
9. **README** (este arquivo).
10. **Revisão final contra a checklist** de critérios de aceite, item por item, com um script para a parte mensurável (contagens, caminhos, timestamps).

## Prompts customizados

Os prompts abaixo estão **exatamente como foram enviados**, com erros de digitação e tudo. A única edição é no prompt 1: o enunciado colado foi substituído por um marcador, para não repetir aqui o texto inteiro do desafio.

**1. Prompt inicial (meu, para o Claude Code).** Definiu o ponto de partida: clonar o fork antes de qualquer coisa e trabalhar a partir do enunciado completo, inclusive a checklist de critérios de aceite.

```text
https://github.com/maaxg/mba-ia-desafio-design-docs-com-ia -> Clone isso antes de tudo. Descricao da sua tarefa: 

<enunciado completo do desafio colado aqui, incluindo requisitos 1–6 e os critérios de aceite>
```

**2. Revisão adversarial (do agente principal para um subagente revisor).** Pedido de uma leitura que não estivesse ancorada na interpretação de quem escreveu os documentos. O revisor recebeu só os caminhos e as fontes de verdade:

```text
Você é revisor crítico de um pacote de design docs (em PT-BR) em /Users/maxsilva/Code/mba/mba-ia-desafio-design-docs-com-ia. Fontes de verdade: TRANSCRICAO.md (reunião) e o código em src/, prisma/, tests/, package.json. Documentos sob revisão: docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/ADR-*.md. NÃO edite nenhum arquivo; só leia e reporte.

Procure, com evidência (citação do doc + citação da transcrição/código):
1. Afirmações atribuídas a um timestamp/falante que NÃO batem com a transcrição (timestamp errado, falante errado, ou a pessoa não disse aquilo).
2. Requisitos/decisões/restrições sem origem na transcrição ou código (invenções), exceto os marcados com 🔧 no FDD (são decisões de implementação declaradas — avalie só se contradizem algo).
3. Contradições com a transcrição (ex.: algo descartado/adiado aparecendo como requisito) ou com o código (caminhos de arquivo inexistentes, nomes de função/classe errados, comportamento do código descrito errado).
4. Contradições entre os documentos (ex.: valores diferentes de backoff, nomes de endpoint, códigos de erro, nomes de tabela).
5. Duplicação indevida: RFC descendo a nível de implementação do FDD.
6. Requisitos do enunciado: PRD ≥8 FRs, objetivo com meta quantitativa, fora de escopo ≥2 itens, ≥2 riscos com prob/impacto/mitigação; RFC com metadados, TL;DR, ≥2 alternativas com trade-off, ≥2 questões em aberto, links ≥2 ADRs; FDD com ≥4 endpoints com request/response/status, matriz WEBHOOK_*, seção "Integração com o sistema existente" com ≥4 caminhos reais, observabilidade com métricas/logs/tracing; ADRs com Status/Contexto/Decisão/Alternativas Consideradas/Consequências.

Verifique caminhos de arquivo citados com ls/grep. Reporte uma lista priorizada (mais grave primeiro), cada item com arquivo, trecho, problema, e correção sugerida. Seja conciso; no máximo ~25 itens; ignore estilo.
```

**3. Correção de rumo (meu, depois de ler a primeira versão completa).** A IA entregou o pacote com três pontos "para eu decidir": a ambiguidade de "5 tentativas", as decisões que a reunião não detalhou e o fato de os prompts do README terem sido reconstruídos depois. Os números se referem a esses pontos. A resposta devolveu a decisão para a IA e exigiu que tudo fosse conferido contra os critérios de aceite:

```text
1 -> O que voce acha que deve fazer? Sempre siga as instrucoes do prompt. 2 -> Tome a decisao e adicione a doc necessaria que faz mais sentido. 4-> ele espera os prompts exatmente como voce os digitou, entao corrija para ser adequeado aos criterios de aceite. Tudo o que voce fez deve ser passado pelos criterios de aceite.
```

## Iterações e ajustes

Foram **4 iterações principais**: redação inicial com autocrítica, revisão adversarial, validação mecânica do tracker e revisão item a item contra a checklist do enunciado. Os ajustes mais relevantes estão abaixo.

**Iteração 1: redação inicial (autocrítica contra o código)**
- **customer_id do JWT.** Em [09:31] o PM disse "customer_id implícito do JWT". Um minuto depois ([09:32]), Bruno e Larissa corrigiram: o JWT é do operador, e o customer vem no body ou no path. Uma leitura linear teria registrado a versão errada.
- **`WEBHOOK_INVALID_URL` × código real.** A reunião queria a validação de https "no schema Zod" e o código `WEBHOOK_INVALID_URL`. Lendo `validate.middleware.ts`, vi que todo `ZodError` vira `VALIDATION_ERROR`, então as duas coisas não convivem sem alterar código compartilhado. O FDD documenta a solução (código `WEBHOOK_*` em `details[].message`), e o ADR-006 registra o trade-off.
- **Limite de 64 KB.** Verificar o tamanho na inserção da outbox faria a mudança de status sofrer rollback por causa de um problema de notificação. A verificação foi movida para o worker.

**Iteração 2: revisão adversarial** (prompt 2; 22 apontamentos, nenhum grave)
- **Envio sequencial do lote.** Um cliente lento (10s × 50 eventos) atrasaria todos os outros, contradizendo a meta de < 10s e o isolamento entre clientes. O worker passou a enviar em paralelo entre endpoints e em sequência dentro de cada um.
- **Nome da tabela de DLQ** divergia entre o RFC (`webhook_dead_letter`, como dito na reunião) e o FDD (no plural). Unificado.
- **Critério de aceite inventado no PRD** (homologação com os 3 clientes, sem origem). Foi substituído por um critério rastreável: documentação publicada no portal ([09:26], [09:40]).
- **Citações com falante trocado** (ADR-006) e **alternativas não discutidas** apresentadas como se tivessem sido. Foram corrigidas e marcadas como "não discutida na reunião; registrada para completude".
- **`createPrismaClient()` no worker** criaria um segundo client, porque `database.ts` já instancia um singleton no import. Corrigido no ADR-002 e no FDD.
- **Ordem das sprints:** o worker da sprint 2 precisa assinar, mas o HMAC estava na sprint 3. A ordem foi refeita.

**Iteração 3: validação mecânica**
- Um script conferiu o tracker: todo `[hh:mm] Falante` existe na transcrição, e todo caminho `CODIGO` existe no repositório. Também confirmou que nenhum timestamp citado em qualquer documento é inexistente.

**Iteração 4: checklist de critérios de aceite** (prompt 3)
- **Questões em aberto do RFC que não vieram da reunião.** O enunciado pede pontos "levantados na reunião e não decididos ou adiados". Três questões eram dúvidas da própria IA (ordem durante o retry, contagem de "5 tentativas", proteção da secret em repouso). Cada uma foi **decidida e documentada**, e as cinco que restaram no RFC são todas falas da reunião:
  - **Ordem × retry** virou o **ADR-008**: a ordem por pedido vale também durante o retry, e cada endpoint é isolado para que um cliente lento não trave os outros.
  - **"5 tentativas"** foi resolvida no **ADR-003** como 1 envio + 5 retentativas. É a única leitura que acomoda os 5 intervalos (1m/5m/30m/2h/12h, somando 14h36m) e os "quase 15 horas" do Diego em [09:17].
  - **Secret em repouso** ficou no **ADR-004**: cifrada com AES-256-GCM, a partir do raciocínio de [09:21] ("se vaza uma, vaza tudo").
- **Formato literal das seções.** Os ADRs tinham Status como bullet, e não como seção, e o título era "Alternativas consideradas" em vez de "Alternativas Consideradas". No FDD, a seção passou a se chamar "Matriz de erros previstos", como no enunciado.
- **Pasta de ADRs com 5 a 8 arquivos.** Foi removido um `docs/adrs/README.md` (índice) para que a pasta contenha apenas os 8 ADRs.
- **Cobertura do tracker.** Cenários, critérios de aceitação, riscos, dependências e componentes estavam sem linha. O tracker foi de 183 para 306 linhas, com 86% `TRANSCRICAO` e 43 linhas `CODIGO`.
- **Prompts deste README.** A primeira versão mostrava prompts reescritos depois, como resumo das instruções. Foram trocados pelos prompts literais acima.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md): por que e o quê (problema, métricas, escopo, requisitos).
2. [`docs/RFC.md`](docs/RFC.md): a proposta técnica em poucas páginas, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/): uma decisão por arquivo.
   - [ADR-001 Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 Worker separado em polling](docs/adrs/ADR-002-worker-separado-em-polling.md)
   - [ADR-003 Retry com backoff e DLQ](docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
   - [ADR-004 HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
   - [ADR-005 At-least-once com X-Event-Id](docs/adrs/ADR-005-at-least-once-com-x-event-id.md)
   - [ADR-006 Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
   - [ADR-007 Payload snapshot na inserção](docs/adrs/ADR-007-payload-snapshot-na-insercao.md)
   - [ADR-008 Ordem por pedido e isolamento por endpoint](docs/adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md)
4. [`docs/FDD.md`](docs/FDD.md): como construir (modelo de dados, fluxos, contratos, erros, observabilidade, integração com o código). A §14 lista as decisões de implementação que a reunião não cobriu.
5. [`docs/TRACKER.md`](docs/TRACKER.md): de onde veio cada item.

Fonte primária: [`TRANSCRICAO.md`](TRANSCRICAO.md). O código da aplicação (`src/`, `prisma/`, `tests/`) não foi alterado.
