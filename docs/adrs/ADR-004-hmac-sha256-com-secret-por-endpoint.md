# ADR-004 — Autenticação de envio por HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Data:** reunião técnica de quinta-feira
- **Decisores:** Sofia (Segurança), com Bruno e Diego
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [FDD — Contratos públicos](../FDD.md#6-contratos-públicos)

## Status

Aceito.

## Contexto

Os webhooks levam dados de pedidos para um endpoint fora da nossa infraestrutura. O cliente precisa conseguir verificar que (a) a requisição veio de nós e (b) o payload não foi adulterado no caminho ([09:19] Sofia). O fluxo é só outbound: os clientes recebem, não enviam ([09:02] Marcos, [09:03] Sofia).

Diego lembrou que já houve cliente que vazou secret em log da própria aplicação ([09:22]).

## Decisão

1. **Assinatura HMAC-SHA256 do corpo da requisição**, enviada no header `X-Signature` ([09:20] Sofia).
2. **Uma secret única por endpoint de webhook**, nunca uma secret global da plataforma ([09:21] Sofia). Ela fica na tabela de configuração junto de `url`, `customer_id` e estado ativo ([09:21] Bruno, Sofia).
3. **A secret é gerada pelo servidor e devolvida na criação** do webhook ([09:31] Marcos).
4. **Rotação via API:** o cliente pede uma nova secret, e a anterior **continua válida em paralelo por 24 horas**. Depois disso, deixa de valer ([09:21] Sofia; decisão em [09:22]).
5. **URL obrigatoriamente `https`.** Cadastro com `http` é recusado por validação no schema Zod ([09:23] Sofia). Sofia observou que isso "nem é decisão arquitetural", mas registramos aqui porque é parte do modelo de confiança.
6. O código de HMAC e de geração de secret passa por **revisão de segurança da Sofia (≥ 2 dias úteis) antes do deploy** ([09:46] Sofia).
7. **Secret cifrada em repouso** com AES-256-GCM (`node:crypto`, sem dependência nova), usando a chave `WEBHOOK_SECRET_ENCRYPTION_KEY` validada em `src/config/env.ts`. A secret anterior do grace period recebe o mesmo tratamento. Isso decorre do raciocínio da própria reunião: o vazamento de uma secret é o risco central ([09:21] Sofia: "se vaza uma, vaza tudo"; [09:22] Diego), e um dump ou backup do banco não pode expor as secrets de todos os clientes.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | "Se vaza uma, vaza tudo" ([09:21] Sofia). |
| **Rotação com corte imediato da secret antiga** | Não dá tempo ao cliente de migrar os sistemas dele. O grace period de 24h foi a resposta ([09:21] Sofia). |
| **Outro algoritmo de assinatura** | Não chegou a ser discutido. HMAC-SHA256 foi escolhido por ser "padrão de mercado, todo cliente sério tem biblioteca pra isso" ([09:20] Sofia). |
| **Secret em texto puro no banco** (não discutida na reunião; plausível) | Mais simples, mas um acesso indevido ao banco ou a um backup expõe as secrets de todos os endpoints de uma vez, exatamente o "vaza tudo" que a secret por endpoint quer evitar ([09:21] Sofia). |

## Consequências

**Positivas**
- O cliente valida autenticidade e integridade com bibliotecas padrão.
- O vazamento de uma secret afeta só um endpoint.
- A rotação não exige janela de manutenção coordenada.

**Negativas / trade-offs**
- **A secret precisa ser recuperável pelo worker** para assinar. Por isso não pode ser guardada como hash (diferente de `passwordHash` em `prisma/schema.prisma`) e é guardada **cifrada** (item 7). Isso exige uma chave nova no `.env` e cuidado operacional com ela. Perder a chave invalida todas as secrets, e os clientes teriam de rotacionar.
- **Durante as 24h de grace period existem duas secrets válidas.** O FDD define como as duas assinaturas são enviadas. Isso aumenta um pouco a complexidade do lado do cliente.
- A responsabilidade de validar a assinatura, e de usar `X-Timestamp` contra replay attack, é do cliente ([09:44] Diego: "se quiser").
