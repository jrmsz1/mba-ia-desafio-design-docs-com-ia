# FDD — Webhooks de Notificação de Pedidos

| Metadado | Valor |
|---|---|
| **Status** | **Draft — para revisão técnica** (não autoriza implementação nem deploy) |
| **Data do documento** | 2026-11-16 |
| **Autor** | [@jrmsz1](https://github.com/jrmsz1) |
| **Revisores previstos** | Revisão do design: Larissa (Tech Lead), Bruno (Pedidos) e Diego (Plataforma), conforme [09:50] Larissa. Revisão de segurança do código antes do deploy: Sofia ([09:46] Sofia). Incluir Marcos (PM) é sugestão deste documento. |
| **Base de código** | commit `e7f6311` |
| **Documentos de origem** | [RFC](RFC.md), [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) a [ADR-006](adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md), [`TRANSCRICAO.md`](../TRANSCRICAO.md) |

## 0. Como ler este documento

Cada afirmação relevante recebe um rótulo de proveniência:

| Rótulo | Significado |
|---|---|
| `DECIDIDO` | Decisão confirmada na reunião; fonte `[hh:mm] Nome` e ADR correspondente |
| `ACORDADO` | Detalhe proposto por um participante e aceito sem objeção |
| `CÓDIGO` | Fato verificado no commit `e7f6311` (caminho + símbolo) |
| `PROPOSTA_TÉCNICA` | Desenho deste FDD para viabilizar a implementação; **não** foi decidido na reunião |
| `PROPOSTA_PARA_REVISAO` | Controle de segurança sugerido pela análise; depende da revisão da Sofia ([09:46] Sofia) |
| `NEEDS_DECISION` | Ponto que exige decisão da equipe antes de implementar; ver §15 |

Nomes de tabelas, colunas, rotas e campos marcados como `PROPOSTA_TÉCNICA` podem mudar sem conflito com as decisões da reunião.

---

## 1. Contexto e motivação técnica

- Três clientes B2B consultam `GET /orders` periodicamente para descobrir mudanças de status; querem notificação em "tempo real", definido como **abaixo de 10 segundos**. É um requisito dos clientes relatado pelo PM ([09:00] Marcos, [09:02] Marcos); a equipe considerou o polling de 2 s suficiente para ele ([09:09] Diego, [09:10] Marcos). Os webhooks são **apenas de saída** ([09:02] Marcos, [09:03] Sofia).
- `CÓDIGO`: a mudança de status ocorre em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 126–179), dentro de `this.prisma.$transaction(async (tx) => …)`. A transação valida a transição (`canTransition`), debita ou repõe estoque (`shouldDebitStock`/`shouldReplenishStock`), atualiza `orders.status` e grava `order_status_history`.
- `CÓDIGO`: não existe hoje nenhuma chamada HTTP de saída, fila, evento ou job em `src/`.
- Chamar o cliente de forma síncrona dentro dessa transação foi descartado: um cliente lento travaria mudanças de status e não há como desfazer a mudança se o cliente estiver fora ([09:04] Bruno, [09:06] Diego; [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md)).

## 2. Objetivos técnicos

| ID | Objetivo | Fonte |
|---|---|---|
| OT-01 | Registrar o evento de mudança de status **atomicamente** com a própria mudança (se a transação confirma, o evento existe; se desfaz, o evento some) | `DECIDIDO` [09:06] Diego, [09:08] Larissa, [09:40] Bruno; ADR-001 |
| OT-02 | Entregar o evento ao endpoint HTTPS do cliente por um **worker em processo separado**, com polling de 2 s | `DECIDIDO` [09:09] Diego, [09:10] Larissa, [09:11] Diego; ADR-002 |
| OT-03 | Atender a meta de entrega em menos de 10 s em operação normal (a reunião considerou 2 s de polling suficiente) | [09:02] Marcos, [09:09] Diego, [09:10] Marcos |
| OT-04 | Assinar cada entrega com **HMAC-SHA256** usando secret própria do endpoint, com rotação e 24 h de convivência | `DECIDIDO` [09:22] Sofia; ADR-004 |
| OT-05 | Garantir entrega **at-least-once** com `X-Event-Id` para deduplicação pelo cliente | `DECIDIDO` [09:26] Larissa; ADR-005 |
| OT-06 | Retentar com backoff (5 tentativas; 1m/5m/30m/2h/12h), mover para DLQ e permitir replay manual por ADMIN | `DECIDIDO` [09:17] Larissa, [09:19] Larissa, [09:36] Larissa; ADR-003 |
| OT-07 | Reaproveitar a arquitetura modular, `AppError`, Pino, `errorMiddleware`, Zod e `requireRole` | `DECIDIDO` [09:30] Larissa (reuso) e [09:36] Larissa (`requireRole`); ADR-006 |

**Não-objetivos** (não prometidos): exactly-once ([09:25] Diego); ordenação global ([09:13] Larissa); vários workers ou escala horizontal (adiado, [09:13] Diego). Alta disponibilidade do worker não foi discutida e também não é objetivo desta fase.

## 3. Escopo e exclusões

**Incluso**

- CRUD de configuração de webhook por customer, com filtro de status ([09:31] Marcos, [09:33] Bruno, [09:33] Marcos).
- Geração da secret pela plataforma e rotação ([09:31] Marcos, [09:21] Sofia).
- Gravação na outbox dentro de `changeStatus` ([09:40] Bruno).
- Worker, retry, DLQ e replay administrativo ([09:09] Diego, [09:17] Larissa, [09:18] Diego).
- Histórico das últimas 100 entregas por webhook ([09:34] Marcos).

**Fora do escopo**

| Item | Situação | Fonte |
|---|---|---|
| Email ao cliente quando o webhook falha | Adiado para a próxima fase | [09:37] Larissa |
| Dashboard visual | Fora do escopo (projeto do time de frontend) | [09:40] Larissa |
| Webhooks de entrada | Fora do escopo | [09:02] Marcos |
| Arquivamento de linhas entregues (~30 dias) | Fora desta feature | [09:08] Diego |
| Vários workers e particionamento | Adiado | [09:13] Diego |
| Itens do pedido no payload | Excluídos; o cliente consulta `GET /orders/:id` | [09:43] Diego |

**Em avaliação, não excluído:** rate limiting de saída. A reunião não o aprovou, não o rejeitou nem o adiou em definitivo; ficou como "observar e decidir depois" ([09:39] Larissa) e foi registrado como ponto em aberto ([09:39] Diego). É o mesmo tratamento do PRD §6 e do RFC §5.

## 4. Arquitetura e componentes

```text
                    ┌──────────────────────── processo API (src/server.ts) ───────────────────────┐
 PATCH /orders/:id/status ─► OrderService.changeStatus ── $transaction(tx) ─────────────────────┐ │
                    │            ├─ update orders / insert order_status_history (existente)      │ │
                    │            └─ publishWebhookEvent(tx, order, from, to) ─► INSERT webhook_outbox│
                    │                                                                            │ │
 /api/v1/customers/:customerId/webhooks, /api/v1/webhooks/:id… ─► módulo webhooks (CRUD/consulta)  │
 /api/v1/admin/webhooks/dead-letter/:id/replay ─► requireRole('ADMIN') ─► replay                  │
                    └────────────────────────────────────────────────────────────────────────────┘
                                          │ MySQL (mesmo banco, mesma DATABASE_URL)
                    ┌──────────────── processo worker (src/worker.ts) ─────────────────┐
                    │ loop a cada 2 s: busca pendentes ► assina HMAC ► POST HTTPS (10 s) │
                    │ ► registra entrega ► entregue | reagenda | move para DLQ          │
                    └───────────────────────────────────────────────────────────────────┘
```

| Componente | Local (futuro) | Responsabilidade | Fonte |
|---|---|---|---|
| Módulo webhooks | `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts` | CRUD, rotação, histórico e replay | `DECIDIDO` [09:27] Bruno, [09:30] Larissa; ADR-006 |
| `publishWebhookEvent(tx, order, fromStatus, toStatus)` | dentro do módulo webhooks (arquivo: `PROPOSTA_TÉCNICA`, por exemplo `webhook.publisher.ts`) | Filtrar interessados e inserir na outbox usando o `tx` recebido | `ACORDADO` [09:41] Bruno, [09:41] Diego |
| Processador de entregas | `src/modules/webhooks/webhook.worker.ts` **ou** `webhook.processor.ts` | Ciclo de polling, envio, retry e DLQ | `ACORDADO` [09:28] Bruno; nome em aberto (P-14) |
| Entry point do worker | `src/worker.ts` + script `npm run worker` | Criar `PrismaClient` próprio, iniciar o loop e encerrar de forma graciosa | `ACORDADO` [09:11] Larissa, [09:28] Bruno, [09:30] Bruno |

## 5. Modelo de dados proposto

Todo o modelo é `PROPOSTA_TÉCNICA`, com estas exceções:
- `DECIDIDO`: a existência da outbox ([09:06] Diego, [09:08] Larissa) e da tabela de DLQ separada ([09:18] Diego, [09:48] Larissa);
- `ACORDADO`: os campos citados na reunião (url, secret, customer_id, ativo, em [09:21] Bruno e [09:21] Sofia; payload, motivo e timestamp da DLQ, em [09:18] Diego);
- `ACORDADO`: IDs UUID ([09:51] Larissa; ADR-001 item 6);
- `DECIDIDO`: payload como snapshot ([09:52] Bruno).

O índice em `status` (pendente, processando, falhou, entregue) e em `created_at` e a leitura de pendentes em batch pequeno foram descritos por Diego ([09:08] Diego) sem fechamento formal específico. Permanecem `PROPOSTA_TÉCNICA`, alinhados ao ADR-001 item 7.

As convenções seguem `prisma/schema.prisma`: `@id @default(uuid()) @db.Char(36)`, `@@map` em snake_case, `createdAt`/`updatedAt` e `@@index` explícitos.

```prisma
// PROPOSTA_TÉCNICA — esboço para revisão; não aplicar sem aprovação (rule 07).
enum WebhookOutboxStatus {
  PENDING      // "pendente"    [09:08] Diego
  PROCESSING   // "processando" [09:08] Diego
  FAILED       // "falhou"      [09:08] Diego — papel em aberto (P-15)
  DELIVERED    // "entregue"    [09:08] Diego
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(255)  // forma de armazenamento: P-05
  previousSecret          String?   @db.VarChar(255)  // válida até previousSecretExpiresAt
  previousSecretExpiresAt DateTime?
  statuses                Json                        // array de OrderStatus
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer Customer @relation(fields: [customerId], references: [id], onDelete: Cascade)

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @db.Char(36)   // X-Event-Id; ver P-11 (fan-out)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)   // sem FK: snapshot sobrevive à exclusão do pedido
  eventType     String              @db.VarChar(64) // "order.status_changed"
  payload       Json                               // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  @@unique([eventId, webhookId])
  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id           String    @id @default(uuid()) @db.Char(36)
  outboxId     String    @db.Char(36)
  eventId      String    @db.Char(36)
  webhookId    String    @db.Char(36)
  payload      Json
  reason       String    @db.VarChar(500)
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)
  createdAt    DateTime  @default(now())

  @@index([webhookId, failedAt])
  @@map("webhook_dead_letter")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)
  attempt        Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text   // armazenamento truncado: PROPOSTA_PARA_REVISAO (§9)
  durationMs     Int
  errorReason    String?  @db.VarChar(500)
  createdAt      DateTime @default(now())

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}
```

**Notas de desenho** (`PROPOSTA_TÉCNICA`, salvo indicação):

- **Uma linha de outbox por par (evento, webhook interessado).** Isso permite retry e DLQ independentes por endpoint e preenche `X-Webhook-Id` ([09:44] Sofia). O `eventId` se repete entre endpoints do mesmo evento, e `@@unique([eventId, webhookId])` impede inserção duplicada. A decisão está em aberto: P-11.
- O payload já está na outbox. O histórico de entregas ([09:34] Marcos: "sucesso/falha, payload, response, tempo de resposta") junta `WebhookDelivery` com o payload da linha de origem.
- `orderId` fica sem FK para que `DELETE /orders/:id` (permitido em PENDING/CANCELLED, `order.service.ts` linhas 181–192) não seja bloqueado nem apague eventos já gerados.
- A configuração tem `onDelete: Cascade` em relação a `Customer`. Hoje `CustomerService.delete` remove o customer, e uma FK `RESTRICT` geraria erro Prisma `P2003`, que o `errorMiddleware` não trata (cairia em 500).
- **Migração:** somente aditiva (novas tabelas, um enum e a relação reversa `Customer.webhooks`, que não altera colunas de `customers`). Nenhuma alteração destrutiva. A migration deve ser gerada e revisada antes de aplicar; está fora deste documento.

## 6. Fluxos detalhados

### 6.1 Cadastro, edição, listagem e exclusão de webhooks

| Passo | Comportamento | Fonte |
|---|---|---|
| 1 | `authenticate` valida o JWT; **qualquer role autenticada** pode usar o CRUD ("por enquanto") | `DECIDIDO` [09:37] Sofia; ADR-006 |
| 2 | `validate({ params, body })` valida o formato com Zod: `url` string; `statuses` array não vazio de `OrderStatus` (padrão `z.nativeEnum(OrderStatus)` de `order.schemas.ts`); `customerId` UUID | `ACORDADO` [09:30] Larissa; `CÓDIGO` |
| 3 | O service confirma que o customer existe; se não existir, `WEBHOOK_CUSTOMER_NOT_FOUND` | `PROPOSTA_TÉCNICA` |
| 4 | Somente URLs `https` são aceitas | `ACORDADO` [09:23] Sofia (ADR-004 item 6) |
| 4a | A checagem é feita no service e emite `WEBHOOK_INVALID_URL` (ver §6.2) | `PROPOSTA_TÉCNICA` (opção P1); `NEEDS_DECISION` (P-02, I-2) |
| 5 | Na criação, gera a secret (§6.3) e devolve-a **uma única vez** na resposta 201 | `ACORDADO` [09:31] Marcos; devolver só uma vez é `PROPOSTA_PARA_REVISAO` |
| 6 | Edição (PATCH): `url`, `statuses` e `active`; a secret não é editável por PATCH (só por rotação) | `PROPOSTA_TÉCNICA` |
| 7 | Exclusão (DELETE): remove a configuração; destino dos eventos pendentes desse webhook em aberto (P-16) | `ACORDADO` [09:33] Bruno; destino: `NEEDS_DECISION` |
| 8 | Listagem: webhooks de um customer, **sem** a secret | `ACORDADO` [09:33] Bruno; omissão da secret: `PROPOSTA_PARA_REVISAO` |

O `customer_id` é informado no **path** ou no **body**, nunca derivado do JWT ([09:32] Larissa). Este FDD adota o path como `PROPOSTA_TÉCNICA` (`/customers/:customerId/webhooks`); a escolha final é `NEEDS_DECISION` (P-13).

### 6.2 Validação de URL HTTPS

- `ACORDADO`: URL `http` é recusada com erro de validação. Sofia classificou isso como "só uma validação no schema Zod" ([09:23] Sofia).
- `CÓDIGO` (I-2): se a regra ficar só no schema Zod, o `validate` converte o erro em `ValidationError` com código fixo `VALIDATION_ERROR` (`src/middlewares/validate.middleware.ts`; `src/shared/errors/http-errors.ts`). Nesse caso `WEBHOOK_INVALID_URL` ([09:28] Bruno) nunca seria emitido.
- `PROPOSTA_TÉCNICA` (opção P1 do ADR-006):
  - o Zod valida o formato com `z.string().url().max(2048)`;
  - o service verifica `new URL(url).protocol === 'https:'` e lança `WebhookInvalidUrlError` com `WEBHOOK_INVALID_URL`;
  - o `validate` compartilhado não é alterado.
  - P1 se afasta do "no schema Zod" literal de [09:23] Sofia, porque move a regra `https` para o service. É justamente o ponto que precisa de decisão.
- `NEEDS_DECISION` (I-2): adotar P1, P2 ou P3; status 400 ou 422.

### 6.3 Geração e rotação de secrets

| Aspecto | Desenho | Classificação |
|---|---|---|
| Geração | `crypto.randomBytes(32)` em hexadecimal (`node:crypto`, sem dependência nova) | `PROPOSTA_TÉCNICA`; revisão da Sofia ([09:46] Sofia) |
| Escopo | Uma secret por endpoint | `DECIDIDO` [09:21] Sofia |
| Rotação | O cliente pede uma nova secret pela API. A atual vira `previousSecret`, com `previousSecretExpiresAt = agora + 24h`. Uma nova é gerada e devolvida uma única vez. | Rotação e 24 h: `DECIDIDO` [09:21] Sofia. Mecânica: `PROPOSTA_TÉCNICA` |
| Expiração | Após `previousSecretExpiresAt`, a secret anterior é descartada (no próximo uso ou numa limpeza periódica) | `PROPOSTA_TÉCNICA` |
| Assinatura durante as 24 h | Qual secret assina os novos envios | `NEEDS_DECISION` (P-06) |
| Armazenamento em repouso | A secret precisa ser recuperável para o HMAC; não pode ser só hash | `NEEDS_DECISION` (P-05) |

### 6.4 Mudança de status do pedido e inserção atômica na outbox

Sequência dentro do `$transaction` existente de `OrderService.changeStatus`. Os passos 1–5 são `CÓDIGO` atual e o passo 6 é a extensão:

1. `tx.order.findUnique({ where: { id }, include: { items: true } })` (linhas 132–135).
2. Mesmo status gera `ConflictError(..., 'INVALID_STATUS_TRANSITION')`; transição inválida gera `InvalidStatusTransitionError` (linhas 140–149).
3. Débito ou reposição de estoque conforme `shouldDebitStock`/`shouldReplenishStock` (linhas 151–156).
4. `tx.order.update` com o novo status (linha 158).
5. `tx.orderStatusHistory.create` (linhas 159–167).
6. **Novo**, `ACORDADO` [09:41] Bruno: `await publishWebhookEvent(tx, order, from, to)`, chamado **antes** do `findUnique` de refresh (linha 169):
   1. Busca, **pelo `tx`**, os `WebhookEndpoint` ativos de `order.customerId` cujo `statuses` contém `to`.
   2. Se não houver nenhum, retorna sem inserir ("Se nenhum webhook do customer quer aquele status, nem insere", [09:34] Bruno).
   3. Gera **um** `eventId` (UUID) para o evento ([09:25] Diego) e renderiza o snapshot (§6.5).
   4. Aplica o limite de 64 KB se a decisão P-10 escolher a inserção.
   5. Insere uma linha `PENDING` em `webhook_outbox` por webhook interessado (P-11), com `attempts = 0` e `nextAttemptAt = agora`.
7. Qualquer exceção nos passos 1–6 propaga pelo callback e o Prisma desfaz a transação. Não pode haver mudança de status sem evento registrado (`DECIDIDO` [09:08] Larissa, [09:40] Bruno, [09:41] Diego, [09:48] Larissa "transação atômica com mudança de status").

**Invariantes** (verificados pelos testes em §14):
- status confirmado ⇔ linha(s) de outbox confirmada(s), desde que haja webhook interessado;
- rollback ⇒ nenhuma linha de outbox.

**Atenção** (`CÓDIGO` e inferência):
- A transação ganha uma leitura e N inserções. O Prisma 5.x tem tempo-limite padrão para `$transaction` interativa; o valor para a versão 5.22.0 deve ser confirmado e, se necessário, ajustado com evidência (rule 24).
- `findUnique` sem lock (linha 132) permite que duas mudanças concorrentes no mesmo pedido passem por `canTransition` (P-17, pré-existente).

### 6.5 Snapshot do payload

`ACORDADO` [09:43] Diego, [09:44] Bruno; snapshot `DECIDIDO` [09:52] Bruno. O payload é renderizado na inserção e nunca reconstruído no envio. Os nomes em snake_case seguem a fala de [09:43] Diego.

```json
{
  "event_id": "c3a1e7d2-8f4b-4a6c-9e1d-2b7f5a3c8d4e",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-16T12:34:56.789Z",
  "order_id": "1f4e6a2b-7c9d-4e3f-8a1b-5c6d7e8f9a0b",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "5b0c2f8e-3d4a-4c1e-9f6a-2a7d8e9b1c3d",
  "total_cents": 15990
}
```

- `timestamp` = instante da mudança de status, em ISO 8601 UTC. Usar a mudança e não a inserção é `PROPOSTA_TÉCNICA`.
- "Campos básicos da order **tipo** total_cents" ([09:43] Diego): só `total_cents` foi nomeado. Incluir `subtotal_cents`/`discount_cents` (existentes em `Order`) é `NEEDS_DECISION` (P-18).
- `order_number` segue o formato `ORD-NNNNNN` de `reserveOrderNumber` (`CÓDIGO`).

### 6.6 Filtragem por status

- `ACORDADO`: o filtro é a lista de status do webhook, aplicada na inserção ([09:33] Marcos, [09:34] Bruno, [09:34] Diego).
- `CÓDIGO`: os valores possíveis são os do enum `OrderStatus` (`prisma/schema.prisma`).
- Pela tabela `transitions` (`src/modules/orders/order.status.ts`), `PENDING` nunca é **destino** de `changeStatus`; só aparece na criação do pedido. Um filtro contendo `PENDING` nunca dispara, a menos que P-09 decida emitir evento na criação. Opções `PROPOSTA_TÉCNICA`: rejeitar `PENDING` no schema ou aceitá-lo e documentar o efeito.
- O filtro é avaliado no momento da inserção. Um webhook criado ou alterado depois não recebe eventos já inseridos (consequência registrada no ADR-001).

### 6.7 Worker e polling de 2 segundos

- `DECIDIDO`: processo separado e polling de 2 s ([09:09] Diego, [09:10] Larissa, [09:11] Diego, [09:12] Larissa, [09:48] Larissa; ADR-002).
- `ACORDADO`: worker único, com ordenação por pedido como limitação conhecida ([09:12] Diego, [09:13] Larissa; ADR-002 item 5).

Os passos 1–5 são `PROPOSTA_TÉCNICA`:

1. **Bootstrap** (`src/worker.ts`): `createPrismaClient()` (`src/config/database.ts`) e logger Pino; tratar `SIGINT`/`SIGTERM` como em `src/server.ts` (terminar a entrega em curso, `$disconnect`, sair).
2. **Tick** a cada 2 s, sem sobreposição: o próximo tick só é agendado quando o anterior termina.
3. **Busca:** `status = PENDING AND nextAttemptAt <= now`, ordenado por `createdAt ASC`, `LIMIT N` ("batch pequeno", [09:08] Diego; valor de N: `NEEDS_DECISION`, P-19).
4. **Reivindicação:** `UPDATE … SET status = PROCESSING WHERE id = ? AND status = PENDING`. Se nenhuma linha for afetada, outro processo já pegou o evento. Com worker único é uma defesa, não um requisito.
5. **Para cada evento:** entrega (§6.8), registro em `webhook_deliveries` e transição de estado (§6.9).
6. **Recuperação de `PROCESSING` órfão** após queda: `NEEDS_DECISION` (P-12).

**Máquina de estados da linha de outbox:**

```text
PENDING ──claim──► PROCESSING ──2xx──► DELIVERED
   ▲                    │
   └──falha, ainda há───┤
      tentativa         └──falha, política esgotada (I-1)──► DLQ (+ FAILED ou remoção: P-15)
```

### 6.8 Entrega HTTP assinada (HMAC-SHA256)

| Elemento | Valor | Fonte |
|---|---|---|
| Método e destino | `POST <url do webhook>` (somente `https`) | [09:23] Sofia |
| Corpo | Payload snapshot (§6.5) serializado como JSON. **Os mesmos bytes** são assinados e enviados. | HMAC sobre o corpo: `DECIDIDO` [09:22] Sofia |
| `Content-Type` | `application/json` | `ACORDADO` [09:44] Diego |
| `X-Event-Id` | `eventId` (UUID gerado na inserção) | `DECIDIDO` [09:25] Diego, [09:26] Larissa |
| `X-Webhook-Id` | `id` do `WebhookEndpoint` | `ACORDADO` [09:44] Sofia, [09:45] Diego |
| `X-Timestamp` | Instante do envio | `ACORDADO` [09:44] Diego. Formato (ISO 8601 ou epoch): `PROPOSTA_TÉCNICA` ISO 8601 |
| `X-Signature` | `HMAC-SHA256(secret, corpo)` em hexadecimal | Algoritmo sobre o corpo: `DECIDIDO` [09:22] Sofia. Nome do header: `ACORDADO` [09:20] Sofia, [09:44] Diego. Codificação e prefixo (por exemplo `sha256=`): `PROPOSTA_TÉCNICA`. Inclusão do timestamp: `NEEDS_DECISION` (P-07) |
| Timeout | 10 s; estourou = falha com retry | `ACORDADO` [09:42] Diego, [09:42] Sofia |
| Limite de tamanho | 64 KB; acima disso, erro sem truncar | `ACORDADO` como requisito não funcional ([09:24] Larissa: "Anotado, mas não vejo como decisão arquitetural separada"). Onde é aplicado: P-10 |
| Cliente HTTP | `fetch` global do Node ≥ 20 (`package.json`, `engines.node`) com `AbortSignal.timeout(10000)` e `redirect: 'manual'` | `PROPOSTA_TÉCNICA`. `redirect: 'manual'` é `PROPOSTA_PARA_REVISAO` (SSRF) |
| Critério de sucesso | Resposta 2xx | `PROPOSTA_TÉCNICA`; `NEEDS_DECISION` (P-20) |

Exemplo da requisição de saída (`PROPOSTA_TÉCNICA` quanto aos formatos):

```http
POST /hooks/oms HTTP/1.1
Host: integracao.cliente.example
Content-Type: application/json
X-Event-Id: c3a1e7d2-8f4b-4a6c-9e1d-2b7f5a3c8d4e
X-Webhook-Id: 9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c
X-Timestamp: 2026-11-16T12:34:57.120Z
X-Signature: sha256=<hex de 64 caracteres calculado sobre o corpo>

{"event_id":"c3a1e7d2-8f4b-4a6c-9e1d-2b7f5a3c8d4e","event_type":"order.status_changed","timestamp":"2026-11-16T12:34:56.789Z","order_id":"1f4e6a2b-7c9d-4e3f-8a1b-5c6d7e8f9a0b","order_number":"ORD-000123","from_status":"PAID","to_status":"PROCESSING","customer_id":"5b0c2f8e-3d4a-4c1e-9f6a-2a7d8e9b1c3d","total_cents":15990}
```

### 6.9 Retry com backoff e transferência para a DLQ

- `DECIDIDO`: 5 tentativas; intervalos 1m / 5m / 30m / 2h / 12h; DLQ ao esgotar a política ([09:17] Larissa, [09:48] Larissa; ADR-003).
- **Em falha** (timeout, erro de rede, resposta fora do critério de sucesso):
  - incrementar `attempts`;
  - gravar `lastError` com um motivo curto, sem segredos;
  - registrar a tentativa em `webhook_deliveries`.
- **Se a política não esgotou:** `status = PENDING` e `nextAttemptAt = agora + intervalo[k]`.
- **Se esgotou:** numa **única transação**, inserir `webhook_dead_letter` (payload, motivo, `failedAt`) e marcar a linha de outbox como `FAILED` (ou removê-la, P-15).
- **`NEEDS_DECISION` (I-1):** o índice `k` e o limiar de esgotamento dependem de como se conta "5 tentativas".

| Leitura | Envios totais | Intervalos usados | Janela até a DLQ |
|---|---|---|---|
| A — 5 envios no total | 5 | 1m, 5m, 30m, 2h | 2h36min |
| B — 5 retries após o envio inicial | 6 | 1m, 5m, 30m, 2h, 12h | 14h36min ("quase 15 horas", [09:17] Diego) |

- A implementação deve manter a sequência `[1m, 5m, 30m, 2h, 12h]` como configuração única e o limiar como parâmetro explícito, e **não** escolher A ou B sem decisão de Larissa e Diego (ADR-003). Os testes de fronteira da §14 ficam bloqueados até lá.
- **Falhas que não têm retry** (`PROPOSTA_TÉCNICA`, a decidir): payload acima de 64 KB detectado no envio, se P-10 escolher o envio; webhook inativo ou removido (P-16); webhook sem secret utilizável (possível uso de `WEBHOOK_SECRET_REQUIRED`, P-04).

### 6.10 Replay manual e auditoria

| Passo | Comportamento | Fonte |
|---|---|---|
| 1 | `authenticate` + `requireRole('ADMIN')` | `DECIDIDO` [09:36] Larissa |
| 2 | Localiza o registro de DLQ; se não existir, `WEBHOOK_DEAD_LETTER_NOT_FOUND` | `PROPOSTA_TÉCNICA` |
| 3 | Se já foi reprocessado (`replayedAt` preenchido), `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | `PROPOSTA_TÉCNICA` |
| 4 | Numa transação: insere nova linha `PENDING` em `webhook_outbox` com o mesmo payload, `attempts = 0`; marca `replayedAt` e `replayedById = req.user.id` | Recolocar "na outbox como pendente": `ACORDADO` [09:18] Diego. Colunas de auditoria: `PROPOSTA_TÉCNICA` |
| 5 | `eventId` da nova linha: **preservar o original** (opção recomendada) ou gerar um novo | `NEEDS_DECISION` (P-08) |
| 6 | Log Pino `webhook_dead_letter_replayed` com `userId`, `deadLetterId`, `eventId`, `webhookId` e `requestId` (mesmos campos da §11) | Registrar quem fez: `ACORDADO` [09:36] Sofia. Mecanismo: `PROPOSTA_TÉCNICA` |

Com `@@unique([eventId, webhookId])`, preservar o `eventId` exige remover a linha antiga ao mover para a DLQ, ou reutilizá-la no replay. É mais um motivo para decidir P-08 e P-15 juntos.

### 6.11 Consulta do histórico de entregas

`ACORDADO` [09:34] Marcos: "os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta".
- `PROPOSTA_TÉCNICA`: ler `webhook_deliveries` por `webhookId`, ordenar por `createdAt DESC`, `LIMIT 100`, juntando o payload da outbox pelo `eventId`.
- `PROPOSTA_TÉCNICA`: sem paginação além de 100 itens nesta fase, porque a fonte fala em "últimos 100".
- Interpretação em aberto (P-25, baixo impacto): a fala pode significar 100 **tentativas por webhook** (adotado aqui, coerente com a rota `/webhooks/:id/deliveries`) ou 100 **eventos por customer** ("pra mim").

## 7. Contratos públicos HTTP

**Convenções do código** (`CÓDIGO`):
- prefixo `/api/v1` (`src/app.ts`, `app.use('/api/v1', buildApiRouter(controllers))`);
- autenticação `Authorization: Bearer <jwt>` (`authenticate`);
- erros no formato `{ "error": { "code", "message", "details"? } }` (`errorMiddleware`);
- campos da API em camelCase (por exemplo `customerId`, `toStatus` em `order.schemas.ts`);
- exclusão com `204` (`OrderController.delete`);
- respostas de criação com `201` e o objeto criado.

A API de configuração usa camelCase. O payload de saída usa snake_case ([09:43] Diego). As duas convenções convivem por terem consumidores distintos.

**Resumo:**

| # | Método e caminho | Autorização | Origem do caminho |
|---|---|---|---|
| E1 | `POST /api/v1/customers/:customerId/webhooks` | Qualquer role autenticada | Operação `ACORDADO` [09:31] Marcos; caminho `PROPOSTA_TÉCNICA` (P-13) |
| E2 | `GET /api/v1/customers/:customerId/webhooks` | Qualquer role autenticada | Operação `ACORDADO` [09:33] Bruno; caminho `PROPOSTA_TÉCNICA` |
| E3 | `PATCH /api/v1/webhooks/:id` | Qualquer role autenticada | Operação `ACORDADO` [09:33] Bruno; caminho `PROPOSTA_TÉCNICA` |
| E4 | `DELETE /api/v1/webhooks/:id` | Qualquer role autenticada | Operação `ACORDADO` [09:33] Bruno; caminho `PROPOSTA_TÉCNICA` |
| E5 | `POST /api/v1/webhooks/:id/secret/rotate` | Qualquer role autenticada | Operação `DECIDIDO` [09:21] Sofia; caminho `PROPOSTA_TÉCNICA` |
| E6 | `GET /api/v1/webhooks/:id/deliveries` | Qualquer role autenticada | `ACORDADO` [09:34] Marcos (caminho citado) + prefixo `CÓDIGO` |
| E7 | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` | **ADMIN** (`requireRole`) | Replay e role ADMIN: `DECIDIDO` [09:19] Larissa ("Anota também"), [09:36] Larissa, [09:48] Larissa. Caminho: `ACORDADO` [09:18] Diego, [09:35] Diego. Prefixo: `CÓDIGO` |

Para E1–E6, "qualquer role autenticada" reflete a decisão "por enquanto" de [09:37] Sofia. A ausência de verificação por objeto (G-12) está na §9.

### E1 — Criar webhook

- **Headers:** `Authorization: Bearer <jwt>`, `Content-Type: application/json`.
- **Parâmetros:** `customerId` (path, UUID).
- **Body:** `url` (string, https, ≤ 2048), `statuses` (array não vazio de `OrderStatus`), `active` (boolean, opcional, padrão `true`).

```http
POST /api/v1/customers/5b0c2f8e-3d4a-4c1e-9f6a-2a7d8e9b1c3d/webhooks
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "url": "https://integracao.cliente.example/hooks/oms",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

`201 Created`. A secret aparece **somente** nesta resposta e na de E5:

```json
{
  "id": "9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c",
  "customerId": "5b0c2f8e-3d4a-4c1e-9f6a-2a7d8e9b1c3d",
  "url": "https://integracao.cliente.example/hooks/oms",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "EXEMPLO-NAO-REAL-4f1c9a7e2b8d3f6a0c5e9b1d7a3f8c2e6b0d4a9f1e7c3b5d8a2f6e0c4b9d1a7e",
  "createdAt": "2026-11-16T12:00:00.000Z",
  "updatedAt": "2026-11-16T12:00:00.000Z"
}
```

| Status | Código | Situação |
|---|---|---|
| 400 | `WEBHOOK_INVALID_URL` | URL não-https (§6.2; depende de I-2) |
| 400 | `VALIDATION_ERROR` | Formato inválido detectado pelo `validate` (por exemplo `statuses` vazio) |
| 401 | `UNAUTHORIZED` | JWT ausente ou inválido (`authenticate`) |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` | Customer inexistente |

### E2 — Listar webhooks de um customer

- **Headers:** `Authorization: Bearer <jwt>`.
- **Parâmetros:** `customerId` (path, UUID).

```http
GET /api/v1/customers/5b0c2f8e-3d4a-4c1e-9f6a-2a7d8e9b1c3d/webhooks
Authorization: Bearer <jwt>
```

`200 OK`. A secret não é devolvida (`PROPOSTA_PARA_REVISAO`):

```json
{
  "data": [
    {
      "id": "9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c",
      "customerId": "5b0c2f8e-3d4a-4c1e-9f6a-2a7d8e9b1c3d",
      "url": "https://integracao.cliente.example/hooks/oms",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotationPending": false,
      "createdAt": "2026-11-16T12:00:00.000Z",
      "updatedAt": "2026-11-16T12:00:00.000Z"
    }
  ]
}
```

- Envelope `{ data }`: `PROPOSTA_TÉCNICA`. Paginação com `paginated()` (`src/shared/http/response.ts`) é opcional e não foi discutida.
- `secretRotationPending` indica se há secret anterior dentro das 24 h (`PROPOSTA_TÉCNICA`).
- **Erros:** 400 `VALIDATION_ERROR` (UUID inválido); 401 `UNAUTHORIZED`; 404 `WEBHOOK_CUSTOMER_NOT_FOUND`.

### E3 — Editar webhook

- **Headers:** `Authorization: Bearer <jwt>`, `Content-Type: application/json`.
- **Parâmetros:** `id` (path, UUID).
- **Body (parcial):** `url`, `statuses` e `active`. Pelo menos um campo.

```http
PATCH /api/v1/webhooks/9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "statuses": ["PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true
}
```

`200 OK`:

```json
{
  "id": "9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c",
  "customerId": "5b0c2f8e-3d4a-4c1e-9f6a-2a7d8e9b1c3d",
  "url": "https://integracao.cliente.example/hooks/oms",
  "statuses": ["PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true,
  "secretRotationPending": false,
  "createdAt": "2026-11-16T12:00:00.000Z",
  "updatedAt": "2026-11-16T13:10:00.000Z"
}
```

**Erros:** 400 `WEBHOOK_INVALID_URL`; 400 `VALIDATION_ERROR`; 401 `UNAUTHORIZED`; 404 `WEBHOOK_NOT_FOUND` ([09:28] Bruno).

### E4 — Remover webhook

- **Headers:** `Authorization: Bearer <jwt>`.
- **Parâmetros:** `id` (path, UUID).

```http
DELETE /api/v1/webhooks/9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c
Authorization: Bearer <jwt>
```

- `204 No Content`, sem corpo (padrão de `OrderController.delete`).
- **Erros:** 401 `UNAUTHORIZED`; 404 `WEBHOOK_NOT_FOUND`.
- O tratamento de eventos `PENDING` desse webhook é `NEEDS_DECISION` (P-16).

### E5 — Rotacionar a secret

- **Headers:** `Authorization: Bearer <jwt>`.
- **Parâmetros:** `id` (path, UUID).
- **Body:** vazio.

```http
POST /api/v1/webhooks/9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c/secret/rotate
Authorization: Bearer <jwt>
```

`200 OK`:

```json
{
  "id": "9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c",
  "secret": "EXEMPLO-NAO-REAL-8b2e6d1f9a4c7e3b0d5f8a2c6e9b1d4f7a0c3e8b5d2f9a6c1e4b7d0a3f8c5e2b",
  "previousSecretExpiresAt": "2026-11-17T12:00:00.000Z"
}
```

| Status | Código | Situação |
|---|---|---|
| 401 | `UNAUTHORIZED` | JWT ausente ou inválido |
| 404 | `WEBHOOK_NOT_FOUND` | Webhook inexistente |
| 409 | `WEBHOOK_ROTATION_IN_PROGRESS` | Já existe secret anterior dentro das 24 h. Bloquear uma segunda rotação, ou permitir e descartar a anterior, é `NEEDS_DECISION` (P-24). |

### E6 — Histórico de entregas

- **Headers:** `Authorization: Bearer <jwt>`.
- **Parâmetros:** `id` (path, UUID).

```http
GET /api/v1/webhooks/9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c/deliveries
Authorization: Bearer <jwt>
```

`200 OK`, com no máximo 100 itens, do mais recente para o mais antigo:

```json
{
  "data": [
    {
      "id": "2d3e4f5a-6b7c-4d8e-9f0a-1b2c3d4e5f6a",
      "eventId": "c3a1e7d2-8f4b-4a6c-9e1d-2b7f5a3c8d4e",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 184,
      "errorReason": null,
      "createdAt": "2026-11-16T12:36:08.410Z",
      "payload": {
        "event_id": "c3a1e7d2-8f4b-4a6c-9e1d-2b7f5a3c8d4e",
        "event_type": "order.status_changed",
        "timestamp": "2026-11-16T12:34:56.789Z",
        "order_id": "1f4e6a2b-7c9d-4e3f-8a1b-5c6d7e8f9a0b",
        "order_number": "ORD-000123",
        "from_status": "PAID",
        "to_status": "PROCESSING",
        "customer_id": "5b0c2f8e-3d4a-4c1e-9f6a-2a7d8e9b1c3d",
        "total_cents": 15990
      }
    },
    {
      "id": "6f7a8b9c-0d1e-4f2a-8b3c-4d5e6f7a8b9c",
      "eventId": "c3a1e7d2-8f4b-4a6c-9e1d-2b7f5a3c8d4e",
      "attempt": 1,
      "success": false,
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10000,
      "errorReason": "timeout after 10000 ms",
      "createdAt": "2026-11-16T12:34:57.120Z",
      "payload": {
        "event_id": "c3a1e7d2-8f4b-4a6c-9e1d-2b7f5a3c8d4e",
        "event_type": "order.status_changed",
        "timestamp": "2026-11-16T12:34:56.789Z",
        "order_id": "1f4e6a2b-7c9d-4e3f-8a1b-5c6d7e8f9a0b",
        "order_number": "ORD-000123",
        "from_status": "PAID",
        "to_status": "PROCESSING",
        "customer_id": "5b0c2f8e-3d4a-4c1e-9f6a-2a7d8e9b1c3d",
        "total_cents": 15990
      }
    }
  ]
}
```

**Erros:** 400 `VALIDATION_ERROR`; 401 `UNAUTHORIZED`; 404 `WEBHOOK_NOT_FOUND`.

### E7 — Replay de evento da DLQ (administrativo)

- **Headers:** `Authorization: Bearer <jwt de usuário ADMIN>`.
- **Parâmetros:** `id` (path, UUID do registro de DLQ).
- **Body:** vazio.

```http
POST /api/v1/admin/webhooks/dead-letter/7a8b9c0d-1e2f-4a3b-8c4d-5e6f7a8b9c0d/replay
Authorization: Bearer <jwt>
```

`200 OK` (`PROPOSTA_TÉCNICA`; `202` também é defensável porque a entrega é assíncrona). O exemplo assume que o `eventId` original é preservado, o que **ainda depende de P-08**:

```json
{
  "deadLetterId": "7a8b9c0d-1e2f-4a3b-8c4d-5e6f7a8b9c0d",
  "eventId": "c3a1e7d2-8f4b-4a6c-9e1d-2b7f5a3c8d4e",
  "webhookId": "9e2f7a1c-6b3d-4f8e-a1c2-7d4e5f6a8b9c",
  "status": "PENDING",
  "replayedAt": "2026-11-17T09:15:00.000Z",
  "replayedById": "0a1b2c3d-4e5f-4a6b-8c7d-9e0f1a2b3c4d"
}
```

| Status | Código | Situação |
|---|---|---|
| 401 | `UNAUTHORIZED` | JWT ausente ou inválido |
| 403 | `FORBIDDEN` | Role diferente de ADMIN (`requireRole`) |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | Registro inexistente |
| 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | Registro já reprocessado |
| 422 | `WEBHOOK_INACTIVE` | O webhook de destino foi desativado ou removido (depende de P-16) |

Ressalva de segurança: enquanto I-3 existir, o `403` não impede um atacante que tenha se registrado como ADMIN (§9).

## 8. Matriz de erros `WEBHOOK_*`

O prefixo `WEBHOOK_` é `DECIDIDO` para todo o módulo ([09:29] Larissa). Todos os erros são subclasses de `AppError` (`src/shared/errors/app-error.ts`) e saem pelo `errorMiddleware` sem alteração (`ACORDADO` [09:29] Bruno).

**Erros da API (respostas HTTP)**

| Código | HTTP | Classe base proposta | Quando | Fonte |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `AppError` (o `NotFoundError` fixa `NOT_FOUND`) | Webhook inexistente | Código citado [09:28] Bruno |
| `WEBHOOK_INVALID_URL` | 400 (ou 422, I-2) | `BadRequestError` | URL não-https ou inválida | Código citado [09:28] Bruno; regra [09:23] Sofia |
| `WEBHOOK_SECRET_REQUIRED` | — | — | Semântica indefinida: a secret é gerada pela plataforma ([09:31] Marcos). Candidato: falha interna de envio sem secret utilizável | Código citado [09:28] Bruno; uso `NEEDS_DECISION` (P-04) |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `AppError` | Customer inexistente em E1 e E2 | `PROPOSTA_TÉCNICA` |
| `WEBHOOK_ROTATION_IN_PROGRESS` | 409 | `ConflictError` | Rotação durante a janela de 24 h | `PROPOSTA_TÉCNICA` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `AppError` | Registro de DLQ inexistente | `PROPOSTA_TÉCNICA` |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `ConflictError` | Replay repetido | `PROPOSTA_TÉCNICA` |
| `WEBHOOK_INACTIVE` | 422 | `UnprocessableEntityError` | Replay para webhook inativo | `PROPOSTA_TÉCNICA` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 (se aplicado na inserção) | `UnprocessableEntityError` | Snapshot > 64 KB. Se P-10 escolher a inserção, **aborta `changeStatus`**, o que é uma mudança de contrato de `PATCH /orders/:id/status` | Limite `ACORDADO` (NFR) [09:24] Larissa; código e etapa `PROPOSTA_TÉCNICA`/`NEEDS_DECISION` |

**Motivos de falha do worker** (gravados em `webhook_deliveries.errorReason`, `webhook_outbox.lastError` e `webhook_dead_letter.reason`; não são respostas HTTP; `PROPOSTA_TÉCNICA`):

| Código | Situação | Retry? |
|---|---|---|
| `WEBHOOK_DELIVERY_TIMEOUT` | Sem resposta em 10 s ([09:42] Diego) | Sim |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | DNS, conexão recusada, TLS inválido | Sim |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | Resposta fora do critério de sucesso (P-20) | Sim (se 4xx também: P-20) |
| `WEBHOOK_DELIVERY_REDIRECT_BLOCKED` | Resposta 3xx com `redirect: 'manual'` | A decidir (SSRF, §9) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | Se P-10 escolher o envio | Não; vai direto à DLQ |
| `WEBHOOK_RETRIES_EXHAUSTED` | Política esgotada (I-1) | Não; vai para a DLQ |

**Erros genéricos existentes que continuam aparecendo** (`CÓDIGO`): `UNAUTHORIZED` e `FORBIDDEN` (de `auth.middleware.ts`) e `VALIDATION_ERROR` (de `validate.middleware.ts`). Eles vêm de middlewares compartilhados, e alterá-los mudaria o contrato de todos os módulos (ADR-006).

## 9. Segurança e autorização

**Decisões e acordos da reunião:**
- `DECIDIDO`: HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com 24 h ([09:22] Sofia);
- `ACORDADO`: somente HTTPS ([09:23] Sofia);
- replay restrito a ADMIN via `requireRole` ([09:36] Larissa);
- registro de quem fez o replay ([09:36] Sofia);
- CRUD liberado para qualquer role autenticada "por enquanto" ([09:37] Sofia);
- revisão de segurança da Sofia de pelo menos dois dias úteis antes do deploy ([09:46] Sofia, [09:47] Larissa).

**Riscos pré-existentes** (`CÓDIGO`, leitura estática; detalhes no ADR-006):

| ID | Fato | Impacto na feature |
|---|---|---|
| I-3 | `POST /api/v1/auth/register` é público (`auth.routes.ts`); `registerSchema.role` aceita `'ADMIN'` (`auth.schemas.ts`); o role é gravado (`UserService.createUser`) e assinado no JWT (`AuthService.login`) | `requireRole('ADMIN')` no E7 não restringe efetivamente quem faz o replay |
| G-12 | `User` não tem vínculo com `Customer` (`prisma/schema.prisma`); o JWT carrega só `id`, `email` e `role` (`AuthUser`) | Qualquer usuário autenticado pode cadastrar em qualquer customer uma URL própria e receber eventos de pedidos de outro cliente, legitimamente assinados. Como o registro é público, isso vale até para OPERATOR. |

**Controles propostos (`PROPOSTA_PARA_REVISAO`; nenhum foi decidido na reunião; todos dependem da revisão da Sofia):**

| Tema | Proposta |
|---|---|
| Cadastro com ADMIN (I-3) | Impedir `role: 'ADMIN'` no registro público. Mudança fora do escopo da feature, com autorização própria e teste de regressão. Revisar os ADMIN existentes antes do deploy. |
| Isolamento entre clientes (G-12) | Definir vínculo usuário–customer e checar por objeto em E1–E6, **ou** registrar formalmente o risco aceito enquanto valer o "por enquanto" ([09:37] Sofia) |
| Proteção da secret em repouso (P-05) | Cifrar com AES-256-GCM (`node:crypto`), chave em variável de ambiente (nova entrada em `env.ts`), **ou** manter em claro com acesso restrito ao banco |
| Exposição da secret | Devolver só em E1 e E5; nunca em E2, E3 ou E6; nunca em logs |
| Redação de logs | Acrescentar `'*.secret'`, `'*.previousSecret'` e `'headers["x-signature"]'` aos `redactPaths` de `src/shared/logger/index.ts` |
| SSRF | Bloquear destinos que resolvam para IPs privados, loopback ou link-local; `redirect: 'manual'`; resolver DNS no envio; não desativar a verificação TLS |
| HMAC | Recomendar ao cliente comparação em tempo constante, na documentação do portal (o portal foi citado por Marcos em [09:26] só para at-least-once). Do nosso lado, assinar exatamente os bytes enviados. |
| Replay pelo cliente | Incluir o `X-Timestamp` na assinatura para permitir janela de aceitação (P-07) |
| Duplicatas | O `X-Event-Id` é estável por evento; preservá-lo no replay (P-08) |
| Armazenamento de respostas | Truncar `responseBody` (por exemplo 2 KB), porque pode conter dados do cliente |
| Auditoria do replay | Gravar `replayedById`/`replayedAt` e emitir log Pino com `requestId` |

## 10. Estratégias de resiliência

| Mecanismo | Valor | Fonte |
|---|---|---|
| Timeout HTTP | 10 s | `ACORDADO` [09:42] Diego |
| Retry | 5 tentativas; 1m/5m/30m/2h/12h; contagem pendente (I-1) | `DECIDIDO` [09:17] Larissa |
| DLQ | Tabela separada; replay manual | `DECIDIDO` [09:18] Diego, [09:19] Larissa, [09:48] Larissa |
| Fallback | Nenhum nesta fase (email adiado, [09:37] Larissa). A DLQ e o replay são o caminho de recuperação. | Fonte da exclusão |
| Polling | 2 s, sem sobreposição de ticks | `DECIDIDO` [09:10] Larissa; sem sobreposição: `PROPOSTA_TÉCNICA` |
| Banco indisponível no worker | Registrar o erro, manter a linha como está e tentar no próximo tick; sem `process.exit` por falha transitória | `PROPOSTA_TÉCNICA` |
| Crash do worker | Linhas `PENDING` permanecem; linhas `PROCESSING` órfãs precisam de recuperação (P-12) | `PROPOSTA_TÉCNICA` |
| Contenção | Batch pequeno ([09:08] Diego); rate limiting por cliente: observar ([09:39] Larissa) | Fontes indicadas |
| Fila bloqueada por cliente lento | Worker sequencial + timeout de 10 s pode atrasar os demais. Mitigações a avaliar: processar o batch com concorrência limitada por endpoint, ou reduzir N. | Risco de ADR-002; mitigação `PROPOSTA_PARA_REVISAO` |

## 11. Observabilidade

A reunião decidiu manter o **Pino** como logger, sem nada novo ([09:29] Bruno, [09:30] Larissa), e evitar infraestrutura adicional ([09:07] Diego). Métricas e tracing não foram discutidos e não há biblioteca para isso no código. Tudo abaixo, exceto os logs Pino, é `PROPOSTA_TÉCNICA` sem dependência nova.

**Logs** (Pino, `src/shared/logger/index.ts`; mensagens em snake_case como `server_started` e `http_request`):

| Evento | Campos |
|---|---|
| `webhook_event_enqueued` | `eventId`, `orderId`, `webhookIds`, `toStatus` |
| `webhook_delivery_succeeded` / `webhook_delivery_failed` | `eventId`, `webhookId`, `attempt`, `responseStatus`, `durationMs`, `errorReason` |
| `webhook_moved_to_dead_letter` | `eventId`, `webhookId`, `attempts`, `reason` |
| `webhook_dead_letter_replayed` | `deadLetterId`, `eventId`, `webhookId`, `userId`, `requestId` |
| `webhook_worker_started` / `webhook_worker_stopped` / `webhook_worker_tick_failed` | `batchSize`, `err` |

Nunca registrar a secret ou o corpo completo de respostas (ver redação na §9).

**Métricas** (derivadas do banco e emitidas como log estruturado periódico, por exemplo a cada minuto, até existir um coletor):

| Métrica | Uso |
|---|---|
| `webhook_outbox_pending_count` e `webhook_outbox_oldest_pending_age_seconds` | Atraso do worker frente à meta de menos de 10 s |
| `webhook_delivery_latency_ms` (p50/p95 a partir de `webhook_deliveries`) e latência fim a fim `createdAt` da outbox → entrega bem-sucedida | Verificação de OT-03 |
| `webhook_delivery_failures_total` por `errorReason` e `webhook_dead_letter_count` | Saúde da entrega |
| `webhook_outbox_processing_stuck_count` | Linhas `PROCESSING` acima de um limite de tempo (P-12) |

**Tracing / correlação:**
- O `eventId` é o identificador de correlação de ponta a ponta: está na outbox, no log, no header `X-Event-Id` e no histórico.
- **Proposta:** acrescentar `correlationId` à outbox com o `X-Request-Id` da requisição que mudou o status (`requestLogger`, `src/middlewares/request-logger.middleware.ts`). Para isso, `changeStatus` precisaria receber esse valor, o que altera a assinatura interna do método.
- Não há OpenTelemetry no projeto; adotá-lo exigiria decisão própria.

## 12. Integração com o sistema existente

Os símbolos e linhas abaixo foram verificados no commit `e7f6311`.

| Arquivo | Comportamento atual | Alteração futura proposta | Contrato a preservar | Risco de regressão | Teste de comprovação |
|---|---|---|---|---|---|
| `src/modules/orders/order.service.ts` | `OrderService.changeStatus` (126–179) executa `$transaction` com validação, estoque, `order.update`, `orderStatusHistory.create` e refresh. `type TxClient = Prisma.TransactionClient` (24). `create` (50–124) grava PENDING sem `changeStatus`. | Chamar `publishWebhookEvent(tx, order, from, to)` após a linha 167 e antes do refresh (169) ([09:40] Bruno, [09:41] Bruno). Injetar a função no construtor ou importá-la: `PROPOSTA_TÉCNICA`. | `PATCH /api/v1/orders/:id/status` continua retornando `OrderWithRelations` com 200, e os erros `INVALID_STATUS_TRANSITION`/`INSUFFICIENT_STOCK` não mudam | Transação mais longa; nova causa de falha (outbox ou 64 KB se P-10 = inserção); possível timeout da transação interativa | T-OUT-01..05; regressão: os 8 casos atuais de `tests/orders.test.ts` continuam passando |
| `src/modules/orders/order.status.ts` | `transitions`, `canTransition`, `shouldDebitStock`, `shouldReplenishStock`; `PENDING` não é destino de nenhuma transição | Nenhuma alteração. O módulo de webhooks usa `OrderStatus` para validar `statuses`. | Máquina de estados inalterada | Filtro com `PENDING` nunca dispara (§6.6) | T-CRUD-04 (validação do filtro) |
| `src/middlewares/auth.middleware.ts` | `authenticate` (27–47) popula `req.user`; `requireRole(...roles)` (49–61) devolve 401/403 | Usar `authenticate` nas rotas E1–E7 e `requireRole('ADMIN')` em E7 (`DECIDIDO` [09:36] Larissa) | Formato de `req.user` e códigos `UNAUTHORIZED`/`FORBIDDEN` | Nenhum no middleware; risco de segurança herdado (I-3) | T-SEC-01, T-SEC-02 |
| `src/middlewares/validate.middleware.ts` | `validate({ body, query, params })` converte `ZodError` em `ValidationError` (`VALIDATION_ERROR`) | Usado sem alteração nos schemas do módulo; a regra https vai no service (§6.2) | Formato `details: [{ path, message }]` para todos os módulos | Alterá-lo mudaria o contrato global (não proposto) | T-CRUD-02, T-CRUD-03 |
| `src/middlewares/error.middleware.ts` | `errorMiddleware` serializa `AppError`, `ZodError` e Prisma P2002/P2025; o resto vira 500 | Nenhuma alteração ([09:29] Bruno) | `{ error: { code, message, details? } }` | P2003 (FK) cai em 500; o desenho com `onDelete: Cascade`/sem FK em `orderId` evita esse caminho | T-CRUD-06 (exclusão de customer com webhook) |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | `AppError(message, statusCode, errorCode, details)`; `ValidationError` e `NotFoundError` com códigos fixos; `BadRequestError`, `ConflictError` e `UnprocessableEntityError` aceitam código | Criar as classes `WEBHOOK_*` (§8). Local (`shared/errors` ou o módulo) em aberto (ADR-006). | Classes existentes inalteradas | Nenhum se for só adição | T-ERR-01 (cada código → status e corpo esperados) |
| `src/shared/logger/index.ts` | `createLogger`/`logger` com `redactPaths` (4–11) sem `secret` | Acrescentar caminhos de secret e assinatura (`PROPOSTA_PARA_REVISAO`) | Campos e formato atuais dos logs | Redação excessiva esconder dados úteis | T-SEC-06 (log não contém secret) |
| `prisma/schema.prisma` | Models `User`, `Customer`, `Order`, `OrderStatusHistory` etc.; UUID `Char(36)`; `provider = "mysql"` | Novos models e enum (§5); relação reversa `Customer.webhooks`; migration aditiva | Tabelas e colunas existentes inalteradas | Migration mal gerada; lock em tabelas grandes (só criação de tabelas) | Migration aplicada em banco limpo e em cópia com dados (rule 07) |
| `src/app.ts` | `buildControllers(prisma)` (26–53) monta os controllers; `buildApp` monta `/api/v1` e o `errorMiddleware` | Instanciar `WebhookRepository`, `WebhookService` e `WebhookController` e passá-los a `Controllers`; injetar o publisher em `OrderService` | `buildApp({ prisma })` continua sendo a fábrica usada pelos testes (`tests/helpers/factories.ts`) | Ordem de construção de dependências | Todos os testes atuais; T-CRUD-01 |
| `src/routes/index.ts` | Tipo `Controllers` e `buildApiRouter` montam 5 routers | Montar `buildWebhookRouter` (E1–E6) e `buildAdminWebhookRouter` (E7, `/admin/webhooks`) | Rotas existentes inalteradas | Conflito de prefixo: `/customers/:customerId/webhooks` entra no espaço de `/customers`, então a ordem de montagem precisa ser testada | T-CRUD-01, T-SEC-02 |
| `src/config/database.ts` | `createPrismaClient()` e o singleton `prisma` | O worker chama `createPrismaClient()` ([09:30] Bruno) | Singleton da API inalterado | Importar o módulo também instancia o singleton; o worker deve importar só a fábrica ou aceitar uma instância extra ociosa | T-WRK-01 |
| `src/config/env.ts` | Zod exige `DATABASE_URL` e `JWT_SECRET` e encerra o processo se faltarem | Para o worker: reaproveitar (exige `JWT_SECRET` sem usá-lo) ou criar schema próprio; nova variável se P-05 = cifrar | Variáveis atuais | Worker não sobe sem `JWT_SECRET` | T-WRK-01 |
| `src/server.ts` | `bootstrap()` com tratamento de `SIGINT`/`SIGTERM` | Novo `src/worker.ts` seguindo o mesmo padrão ([09:11] Larissa) | API inalterada | — | T-WRK-02 |
| `package.json` | Scripts `dev`, `build`, `start`, `db:*`, `test`, `lint`, `format`; sem `worker`; `engines.node >= 20` | Script `worker` (e eventualmente `worker:dev`) | Scripts existentes | `tsconfig.build.json` inclui `src/**/*.ts`, então o worker é compilado em `dist/worker.js` | Execução manual em ambiente autorizado |
| `tests/setup.ts` | `beforeEach` limpa as tabelas com `deleteMany` na ordem das FKs | Incluir as novas tabelas, antes de `customer` | Isolamento entre testes | Esquecer uma tabela causa interferência entre testes | Suíte inteira |

## 13. Dependências e compatibilidade

| Item | Situação |
|---|---|
| Bibliotecas novas | **Nenhuma.** `node:crypto` para HMAC e geração de secret; `fetch`/`AbortSignal.timeout` nativos do Node ≥ 20 (`package.json`, `engines`) |
| Prisma | `@prisma/client`/`prisma` 5.22.0 (`package.json`). Transação interativa já usada em `changeStatus`. O timeout padrão deve ser verificado na doc da 5.22 antes de ajustar. |
| Banco | MySQL 8.0 (`docker-compose.yml`, `image: mysql:8.0`). Nada específico de outro SGBD; `SKIP LOCKED` não é necessário com worker único. |
| Contratos HTTP existentes | Inalterados, exceto pela possível nova falha em `PATCH /orders/:id/status` se P-10 escolher a inserção. Isso exige avaliação de impacto (rule 09). |
| Contrato com clientes externos | Novo: payload snake_case, headers `X-*` e semântica at-least-once documentada no portal ([09:26] Marcos). Mudanças futuras no payload devem ser aditivas. Um `schema_version` no payload não foi discutido (`PROPOSTA_TÉCNICA` opcional). |
| Deploy | Novo processo (`npm run worker`) a operar junto da API. Migration aplicada antes de subir a nova versão da API. |
| Prazo e processo | Três sprints, incluindo a revisão da Sofia ([09:47] Larissa); revisão de segurança de no mínimo dois dias úteis ([09:46] Sofia); sessão de revisão do design com Bruno e Diego antes de codar ([09:50] Larissa) |

## 14. Estratégia de testes e critérios de aceite técnicos

Testes especificados para implementação futura em TDD. Nenhum foi escrito ou executado: **status NOT_RUN**.
- O runner existente é Vitest com Supertest contra MySQL real (`vitest.config.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`).
- Testes de integração exigem banco isolado de teste, porque `tests/setup.ts` apaga tabelas.

**Legenda:**
- **Nível:** U = unidade; I = integração com MySQL real; C = contrato HTTP; E = ponta a ponta (API, worker e servidor HTTP de teste local).
- **Pronto?** "Sim" = executável com as decisões atuais; "Bloqueado (P-xx)" = depende de decisão.

### 14.1 Outbox e atomicidade

| ID | Dado / Quando / Então | Nível | Pronto? |
|---|---|---|---|
| T-OUT-01 | Dado um webhook ativo do customer com `statuses` contendo `PROCESSING`; quando `PAID → PROCESSING`; então o status muda **e** existe 1 linha `PENDING` na outbox com o snapshot esperado | I | Sim |
| T-OUT-02 | Dado o mesmo cenário; quando a inserção na outbox falha (falha injetada no publisher); então o status do pedido, o histórico e o estoque permanecem como antes (rollback total) | I | Sim |
| T-OUT-03 | Dado que nenhum webhook escuta `SHIPPED`; quando `PROCESSING → SHIPPED`; então nenhuma linha é inserida | I | Sim |
| T-OUT-04 | Dada uma transição inválida (`PENDING → SHIPPED`); quando chamada; então 409 `INVALID_STATUS_TRANSITION` e nenhuma linha de outbox | I | Sim |
| T-OUT-05 | Dado `PENDING → PAID` sem estoque; quando chamada; então 422 `INSUFFICIENT_STOCK` e nenhuma linha de outbox | I | Sim |
| T-OUT-06 | Dado o pedido alterado depois do evento (por exemplo excluído quando permitido); quando o worker envia; então o payload enviado é o snapshot original | I | Sim |
| T-OUT-07 | Dados 2 webhooks interessados; quando a transição ocorre; então N linhas e mesmo `eventId` | I | Bloqueado (P-11) |
| T-OUT-08 | Dadas duas mudanças concorrentes no mesmo pedido; então registrar o comportamento observado (caracterização de P-17) | I | Sim (caracterização) |

### 14.2 Worker, entrega e idempotência

| ID | Dado / Quando / Então | Nível | Pronto? |
|---|---|---|---|
| T-WRK-01 | O worker sobe com `PrismaClient` próprio e encerra no `SIGTERM` sem perder linhas | I | Sim |
| T-WRK-02 | Dada uma linha `PENDING`; quando o tick roda e o servidor de teste responde 200; então `DELIVERED`, 1 registro em `webhook_deliveries` e headers `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`, `X-Signature` e `Content-Type` presentes | E | Sim |
| T-WRK-03 | Assinatura: o servidor de teste recalcula `HMAC-SHA256(secret, corpo bruto)` e compara com `X-Signature` | E | Sim (formato do header: proposta) |
| T-WRK-04 | Falha de HMAC: com o corpo alterado em 1 byte, a verificação do teste falha (prova que a assinatura cobre o corpo) | U | Sim |
| T-WRK-05 | Timeout: o servidor de teste não responde; após 10 s (relógio controlado), falha `WEBHOOK_DELIVERY_TIMEOUT` e reagendamento com o 1º intervalo | E | Sim |
| T-WRK-06 | O servidor responde 500 e, depois, 200; então 2 registros de entrega e `DELIVERED` | E | Sim |
| T-WRK-07 | Resposta 4xx: retry ou não | E | Bloqueado (P-20) |
| T-WRK-08 | Crash depois de o cliente receber e antes de marcar `DELIVERED`; após o reinício há reenvio com o **mesmo** `X-Event-Id` (at-least-once) | E | Bloqueado (P-12, mecânica de recuperação) |
| T-WRK-09 | Linha `PROCESSING` órfã após reinício é recuperada sem perda | I | Bloqueado (P-12) |
| T-WRK-10 | Dois eventos do mesmo pedido sem falhas são entregues na ordem de `createdAt` | E | Sim |
| T-WRK-11 | Evento 1 em retry e evento 2 do mesmo pedido: ordem observada | E | Bloqueado (P-21) |
| T-WRK-12 | Banco indisponível durante o tick: o worker registra o erro e continua no tick seguinte | I | Sim |
| T-WRK-13 | Resposta 3xx não é seguida | E | Bloqueado (decisão SSRF, Sofia) |

### 14.3 Retry, DLQ e replay

| ID | Dado / Quando / Então | Nível | Pronto? |
|---|---|---|---|
| T-RTY-01 | A configuração de backoff contém exatamente `[1m, 5m, 30m, 2h, 12h]`, nessa ordem | U | Sim (valores aprovados) |
| T-RTY-05 | Os intervalos **efetivamente consumidos** até a DLQ (4 na leitura A, 5 na leitura B) | I | **Bloqueado (I-1)** |
| T-RTY-02 | Fronteira: a última tentativa permitida falha → DLQ com payload, motivo e `failedAt` | I | **Bloqueado (I-1)** |
| T-RTY-03 | Fronteira: a penúltima tentativa falha → reagendada, não vai para a DLQ | I | **Bloqueado (I-1)** |
| T-RTY-04 | A movimentação para a DLQ é atômica: falha ao inserir a DLQ mantém a linha na outbox | I | Sim |
| T-DLQ-01 | Replay por ADMIN cria uma linha `PENDING`, marca `replayedAt`/`replayedById` e emite log com `userId` | I | Sim (identidade: ver T-DLQ-03) |
| T-DLQ-02 | Replay por OPERATOR → 403 `FORBIDDEN` | C | Sim |
| T-DLQ-03 | O evento reenviado mantém o `X-Event-Id` original | E | Bloqueado (P-08) |
| T-DLQ-04 | Segundo replay do mesmo registro → 409 `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | C | Sim (código proposto) |
| T-DLQ-05 | Registro inexistente → 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND` | C | Sim |

### 14.4 CRUD, secrets, limites e segurança

| ID | Dado / Quando / Então | Nível | Pronto? |
|---|---|---|---|
| T-CRUD-01 | E1 cria e devolve a secret; E2 lista sem a secret; E3 altera; E4 remove (204) | C | Sim (caminhos propostos; P-13) |
| T-CRUD-02 | URL `http://` → 400 `WEBHOOK_INVALID_URL` | C | **Bloqueado (I-2)** |
| T-CRUD-03 | `statuses` vazio ou com valor fora de `OrderStatus` → 400 `VALIDATION_ERROR` | C | Sim |
| T-CRUD-04 | `statuses` com `PENDING`: comportamento escolhido (§6.6) | C | Bloqueado (P-09) |
| T-CRUD-05 | Customer inexistente → 404 `WEBHOOK_CUSTOMER_NOT_FOUND` | C | Sim (código proposto) |
| T-CRUD-06 | Exclusão de customer **sem pedidos** e com webhook não gera 500. Excluir customer com pedidos já gera P2003/500 hoje, por causa do `ON DELETE RESTRICT` de `orders_customerId_fkey` (`prisma/migrations/20260519182739_init/migration.sql`). É comportamento pré-existente, fora do escopo. | I | Sim |
| T-SEC-01 | Rotação: nova secret devolvida; anterior válida até +24 h; após 24 h, descartada (relógio controlado) | I | Sim (janela). Assinatura na janela: **Bloqueado (P-06)** |
| T-SEC-02 | Rotas E1–E7 sem token → 401 | C | Sim |
| T-SEC-03 | Usuário sem vínculo com o customer tenta E1–E6 → 403 | C | **Bloqueado (G-12)** |
| T-SEC-04 | Registro público com `role: 'ADMIN'` rejeitado | C | **Bloqueado (I-3; mudança fora do escopo)** |
| T-SEC-05 | URL que resolve para IP privado é recusada | C/E | Bloqueado (SSRF, Sofia) |
| T-SEC-06 | Logs do CRUD, da rotação e do worker não contêm a secret | I | Sim, se a redação for aprovada |
| T-LIM-01 | Snapshot de 64 KB exatos é aceito; 64 KB + 1 byte gera `WEBHOOK_PAYLOAD_TOO_LARGE` na etapa decidida | I | **Bloqueado (P-10)** |
| T-LIM-02 | Snapshot típico (§6.5) fica muito abaixo de 64 KB | U | Sim |
| T-ERR-01 | Cada erro de API da §8 gera o status HTTP da matriz e o corpo `{ "error": { "code": "WEBHOOK_…", "message": … } }` via `errorMiddleware` | C | Sim para os códigos definidos; `WEBHOOK_INVALID_URL` bloqueado (I-2) e `WEBHOOK_SECRET_REQUIRED` bloqueado (P-04) |

### 14.5 Critérios de aceite técnicos

| ID | Critério verificável |
|---|---|
| AC-T-01 | T-OUT-01..05 verdes: nenhuma mudança de status confirmada sem evento quando há interessado, e nenhum evento após rollback |
| AC-T-02 | Em ambiente de teste, a latência entre o commit e a entrega bem-sucedida, com cliente respondendo em menos de 1 s, fica abaixo de 10 s (meta de [09:02] Marcos). Medida pela métrica de latência fim a fim da §11. |
| AC-T-03 | Toda requisição de saída contém os cinco headers da §6.8 e a assinatura verificável (T-WRK-02/03) |
| AC-T-04 | Reenvios (retry, recuperação e replay) preservam o `X-Event-Id` do evento (depende de P-08 e P-12) |
| AC-T-05 | A configuração de retry contém exatamente 1m/5m/30m/2h/12h. Os intervalos consumidos e o limiar de DLQ seguem a decisão de I-1 (pendente). |
| AC-T-06 | E7 só é executável por ADMIN e registra quem fez o replay; com I-3 em aberto, o critério de segurança fica **pendente da revisão da Sofia** |
| AC-T-07 | Todos os erros do módulo usam o prefixo `WEBHOOK_` e o formato do `errorMiddleware`; nenhum contrato de erro existente muda |
| AC-T-08 | A suíte atual (`tests/auth.test.ts`, `tests/orders.test.ts`) continua verde após a integração |
| AC-T-09 | Nenhuma secret aparece em logs nem em respostas fora de E1 e E5 |
| AC-T-10 | A revisão de segurança da Sofia foi concluída antes do deploy ([09:46] Sofia) |

## 15. Riscos, pendências e decisões ainda necessárias

| ID | Pendência | Origem | Impacto | Alternativas | Decisão necessária? / Responsável sugerido |
|---|---|---|---|---|---|
| P-01 (I-1) | "5 tentativas" contra 5 intervalos | [09:17] Diego, [09:17] Larissa, [09:48] Larissa; ADR-003 | Limiar da DLQ, janela comunicada ao cliente, T-RTY-02/03 | A: 5 envios (2h36min); B: 1 + 5 retries (14h36min) | **Sim**: Larissa e Diego; consultar Marcos se a janela mudar |
| P-02 (I-2) | `VALIDATION_ERROR` contra `WEBHOOK_INVALID_URL` | `validate.middleware.ts`; [09:23] Sofia, [09:28] Bruno; ADR-006 | Contrato de erro de E1/E3; T-CRUD-02 | P1 checagem no service; P2 wrapper; P3 aceitar `VALIDATION_ERROR` | **Sim**: Bruno e Larissa |
| P-03 (I-3) | Registro público aceita ADMIN | `auth.schemas.ts`, `auth.routes.ts`; ADR-006 | O replay não fica efetivamente protegido | Restringir o role no registro (fora do escopo) | **Sim**: Sofia (revisão) e Larissa (escopo) |
| P-04 | Semântica de `WEBHOOK_SECRET_REQUIRED` | [09:28] Bruno, [09:31] Marcos | Código sem uso definido | Falha interna de envio / remover / outro uso | Sim: Bruno |
| P-05 | Armazenamento da secret em repouso | ADR-004 | Exposição em caso de vazamento do banco | Cifrar (AES-256-GCM, chave em env) / em claro com acesso restrito | **Sim**: Sofia |
| P-06 | Qual secret assina durante as 24 h | ADR-004; [09:21] Sofia | O cliente pode rejeitar envios durante a migração | A: duas assinaturas no `X-Signature`; B: assinar com a anterior até expirar; C: assinar só com a nova (contraria o propósito da janela) | **Sim**: Sofia |
| P-07 | `X-Timestamp` dentro do HMAC | ADR-004/005; [09:44] Diego | Proteção contra replay no cliente | Assinar `timestamp + "." + corpo` / só o corpo (decisão atual: "corpo") | **Sim**: Sofia |
| P-08 | `event_id` após replay | ADR-003/005; [09:18] Diego, [09:25] Diego | Deduplicação pelo cliente | Preservar (recomendado) / gerar novo | **Sim**: Diego e Larissa |
| P-09 | Evento na criação do pedido | ADR-001; `OrderService.create` | Filtro `PENDING` inerte; cobertura do ciclo de vida | Emitir em `create` (mesma técnica do publisher) / não emitir e rejeitar `PENDING` no filtro | **Sim**: Marcos (produto) e Bruno |
| P-10 | Onde aplicar o limite de 64 KB | ADR-001; [09:23] Sofia, [09:24] Larissa | Inserção: pode abortar `changeStatus` (mudança de contrato). Envio: o evento vai direto à DLQ. | Na inserção / no envio | **Sim**: Larissa e Sofia |
| P-11 | Fan-out para vários endpoints | ADR-001/005 | Modelo da outbox, unicidade, T-OUT-07 | Uma linha por par evento × webhook (proposta) / uma linha por evento com tabela de entregas por webhook | **Sim**: Diego |
| P-12 | Recuperação de `PROCESSING` órfão | ADR-002; [09:08] Diego | Eventos presos após crash | Reset no startup (worker único) / lease com expiração | **Sim**: Diego |
| P-13 | `customer_id` no path ou no body | [09:32] Larissa | Caminhos E1/E2 | Path (proposta deste FDD) / body | Sim: Larissa |
| P-14 | Nome do arquivo do processador | [09:28] Bruno | Organização do módulo | `webhook.worker.ts` / `webhook.processor.ts` | Sim (baixo impacto): Bruno |
| P-15 | Papel do status `FAILED` na outbox | ADR-003; [09:08] Diego, [09:18] Diego | Unicidade `eventId`×`webhookId` no replay; limpeza | Marcar `FAILED` / remover a linha ao mover para a DLQ | Sim: Diego (decidir junto com P-08) |
| P-16 | Eventos pendentes de webhook removido ou desativado | Lacuna deste FDD | Envio a destino indesejado ou perda silenciosa | Cancelar os pendentes / entregar mesmo assim / mover para a DLQ | Sim: Marcos e Bruno |
| P-17 | Concorrência em `changeStatus` sem lock | `order.service.ts` 132; ADR-001 | Eventos duplicados ou inconsistentes (pré-existente) | `SELECT … FOR UPDATE` via SQL / update condicional por status / aceitar | Sim: Bruno (fora do escopo estrito) |
| P-18 | Campos básicos do payload além de `total_cents` | [09:43] Diego | Contrato com o cliente (adicionar depois é compatível, remover não) | Só os nomeados / incluir `subtotal_cents`, `discount_cents` | Sim: Marcos e Diego |
| P-19 | Tamanho do batch | [09:08] Diego ("batch pequeno") | Vazão e bloqueio por cliente lento | Valor fixo configurável (por exemplo 10–50) | Sim: Diego |
| P-20 | Critério de sucesso e retry de 4xx | ADR-003 | Retentar erros permanentes do cliente | Só 2xx é sucesso; 4xx com ou sem retry | **Sim**: Diego e Sofia |
| P-21 | Ordenação com retries | ADR-002/003 | O cliente pode receber `SHIPPED` antes de `PROCESSING` | Aceitar e documentar (o cliente usa `timestamp`) / segurar eventos posteriores do mesmo pedido enquanto um anterior estiver em retry | **Sim**: Diego e Marcos |
| P-22 (G-12) | Autorização por objeto entre usuário e customer | `prisma/schema.prisma`; ADR-006 | Exposição de eventos entre clientes | Vincular usuário–customer / aceite formal do risco "por enquanto" | **Sim**: Sofia |
| P-23 | SSRF | ADR-004 (não discutido na reunião) | Acesso a recursos internos a partir do worker | Bloqueio de faixas privadas, sem redirects, DNS no envio | **Sim**: Sofia |
| P-24 | Segunda rotação dentro das 24 h | Lacuna deste FDD (E5) | Mais de uma secret anterior válida | Bloquear (409) / permitir e descartar a anterior | Sim: Sofia |
| P-25 | Unidade dos "últimos 100" do histórico | [09:34] Marcos | Forma de consulta de E6 | 100 tentativas por webhook (proposta) / 100 eventos por customer | Sim (baixo impacto): Marcos |

**Riscos residuais** (probabilidade e impacto são avaliação da análise):

| Risco | Prob. | Impacto | Mitigação proposta |
|---|---|---|---|
| Cliente lento atrasa os demais (worker sequencial + 10 s) | Média | Alto (meta de 10 s) | P-19 e concorrência limitada; métrica da idade do pendente mais antigo |
| Exposição de eventos entre clientes (G-12) e replay indevido (I-3) | Média | Alto | §9, revisão da Sofia |
| Crescimento da outbox sem arquivamento nesta fase | Alta | Baixo/Médio | Índices propostos; arquivamento em fase futura ([09:08] Diego) |
| Transação de `changeStatus` mais longa | Baixa | Médio | Publisher com uma consulta indexada e inserções simples; T-OUT-* |
| Cliente sem deduplicação | Média | Médio | Documentação no portal ([09:26] Marcos) |

## 16. Referências às fontes e ADRs

- **Transcrição:** [`TRANSCRICAO.md`](../TRANSCRICAO.md). Falas citadas: [09:00]–[09:52], com participante indicado em cada uso.
- **RFC:** [`docs/RFC.md`](RFC.md).
- **ADRs:**
  - [ADR-001 Outbox transacional no MySQL](adrs/ADR-001-outbox-transacional-no-mysql.md)
  - [ADR-002 Worker separado com polling](adrs/ADR-002-worker-separado-com-polling.md)
  - [ADR-003 Retry, backoff e DLQ](adrs/ADR-003-retry-backoff-e-dlq.md)
  - [ADR-004 HMAC-SHA256 e rotação de secret](adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md)
  - [ADR-005 At-least-once com X-Event-Id](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
  - [ADR-006 Reuso da arquitetura e dos componentes](adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md)
- **Código** (commit `e7f6311`):
  - Pedidos: `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/modules/orders/order.schemas.ts`
  - Middlewares: `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`
  - Compartilhado: `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`
  - Montagem e configuração: `src/app.ts`, `src/routes/index.ts`, `src/server.ts`, `src/config/database.ts`, `src/config/env.ts`
  - Autenticação e usuários: `src/modules/auth/auth.routes.ts`, `src/modules/auth/auth.schemas.ts`, `src/modules/auth/auth.service.ts`, `src/modules/users/user.service.ts`
  - Banco, build e infraestrutura: `prisma/schema.prisma`, `package.json`, `tsconfig.build.json`, `docker-compose.yml`, `vitest.config.ts`
  - Testes: `tests/setup.ts`, `tests/helpers/factories.ts`, `tests/orders.test.ts`, `tests/auth.test.ts`
