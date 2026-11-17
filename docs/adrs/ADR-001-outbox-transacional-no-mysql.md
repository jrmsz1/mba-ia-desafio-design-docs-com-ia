# ADR-001 — Outbox transacional no MySQL existente

| Campo | Valor |
| --- | --- |
| Identificador | ADR-001 |
| Status | **Accepted** |
| Data da decisão | Reunião técnica de "quinta-feira, 09:00" (`TRANSCRICAO.md`, cabeçalho). A data de calendário não foi registrada na transcrição. |
| Data de registro | 2026-11-16 |
| Decisores | Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos) |
| Base de código | commit `e7f6311` |
| Relacionados | [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-da-arquitetura-e-componentes-existentes.md) |

**Legenda de classificação:** `DECISÃO_CONFIRMADA` = fechamento explícito na reunião ("decidido", "decisão", "anotado") ou presença no resumo de [09:48]. `DETALHE_ACORDADO` = proposto por um participante e aceito sem objeção, sem fechamento formal. `PROPOSTA_TÉCNICA` = sugestão sem acordo explícito ou inferência da análise do código, pendente de revisão. `QUESTÃO_ABERTA` = ponto não decidido.

---

## Status

**Accepted.** A decisão central (itens 1 e 2) foi fechada em [09:08] Larissa e confirmada no resumo de [09:48] Larissa.

## Contexto

- Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram notificação quando o status de seus pedidos muda; hoje fazem polling em `GET /orders` ([09:00] Marcos). Para eles, "abaixo de 10 segundos" é tempo real ([09:02] Marcos).
- O OMS não possui mecanismo de notificação externa: no commit `e7f6311` não há chamadas HTTP de saída, eventos ou filas em `src/` (inspeção por busca no código).
- A mudança de status ocorre em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126–179), dentro de `this.prisma.$transaction(async (tx) => …)`. Nessa transação o código:
  - lê o pedido;
  - valida a transição com `canTransition` (`src/modules/orders/order.status.ts`);
  - debita ou repõe estoque conforme `shouldDebitStock` / `shouldReplenishStock`;
  - atualiza `orders.status`;
  - insere em `order_status_history`.
- O tipo `TxClient = Prisma.TransactionClient` já existe em `order.service.ts` (linha 24).
- Bruno descreveu essa transação: "A transação de mudança de status hoje já é pesada" ([09:04] Bruno). Nota de precisão: no código, o débito de estoque ocorre apenas em `PENDING → PAID` e a reposição em `PAID|PROCESSING → CANCELLED` (`order.status.ts`, linhas 24–37), não em toda transição.
- Problema central: notificar um sistema externo sem acoplar a mudança de status à disponibilidade do cliente e sem perder eventos quando a transação é confirmada.

## Decisão

1. **[DECISÃO_CONFIRMADA]** Registrar os eventos de webhook numa **tabela outbox no MySQL já existente**. Não haverá chamada HTTP síncrona nem infraestrutura nova de mensageria.
   - Fontes: [09:06] Diego, [09:07] Diego, [09:08] Larissa ("Tá decidido então: outbox em MySQL"), [09:48] Larissa.
2. **[DECISÃO_CONFIRMADA]** Inserir na outbox **dentro da mesma transação** de `changeStatus`. Se a inserção falhar, a transação inteira sofre rollback. "Não pode ter caso de status mudar e evento não sair."
   - Fontes: [09:06] Diego, [09:40] Bruno, [09:41] Diego ("Essencial"), [09:48] Larissa ("transação atômica com mudança de status").
3. **[DETALHE_ACORDADO]** Expor a inserção como uma função que recebe o client transacional, `publishWebhookEvent(tx, order, fromStatus, toStatus)`, chamada pelo `order.service`. Não se injeta um repository inteiro.
   - Fontes: [09:41] Bruno, [09:41] Diego.
4. **[DETALHE_ACORDADO]** Filtrar **na inserção**: o evento só entra na outbox se algum webhook do customer escutar aquele status.
   - Fontes: [09:33] Marcos, [09:34] Bruno, [09:34] Diego ("Concordo").
5. **[DECISÃO_CONFIRMADA]** Guardar o payload **renderizado no momento da inserção** (snapshot), e não reconstruí-lo no envio.
   - Fontes: [09:52] Larissa, [09:52] Diego, [09:52] Bruno ("Beleza, snapshot. Decidido.").
6. **[DETALHE_ACORDADO]** Usar **UUID** como identificador das linhas da outbox, seguindo o padrão do projeto ("UUID, segue o padrão do resto do projeto. Tudo é uuid.").
   - Fontes: [09:51] Larissa, em resposta à pergunta de [09:51] Diego; Diego responde "Beleza, só queria confirmar".
   - Não há marca formal de fechamento e a troca é posterior ao resumo de [09:48].
   - Coerente com o código: todos os models usam `@id @default(uuid()) @db.Char(36)` em `prisma/schema.prisma`.

> Os itens 5 e 6 foram tratados **depois do resumo de [09:48]**, só entre Larissa, Diego e Bruno; Marcos e Sofia saíram em [09:50]. O item 5 foi fechado ("Decidido", [09:52] Bruno); o item 6 foi acordado sem fechamento formal.

7. **[PROPOSTA_TÉCNICA]** Diego descreveu a tabela ([09:08] Diego), sem fechamento formal específico:
   - índice em `status` (pendente, processando, falhou, entregue) e em `created_at`;
   - leitura de pendentes em batch pequeno.

   O schema concreto fica para o FDD.

## Alternativas Consideradas

| Alternativa | Quem levantou | Motivo do descarte |
| --- | --- | --- |
| Disparo HTTP **síncrono** dentro de `changeStatus` | [09:03] Larissa (pergunta) | "Qualquer cliente lento vai travar mudança de status pra outros pedidos" ([09:04] Bruno). Se o cliente estiver fora do ar, não há como desfazer a mudança de status ([09:04] Bruno). "Síncrono está fora de questão" ([09:06] Diego). |
| **Redis Streams** ou similar | [09:07] Larissa | Exige subir infraestrutura adicional ([09:07] Larissa). "Subir Redis Cluster pra isso é overengineering" para um time pequeno ([09:07] Diego). |
| Injetar um **repository de webhook** no `OrderService` | [09:41] Bruno | Preterida em favor de uma função que recebe o `tx` ([09:41] Diego: "Não precisa injetar repository inteiro"). |
| Filtrar os eventos **na hora do envio** | [09:34] Diego (pergunta) | Filtrar na inserção economiza linhas ([09:34] Bruno). |
| Guardar só `order_id` e **renderizar no envio** | [09:51] Bruno (pergunta) | O evento deve refletir o estado do momento da mudança ([09:52] Larissa). |

## Consequências

**Positivas**
- Atomicidade entre a mudança de status e o registro do evento: se a transação confirma, o evento existe; se ela é desfeita, o evento some junto ([09:06] Diego).
- Nenhuma infraestrutura nova; reaproveita o MySQL e o Prisma já em uso ([09:07] Diego).
- A indisponibilidade de clientes externos deixa de afetar `changeStatus`.
- O snapshot preserva o estado no instante da transição.
- O filtro na inserção evita linhas sem destinatário.

**Negativas**
- A transação de `changeStatus` fica maior. Inferência da análise: ela passa a consultar os webhooks do customer e a inserir na outbox. No Prisma 5.x a `$transaction` interativa tem tempo-limite padrão; o valor deve ser confirmado na documentação da versão 5.22.0 antes da implementação. A pendência continua registrada no FDD (§6.4 e §13).
- A outbox cresce continuamente. O arquivamento de linhas entregues ("depois de 30 dias ou assim") ficou **fora do escopo** desta feature ([09:08] Diego).
- A entrega passa a ser assíncrona: a latência depende do worker ([ADR-002](ADR-002-worker-separado-com-polling.md)).
- Com o filtro na inserção, um webhook criado ou alterado depois da transição não recebe eventos já filtrados. Inferência, consequência direta do item 4.

## Trade-offs explícitos

- **Consistência forte entre status e evento, pagando com uma transação maior.** Aceito para eliminar o risco de "status mudar e evento não sair" ([09:40] Bruno).
- **Simplicidade operacional (MySQL existente) em vez de vazão e reatividade de um broker dedicado.** Aceito por ser um time pequeno ([09:07] Diego).
- **Snapshot imutável em vez de dados sempre atuais.** O evento reflete o estado do momento da mudança de status ([09:52] Larissa). Para detalhes que não vão no payload, como os itens, o cliente consulta `GET /orders/:id` ([09:43] Diego).

## Pendências e riscos

| Tipo | Item | Fonte |
| --- | --- | --- |
| QUESTÃO_ABERTA (lacuna da análise, não levantada na reunião) | A **criação** do pedido (`OrderService.create`, `order.service.ts`, linhas 50–124) grava status `PENDING` sem passar por `changeStatus`. A reunião tratou apenas da integração em `changeStatus` ([09:40] Bruno). Falta definir se a criação gera evento. | código + [09:40] Bruno |
| QUESTÃO_ABERTA (lacuna da análise) | Falta definir se há uma linha de outbox por evento ou uma por webhook interessado (fan-out). Ver ADR-005. | — |
| QUESTÃO_ABERTA (lacuna da análise) | Falta definir onde é aplicado o limite de 64 KB, registrado como requisito não funcional (`ACORDADO`) em [09:24] Larissa: se na inserção (o que abortaria a mudança de status) ou no envio. | [09:23] Sofia, [09:24] Diego, [09:24] Larissa |
| Risco (hipótese, não reproduzida) | `changeStatus` lê o pedido com `findUnique`, sem bloqueio de linha (`order.service.ts`, linha 132). Duas mudanças concorrentes do mesmo pedido poderiam passar ambas por `canTransition`. O problema é pré-existente; com a outbox, poderia gerar dois eventos. | código |
| Risco | O crescimento da outbox sem arquivamento nesta fase. | [09:08] Diego |

## Referências às fontes

- `TRANSCRICAO.md`: [09:00] Marcos, [09:02] Marcos, [09:03] Larissa, [09:04] Bruno, [09:06] Diego, [09:07] Larissa, [09:07] Diego, [09:08] Diego, [09:08] Larissa, [09:33] Marcos, [09:34] Diego, [09:34] Bruno, [09:40] Bruno, [09:41] Diego, [09:41] Bruno, [09:43] Diego, [09:48] Larissa, [09:51] Larissa, [09:51] Diego, [09:52] Larissa, [09:52] Diego, [09:52] Bruno.
- Código:
  - `src/modules/orders/order.service.ts` (`OrderService.changeStatus`, `OrderService.create`, `type TxClient`)
  - `src/modules/orders/order.status.ts` (`canTransition`, `shouldDebitStock`, `shouldReplenishStock`)
  - `prisma/schema.prisma` (models `Order`, `OrderStatusHistory`; convenção `uuid()`)
