# RFC — Sistema de Webhooks de Notificação de Pedidos

| Metadado | Valor |
|---|---|
| **Status** | **Proposed — Em revisão** |
| **Data do documento** | 2026-11-16 |
| **Autor** | [@jrmsz1](https://github.com/jrmsz1) — responsável pela elaboração e submissão no fork |
| **Revisores propostos** | Larissa (Tech Lead), Marcos (PM), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Fontes primárias** | [`TRANSCRICAO.md`](../TRANSCRICAO.md) e código existente do OMS |
| **Documentos relacionados** | ADR-001 a ADR-006, relacionados na seção 8 |
| **Escopo** | Proposta arquitetural para revisão; **não** é autorização de implementação ou de deploy |

> **Rastreabilidade:** `[hh:mm] Nome` ou uma faixa de horários indicam trechos de [`TRANSCRICAO.md`](../TRANSCRICAO.md), com os participantes identificados quando aplicável. Marcadores `I-*`/`G-*` representam achados posteriores da análise documental do código e dos ADRs; não são decisões da reunião.

## 1. TL;DR

Propõe-se incorporar **webhooks de saída** ao Order Management System (OMS) para notificar clientes B2B quando o status de seus pedidos mudar. A mudança do pedido e a persistência do evento serão **atômicas em MySQL**, via **Transactional Outbox**; um **worker Node.js separado** fará polling a cada 2 segundos e enviará notificações HTTP assíncronas. Cada endpoint usará **HTTPS** e **HMAC-SHA256 com secret própria**, com rotação e convivência de 24 horas. A entrega será **at-least-once**, identificada por `X-Event-Id`, com retries, backoff, **DLQ persistente** e replay administrativo auditável. Prioriza componentes existentes. Fontes: Outbox `[09:06] Diego` e `[09:08] Larissa`; polling e processo separado `[09:09] Diego`, `[09:10] Larissa`, `[09:11–09:12] Diego/Larissa`; retry/DLQ `[09:17–09:19] Diego/Larissa`; HMAC e rotação `[09:20–09:22] Sofia`; at-least-once `[09:24–09:26] Diego/Larissa`; reuso `[09:27] Bruno`, `[09:30] Larissa`; HTTPS `[09:23] Sofia`; auditoria do replay `[09:36] Sofia/Larissa`; resumo `[09:48] Larissa`.


## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo solicitaram notificações quando o estado de seus pedidos mudar. Atualmente, precisam consultar repetidamente a API de pedidos; consideram aceitável uma entrega em **menos de 10 segundos**. O fluxo é exclusivamente **outbound**. Fontes: `[09:00–09:02] Marcos`.

O OMS existente usa **Node.js/TypeScript, MySQL e Prisma**. O método `changeStatus` valida transições e, em uma transação, atualiza estoque quando necessário, pedido e histórico. Ainda não há notificações externas. Fontes: **CÓDIGO** `package.json` (stack), `prisma/schema.prisma` (`provider = "mysql"`) e `src/modules/orders/order.service.ts` (`changeStatus`).

Um envio HTTP síncrono poderia comprometer a transação de pedidos (`[09:03–09:06] Bruno/Diego`).

## 3. Proposta técnica de alto nível

### 3.1 Mudança de estado e persistência atômica

A mesma transação MySQL de `changeStatus` registrará na Outbox os eventos relevantes aos filtros cadastrados. **Falha na escrita da Outbox implica rollback** da mudança do pedido. O payload será um *snapshot* do estado no momento da alteração. Fontes: atomicidade `[09:06] Diego`, `[09:08] Larissa`, `[09:40–09:41] Bruno/Diego`; filtro na inserção `[09:33] Marcos`, `[09:34] Bruno/Diego`; snapshot `[09:52] Larissa/Diego/Bruno`; **CÓDIGO** `src/modules/orders/order.service.ts`.

### 3.2 Processamento e entrega

Um processo **separado da API** consulta a Outbox a cada **2 segundos**, usando o mesmo MySQL e seu próprio PrismaClient. Inicialmente, o worker é único; a reunião reconheceu limites de ordenação e **adiou** a evolução para múltiplos workers. Fontes: polling `[09:09] Diego`, `[09:10] Larissa`; processo independente `[09:11] Diego`, `[09:12] Larissa`; ordenação e escala `[09:12–09:13] Diego/Larissa`; PrismaClient separado `[09:30] Bruno`.

A equipe considerou o polling de **2 segundos** adequado à meta de entrega em menos de **10 segundos** (`[09:02] Marcos`, `[09:09] Diego`, `[09:10] Marcos/Larissa`). **Análise — PROPOSTA_PARA_REVISAO:** o processamento sequencial em worker único, combinado com timeout HTTP de 10 segundos, pode atrasar eventos de outros clientes; avaliar o risco no desenho detalhado (ADR-002, Pendências).

### 3.3 Falhas e recuperação

Timeout HTTP de **10 segundos**; política registrada: **5 tentativas** e intervalos **1m / 5m / 30m / 2h / 12h**. O esgotamento leva à **DLQ persistida em tabela separada**. O replay manual exige `ADMIN` via `requireRole` e registro de auditoria. Fontes: política de tentativas/backoff `[09:15–09:17] Diego/Larissa`; DLQ e replay `[09:18–09:19] Diego/Larissa`; autorização e auditoria `[09:36] Sofia/Larissa`; timeout `[09:42] Diego/Sofia`.

**Ambiguidade:** cinco envios totais admitem quatro esperas, enquanto os cinco intervalos indicados admitem cinco retries após o envio inicial. **NEEDS_DECISION** antes de detalhar o envio à DLQ.

### 3.4 Autenticidade, entrega e contratos

Cada endpoint terá secret própria, gerada pela plataforma, e **HMAC-SHA256 sobre o corpo**; a secret antiga permanecerá válida por **24 horas** na rotação. HTTPS é obrigatório; payloads acima de **64 KB** devem gerar erro, sem truncamento. **Permanece pendente** definir em qual etapa o limite será aplicado (ADR-001; §6). Fontes: HMAC, secret e rotação `[09:20–09:22] Sofia`; HTTPS `[09:23] Sofia`; limite de 64 KB `[09:24] Diego/Larissa`; geração da secret pela plataforma `[09:31] Marcos`.

A entrega é **at-least-once**, com deduplicação pelo cliente via `X-Event-Id`; não se promete *exactly-once*. Fontes: `[09:24–09:26] Diego/Sofia/Larissa`. Payload e headers ficam para o FDD (discussão `[09:43–09:45]`, Diego, Bruno e Sofia).

### 3.5 Integração ao OMS

A feature seguirá `src/modules/webhooks`, reutilizando **Prisma, Zod, AppError, middleware de erro, Pino e JWT**, com códigos de erro do módulo prefixados por `WEBHOOK_`. O CRUD será autenticado; `customer_id` não vem automaticamente do JWT atual. **Análise (G-12; §6):** o isolamento entre clientes exige avaliação específica. Fontes: estrutura modular `[09:27] Bruno`; reuso `[09:29–09:30] Bruno/Larissa`; `customer_id` e JWT `[09:32] Bruno/Marcos/Larissa`; CRUD autenticado `[09:36–09:37] Marcos/Sofia`; **CÓDIGO** `src/modules/orders/order.service.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`.

## 4. Alternativas consideradas e rejeitadas

| Alternativa discutida | Motivo do descarte / trade-off | Fonte |
|---|---|---|
| **Notificação síncrona dentro da transação de pedidos** | A lentidão ou indisponibilidade do destinatário afetaria a alteração do pedido e sua transação. | `[09:03–09:06] Bruno, Larissa e Diego` |
| **Redis Streams ou infraestrutura de fila adicional** | Acrescentaria infraestrutura e operação para um time pequeno; optou-se por Outbox no MySQL existente. | `[09:07–09:08] Larissa e Diego` |
| **Trigger MySQL como mecanismo de sinalização do worker** | A alternativa não oferece, por si só, notificação nativa de processos externos e exigiria mecanismos adicionais. | `[09:09–09:10] Bruno e Diego` |
| **Exactly-once ponta a ponta** | Exigiria coordenação entre emissor e destinatário; optou-se por at-least-once e deduplicação pelo cliente. | `[09:25–09:26] Diego e Larissa` |


## 5. Questões abertas e adiamentos registrados na reunião

1. **Rate limiting de saída:** Diego levantou o risco de rajadas de chamadas, e Larissa decidiu **observar e decidir depois**, sem aprovar controle nesta fase (`[09:38–09:39]`).
2. **Endurecimento futuro do CRUD:** Sofia aceitou, *por enquanto*, acesso de qualquer role autenticada ao CRUD, com possível restrição futura (`[09:36–09:37]`).
3. **Localização de `customer_id` no contrato:** Larissa indicou body **ou** path, sem fechar uma das opções (`[09:32]`).

**Fora do escopo:** e-mail (`[09:37]`), dashboard (`[09:40]`), webhooks de entrada (`[09:02]`), arquivamento da Outbox (`[09:08]`) e múltiplos workers (`[09:13]`).

## 6. Pendências identificadas posteriormente na análise

Os pontos abaixo foram identificados em **análise documental posterior do código e dos ADRs**; não são questões levantadas na reunião nem decisões confirmadas:

| Referência | Observação | Consequência para o desenho |
|---|---|---|
| **I-1 — Contagem de retries** | Cinco tentativas e cinco intervalos não definem sem ambiguidade o instante de envio à DLQ. | **NEEDS_DECISION** por Larissa/Diego antes de fechar a máquina de estados e os testes do FDD. |
| **I-2 — Contrato de erro** | O middleware de validação atual transforma falhas Zod em `VALIDATION_ERROR`; isso não produz automaticamente `WEBHOOK_INVALID_URL`. | O FDD precisará propor a adaptação local do módulo ou registrar a escolha pendente, sem alegar compatibilidade já existente. |
| **I-3 — Autoconcessão de ADMIN** | A leitura estática do código identificou registro público aceitando `role: ADMIN`, propagado ao JWT. | `requireRole('ADMIN')` isoladamente não é uma fronteira confiável para o replay. **Risco de segurança preexistente; requer revisão antes do deploy.** |
| **G-12 — Autorização entre clientes** | A análise do modelo atual não identificou vínculo confiável de usuário com customer. | O CRUD de webhooks pode expor notificações de outro cliente se a autorização por objeto não for resolvida. **Risco elevado de segurança a submeter à revisão da Sofia; mitigação ou aceite formal ainda não definidos.** |


**Outras lacunas identificadas na análise documental (não discutidas como decisões na reunião):**

| Referência | Questão a esclarecer antes do detalhamento da implementação |
|---|---|
| **ADR-004 — Rotação de secret** | Qual secret é usada para **assinar novos envios** durante as 24 horas em que a anterior continua válida? A janela de aceitação e a chave usada para novos envios são questões distintas. |
| **ADR-004 — Assinatura de timestamp** | O header `X-Timestamp` integra o conteúdo autenticado pelo HMAC ou apenas acompanha o corpo? A reunião confirma HMAC sobre o corpo, mas não fecha essa extensão. |
| **ADR-005 / ADR-003 — Identidade no replay** | O reenvio manual da DLQ preserva o `event_id` original ou cria outro? Isso afeta a deduplicação baseada em `X-Event-Id`. |
| **ADR-001 — Criação de pedido** | `OrderService.create` também produzirá evento ou a Outbox será alimentada somente pelas mudanças de status via `changeStatus`? |
| **ADR-001 — Limite de 64 KB** | O tamanho é verificado **antes de inserir** o snapshot na Outbox ou **antes de enviar** a notificação? Ambas as interpretações exigem tratamento explícito da falha. |
| **ADR-002 / ADR-003 — Ordenação em retries** | A nova tentativa de um evento pode permitir que notificações posteriores do mesmo pedido cheguem antes? Qual é o limite efetivo da garantia de ordenação acordada? |

**Riscos técnicos identificados pela análise, não decisões da reunião:** SSRF em URLs HTTPS, critérios de sucesso da entrega, fan-out por cliente e recuperação de registros presos em processamento. Esses itens são inferências técnicas para revisão; não possuem aprovação ou fonte direta como decisões na transcrição.

## 7. Impacto e riscos

**Impacto:** tabelas de Outbox/DLQ no MySQL, módulo de webhooks, worker independente, integração com `changeStatus`, API de configuração/consulta e histórico de entregas, sem infraestrutura de mensageria adicional. Fontes: Outbox `[09:06–09:08] Diego/Larissa`; worker `[09:09–09:12] Diego/Larissa`; DLQ `[09:18–09:19] Diego/Larissa`; módulo `[09:27–09:30] Bruno/Larissa`; endpoints `[09:31–09:35] Marcos/Bruno`; histórico `[09:34] Marcos`; integração transacional `[09:40–09:41] Bruno/Diego`.

**Riscos:** as pendências I-1, I-2, I-3, G-12 e as lacunas da §6 demandam revisão técnica e de segurança; a identificação de um risco não equivale a uma nova decisão aprovada pela equipe.

**Dependências:** resolver I-1 e I-2; revisar I-3/G-12; reservar **pelo menos dois dias úteis** para a revisão de Sofia antes do deploy. A reunião estimou **três sprints**. Fontes: `[09:46–09:47] Larissa/Sofia`.

**Próximos passos:** a própria reunião prevê uma sessão de revisão do design com Bruno, Diego e Larissa antes de iniciar a codificação (`[09:50] Larissa`). **Propostas decorrentes da análise:** Larissa/Diego esclarecem I-1; Bruno valida integração e contratos; Sofia revisa segurança; Marcos valida escopo. O **FDD** detalhará fluxos, endpoints, erros e testes. **Status Proposed** até revisão efetiva.

## 8. Decisões arquiteturais relacionadas


1. [ADR-001 — Outbox transacional no MySQL](adrs/ADR-001-outbox-transacional-no-mysql.md)
2. [ADR-002 — Worker separado com polling](adrs/ADR-002-worker-separado-com-polling.md)
3. [ADR-003 — Retry, backoff e DLQ](adrs/ADR-003-retry-backoff-e-dlq.md)
4. [ADR-004 — HMAC-SHA256 e rotação de secret](adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md)
5. [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
6. [ADR-006 — Reuso da arquitetura e dos componentes](adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md)
