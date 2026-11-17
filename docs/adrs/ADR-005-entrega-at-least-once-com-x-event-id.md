# ADR-005 — Entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

| Campo | Valor |
| --- | --- |
| Identificador | ADR-005 |
| Status | **Accepted** |
| Data da decisão | Reunião técnica de "quinta-feira, 09:00" (`TRANSCRICAO.md`, cabeçalho). A data de calendário não foi registrada. |
| Data de registro | 2026-11-16 |
| Decisores | Diego (Eng. Sênior, Plataforma, proponente), Larissa (Tech Lead, fechamento), Sofia (Eng. de Segurança), Bruno (Eng. Pleno), Marcos (PM, comunicação aos clientes) |
| Base de código | commit `e7f6311` |
| Relacionados | [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-003](ADR-003-retry-backoff-e-dlq.md), [ADR-004](ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md) |

**Legenda:** `DECISÃO_CONFIRMADA`, `DETALHE_ACORDADO`, `PROPOSTA_TÉCNICA`, `QUESTÃO_ABERTA`, conforme definidos no [ADR-001](ADR-001-outbox-transacional-no-mysql.md).

---

## Status

**Accepted.** Fechado em [09:26] Larissa ("At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão.") e confirmado no resumo de [09:48] Larissa.

## Contexto

- A entrega envolve dois sistemas sem transação comum: a plataforma, com a outbox no MySQL ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)), e o endpoint HTTP do cliente.
- Há retries ([ADR-003](ADR-003-retry-backoff-e-dlq.md)) e replay manual da DLQ.
- A reunião precisava definir a garantia de entrega e como o cliente identifica repetições ([09:24] Diego, [09:25] Bruno).

## Decisão

1. **[DECISÃO_CONFIRMADA]** A garantia de entrega é **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado.
   - Fontes: [09:24] Diego, [09:26] Larissa ("At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."), [09:48] Larissa ("Idempotência por X-Event-Id, garantia at-least-once").
2. **[DECISÃO_CONFIRMADA]** Cada evento carrega um **identificador único**, enviado no header **`X-Event-Id`**: um **UUID gerado quando o evento entra na outbox**. O cliente deduplica por esse identificador.
   - Fontes: [09:25] Diego, [09:26] Larissa.
3. **[DETALHE_ACORDADO]** O mesmo identificador aparece no corpo como `event_id`.
   - Fonte: [09:43] Diego.
   - O formato completo do payload e o conjunto de headers, incluindo `X-Timestamp` e `X-Webhook-Id`, pertencem ao FDD ([09:44] Diego, [09:44] Sofia, [09:45] Diego).
4. **[DETALHE_ACORDADO]** A semântica at-least-once será **documentada em destaque no portal do desenvolvedor**.
   - Fonte: [09:26] Marcos.
5. **[DECISÃO_CONFIRMADA]** **Exactly-once não é garantido nem prometido.** É consequência direta da decisão do item 1.
   - Fontes: [09:25] Diego ("Garantir exactly-once exigiria coordenação dos dois lados"), [09:26] Larissa ("… Decisão.").
   - Ver Alternativas.
6. **[PROPOSTA_TÉCNICA]** Por que duplicatas podem ocorrer. Inferência da análise, não detalhada na reunião:
   - o cliente processa a requisição, mas a resposta não chega antes do timeout de 10 s ([09:42] Diego), e o worker agenda um novo envio;
   - o worker cai depois de o cliente receber, mas antes de marcar o evento como entregue;
   - um evento é reprocessado pelo replay manual da DLQ ([ADR-003](ADR-003-retry-backoff-e-dlq.md)).

### Idempotência: o que este ADR garante e o que não garante

- **Do lado do cliente:** a deduplicação por `X-Event-Id` é **responsabilidade do cliente** ([09:25] Diego; ressalva de [09:25] Sofia: "Isso joga responsabilidade pro cliente").
- **Do lado da plataforma:** a reunião **não** definiu deduplicação de envio. A plataforma pode reenviar um evento já recebido.
- **Ordem:** at-least-once não implica ordem global. A garantia de ordem é a de [ADR-002](ADR-002-worker-separado-com-polling.md): por `order_id`, com um único worker. Ela está sujeita ao risco de reordenação por retry descrito no [ADR-003](ADR-003-retry-backoff-e-dlq.md).

## Alternativas Consideradas

| Alternativa | Quem levantou | Motivo do descarte |
| --- | --- | --- |
| **Exactly-once** | [09:25] Diego, como contraponto | "Exigiria coordenação dos dois lados e fica muito mais complexo" ([09:25] Diego). |

Diego justificou a escolha como padrão de mercado, citando Stripe e GitHub ([09:25] Diego). Isso é registrado como argumento do participante, não como fato verificado neste ADR.

## Consequências

**Positivas**
- Funciona com os mecanismos já decididos (outbox, retry e replay) sem coordenação distribuída.
- A semântica é simples de explicar e de implementar.
- O identificador nasce na inserção da outbox, dentro da transação ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)), e é estável para aquele evento.

**Negativas**
- Clientes que não deduplicam podem processar um evento duas vezes.
- O ônus de deduplicação recai sobre o cliente ([09:25] Sofia) e exige comunicação clara ([09:26] Marcos).
- "Resolve 99% dos casos" ([09:25] Diego) é uma afirmação retórica do participante. **Não é uma métrica medida**, e este ADR não a adota como meta.

## Trade-offs explícitos

- **Simplicidade e robustez da entrega em troca de duplicatas possíveis e trabalho de deduplicação no cliente.** Aceito por ser o padrão citado e porque exactly-once exigiria coordenação bilateral ([09:25] Diego).

## Pendências e riscos

| Tipo | Item | Fonte |
| --- | --- | --- |
| QUESTÃO_ABERTA (lacuna) | Se o replay da DLQ, que "recoloca na outbox como pendente", **preserva o `event_id` original**. Se gerar um novo identificador, o cliente não conseguirá deduplicar o reenvio. **PROPOSTA_TÉCNICA:** preservar o identificador original; decisão a confirmar. | [09:18] Diego, [09:25] Diego |
| QUESTÃO_ABERTA (lacuna) | Quando um customer tem vários webhooks interessados, se cada endpoint recebe o mesmo `X-Event-Id` (evento único com várias entregas) ou se há um evento por webhook. O `X-Webhook-Id` ([09:44] Sofia) distingue o endpoint, mas o modelo de fan-out não foi definido. | [09:25] Diego, [09:44] Sofia |
| QUESTÃO_ABERTA (lacuna) | Se o `X-Timestamp` integra o conteúdo assinado. Ver [ADR-004](ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md). | [09:44] Diego |
| Risco | Clientes que não implementem deduplicação. Mitigação já acordada: documentação no portal ([09:26] Marcos). | [09:26] Marcos |

## Referências às fontes

- `TRANSCRICAO.md`: [09:18] Diego, [09:24] Diego, [09:25] Bruno, [09:25] Diego, [09:25] Sofia, [09:26] Marcos, [09:26] Larissa, [09:42] Diego, [09:43] Diego, [09:44] Diego, [09:44] Sofia, [09:45] Diego, [09:48] Larissa.
- Código: nenhum componente atual é afetado diretamente; a geração de UUID segue a convenção de `prisma/schema.prisma` (`@default(uuid())`).
