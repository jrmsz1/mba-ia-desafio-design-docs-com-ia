# ADR-006 — Reuso da arquitetura modular e dos componentes existentes do OMS

| Campo | Valor |
| --- | --- |
| Identificador | ADR-006 |
| Status | **Accepted** |
| Data da decisão | Reunião técnica de "quinta-feira, 09:00" (`TRANSCRICAO.md`, cabeçalho). A data de calendário não foi registrada. |
| Data de registro | 2026-11-16 |
| Decisores | Larissa (Tech Lead, fechamento), Bruno (Eng. Pleno, proponente), Diego (Eng. Sênior), Sofia (Eng. de Segurança, regras de acesso), Marcos (PM) |
| Base de código | commit `e7f6311` |
| Relacionados | [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-003](ADR-003-retry-backoff-e-dlq.md), [ADR-004](ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md) |

**Legenda:** `DECISÃO_CONFIRMADA`, `DETALHE_ACORDADO`, `PROPOSTA_TÉCNICA`, `QUESTÃO_ABERTA`, conforme definidos no [ADR-001](ADR-001-outbox-transacional-no-mysql.md). `PROPOSTA_PARA_REVISAO` = mitigação de segurança sugerida pela análise e sujeita à revisão da Sofia; não é decisão da reunião.

---

## Status

**Accepted.** Fechado em [09:30] Larissa ("Decisão: reuso máximo do que já existe") e confirmado no resumo de [09:48] Larissa. As fragilidades herdadas (I-2, I-3 e G-12) ficam como pendências e riscos.

## Contexto

O OMS (commit `e7f6311`) é um **monólito modular em camadas**. Padrões observados no código:

| Padrão existente | Onde |
| --- | --- |
| Módulos por domínio com `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts` | `src/modules/{auth,users,customers,products,orders}/` |
| Composição manual de dependências e registro de rotas sob `/api/v1` | `src/app.ts` (`buildControllers`, `buildApp`), `src/routes/index.ts` (`Controllers`, `buildApiRouter`) |
| Erro base com `statusCode`, `errorCode` e `details` | `src/shared/errors/app-error.ts` (`AppError`) |
| Subclasses HTTP e de domínio com códigos em MAIÚSCULAS (`INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK`) | `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError`, `InsufficientStockError`, entre outras) |
| Tratamento centralizado de `AppError`, `ZodError` e Prisma P2002/P2025, com resposta `{ error: { code, message, details? } }` | `src/middlewares/error.middleware.ts` (`errorMiddleware`) |
| Validação Zod por rota, convertendo `ZodError` em `ValidationError` | `src/middlewares/validate.middleware.ts` (`validate`) |
| Autenticação JWT e autorização por papel | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) |
| Logger Pino com redação de campos sensíveis; log de requisição com `X-Request-Id` | `src/shared/logger/index.ts` (`logger`), `src/middlewares/request-logger.middleware.ts` (`requestLogger`) |

A reunião precisava definir se a feature de webhooks seguiria esses padrões ou introduziria estrutura própria ([09:27] Larissa).

## Decisão

1. **[DECISÃO_CONFIRMADA]** **Reuso máximo do que já existe**: `AppError`, Pino, o error middleware, o padrão de módulos, o padrão de schemas Zod e o padrão de códigos de erro. "Webhook fica como módulo igual aos outros."
   - Fontes: [09:30] Larissa (decisão), [09:48] Larissa.
2. **[DECISÃO_CONFIRMADA]** Um módulo **`src/modules/webhooks`** com controller, service, repository, routes e schemas.
   - Fontes: [09:27] Bruno, [09:28] Diego ("Faz"), [09:30] Larissa, [09:48] Larissa ("módulo em src/modules/webhooks").
3. **[DECISÃO_CONFIRMADA]** **Prefixo `WEBHOOK_`** em todos os códigos de erro do módulo.
   - Fontes: [09:28] Bruno, [09:29] Larissa ("Prefixo WEBHOOK_ pra tudo do módulo"), [09:48] Larissa.
   - **[DETALHE_ACORDADO]** Exemplos citados: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno). O catálogo completo ficou em aberto ("etc.").
4. **[DECISÃO_CONFIRMADA]** **Nenhum logger novo**: o Pino já usado no projeto atende ([09:29] Bruno, "Não vamos botar nada novo"; [09:30] Larissa).
5. **[DETALHE_ACORDADO]** O **error middleware centralizado** "vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno).
   - Confirmado no código para subclasses de `AppError`: `errorMiddleware`, ramo `err instanceof AppError`.
6. **[DECISÃO_CONFIRMADA]** O replay da DLQ **reaproveita `requireRole`** com `ADMIN` ([09:36] Larissa). O detalhamento está em [ADR-003](ADR-003-retry-backoff-e-dlq.md).
7. **[DECISÃO_CONFIRMADA]** Os endpoints de configuração de webhook usam **autenticação normal** (`authenticate`). "Por enquanto", **qualquer role autenticada** pode usá-los.
   - Fontes: [09:36] Marcos (pergunta), [09:37] Sofia ("Por enquanto sim. Mais pra frente a gente pode endurecer"), [09:48] Larissa ("Endpoints CRUD de configuração autenticados normal").
8. **[DECISÃO_CONFIRMADA]** O `customer_id` **não vem do JWT**, porque o JWT é do usuário operador. Ele é informado no body ou no path.
   - Fontes: [09:32] Bruno, [09:32] Marcos, [09:32] Larissa, [09:33] Bruno ("Anotado.").
   - **[QUESTÃO_ABERTA]** Body ou path não foi escolhido ([09:32] Larissa).
9. **[DETALHE_ACORDADO]** A lógica de processamento do worker fica **dentro do módulo**. O nome do arquivo está em aberto (ver [ADR-002](ADR-002-worker-separado-com-polling.md)).
   - Fonte: [09:28] Bruno.

### Reuso direto e necessidade de adaptação (análise do código)

| Componente | Reuso | Adaptação necessária | Classificação |
| --- | --- | --- | --- |
| `AppError` | Base de todas as classes `WEBHOOK_*` | Nenhuma | DERIVADO_DO_CODIGO |
| `errorMiddleware` | Serializa qualquer `AppError` | Nenhuma | DERIVADO_DO_CODIGO |
| `NotFoundError` | — | O código é fixo em `'NOT_FOUND'` (`http-errors.ts`). `WEBHOOK_NOT_FOUND` exige uma subclasse própria ou o uso direto de `AppError` com status 404. | DERIVADO_DO_CODIGO |
| `validate` + Zod | Validação de formato de body, params e query | O código é fixo em `VALIDATION_ERROR` (ver I-2) | DERIVADO_DO_CODIGO |
| `requireRole` | Replay da DLQ com `ADMIN` | Nenhuma no middleware; ver riscos I-3 | DERIVADO_DO_CODIGO |
| `logger` (Pino) | Logs do módulo e do worker | `redactPaths` não cobre `secret` (ver [ADR-004](ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md)) | DERIVADO_DO_CODIGO |
| `buildControllers` / `buildApiRouter` / `Controllers` | Registro do novo controller e das rotas | Inclusão das entradas do módulo; detalhe no FDD | PROPOSTA_TÉCNICA |

## Pendência I-2 — `WEBHOOK_INVALID_URL` e o fluxo de validação existente

**Status: QUESTÃO_ABERTA. Encontrada na análise documental.**

- **Evidência.** Se a regra "URL do webhook tem que ser https" ([09:23] Sofia) for implementada só no schema Zod e aplicada com `validate({ body })`, o fluxo é este:
  1. `schemas.body.parse` lança `ZodError`;
  2. `validate` converte em `new ValidationError('Validation failed', details)` (`src/middlewares/validate.middleware.ts`);
  3. `ValidationError` define `errorCode = 'VALIDATION_ERROR'` com status 400 (`src/shared/errors/http-errors.ts`);
  4. o `errorMiddleware` responde `{"error":{"code":"VALIDATION_ERROR",…}}`.
- **Consequência:** o código `WEBHOOK_INVALID_URL` ([09:28] Bruno) **nunca seria emitido** por esse caminho.
- **Opções para o FDD** (`PROPOSTA_TÉCNICA`, sem decisão):
  - **P1:** o Zod valida o formato e uma checagem no service lança uma subclasse de `AppError` com `WEBHOOK_INVALID_URL`. Não altera componentes compartilhados.
  - **P2:** um wrapper de validação próprio do módulo traduz o erro do campo `url`.
  - **P3:** aceitar `VALIDATION_ERROR` com `details` e não usar `WEBHOOK_INVALID_URL`. Isso diverge do código citado em [09:28] Bruno.
- **[PROPOSTA_TÉCNICA] Não recomendado:** alterar o `validate` compartilhado, porque mudaria o contrato de erro de todos os módulos existentes.
- **Também em aberto:**
  - status 400 ou 422;
  - onde ficam as classes de erro (`src/shared/errors/` ou dentro do módulo);
  - a semântica de `WEBHOOK_SECRET_REQUIRED`, já que a secret é gerada pela plataforma ([09:31] Marcos).
- **Quem decide:** Bruno, autor dos códigos ([09:28]), e Larissa, que fechou o padrão ([09:29], [09:30]).

## Riscos de segurança pré-existentes (I-3 e G-12)

**Classificação:** fatos do código (`DERIVADO_DO_CODIGO`) verificados por leitura estática no commit `e7f6311`; **não foram exercitados em execução**. Todas as mitigações são `PROPOSTA_PARA_REVISAO` e dependem da **revisão de segurança da Sofia** ([09:46] Sofia). Nenhuma delas foi decidida na reunião, e este ADR não altera código.

### I-3 — O cadastro público aceita a role `ADMIN`

| Etapa | Evidência |
| --- | --- |
| A rota de registro é pública, sem `authenticate` | `src/modules/auth/auth.routes.ts` (`buildAuthRouter`: `router.post('/register', validate({ body: registerSchema }), controller.register)`), montada em `/api/v1/auth` (`src/routes/index.ts`) |
| O schema aceita `ADMIN` enviado pelo cliente | `src/modules/auth/auth.schemas.ts` (`registerSchema.role: z.enum(['ADMIN', 'OPERATOR']).default('OPERATOR')`) |
| O role é repassado e gravado sem filtro | `AuthController.register` → `AuthService.register` → `UserService.createUser` (`role: input.role`) → `UserRepository.create` |
| O JWT carrega o role gravado | `AuthService.login` → `signToken(user.id, user.email, user.role)` (`src/modules/auth/auth.service.ts`) |
| A autorização confia no role do token | `authenticate` (`req.user.role = payload.role`) e `requireRole` (`src/middlewares/auth.middleware.ts`) |
| Não há teste negativo | `tests/auth.test.ts` registra apenas com `role: 'OPERATOR'` |

**Impacto no replay da DLQ** (inferência): enquanto o cadastro aceitar `ADMIN`, a decisão "Decidido, role ADMIN obrigatório no replay" ([09:36] Larissa) **não restringe efetivamente** quem pode reprocessar eventos.
- Um ator não autorizado poderia reenfileirar eventos e provocar reenvios de dados de pedidos aos endpoints cadastrados.
- O registro de auditoria ([09:36] Sofia) identificaria apenas um usuário criado pelo próprio ator.

### G-12 — Não há autorização por objeto entre usuário e customer

- **Fato do código:** o model `User` não tem relação com `Customer` (`prisma/schema.prisma`, models `User` e `Customer`).
- **Fato da reunião:** usuários "representam o cliente" ([09:32] Marcos), mas o vínculo não existe no schema.
- **Fato da reunião:** o CRUD de webhooks aceita qualquer role autenticada (item 7 da Decisão).
- **Risco de exposição entre clientes** (inferência):
  - qualquer usuário autenticado poderia cadastrar, alterar ou listar webhooks de **qualquer** customer;
  - poderia, por exemplo, apontar uma URL `https` própria para receber eventos de pedidos de outra empresa, com `order_number`, `customer_id` e `total_cents` ([09:43] Diego);
  - a assinatura HMAC ([ADR-004](ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md)) não impede isso, porque autentica a origem e não o destinatário.
- Como o registro é público (`auth.routes.ts`, rota `/register` sem `authenticate`), um ator externo obtém acesso autenticado como `OPERATOR` sem credenciais prévias. Isso já basta para o cenário acima, porque o CRUD aceita qualquer role. Com o I-3, obtém também `ADMIN`, o que estende o alcance ao replay.

### Mitigações — PROPOSTA_PARA_REVISAO (não decididas)

1. Impedir que o cadastro público defina `role: 'ADMIN'`. É mudança fora do escopo da feature de webhooks e exige autorização própria e teste de regressão.
2. Antes do deploy da feature, revisar usuários `ADMIN` existentes e tokens emitidos.
3. Definir autorização por objeto entre usuário e customer para o CRUD de webhooks, ou registrar formalmente o risco aceito enquanto valer o "por enquanto" de [09:37] Sofia. Isso se relaciona ao endurecimento adiado.
4. Testes negativos: replay com `OPERATOR` retorna 403; cadastro com `ADMIN` é rejeitado, se a mitigação 1 for aprovada.
5. Redação de secrets nos logs (ver [ADR-004](ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md)).

## Alternativas Consideradas

| Alternativa | Quem levantou | Motivo do descarte |
| --- | --- | --- |
| Introduzir **componentes novos de infraestrutura**, como outro logger | Implícita na fala de [09:29] Bruno | "O logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo" ([09:29] Bruno), seguido da decisão de reuso ([09:30] Larissa). |
| Usar **apenas os códigos genéricos existentes** (`NOT_FOUND`, `VALIDATION_ERROR`, `CONFLICT`) | Alternativa plausível; não foi proposta na reunião | Preterida pela decisão de prefixar com `WEBHOOK_` todos os códigos do módulo ([09:29] Larissa). |
| Derivar o `customer_id` **do JWT** | [09:31] Marcos | O JWT é do usuário operador, não do cliente ([09:32] Bruno, [09:32] Larissa). |

## Consequências

**Positivas**
- O módulo novo segue a mesma estrutura dos existentes, o que reduz o custo de leitura e manutenção.
- Nenhuma dependência nova para logging e tratamento de erros.
- Formato de erro uniforme para os clientes da API (`{ error: { code, message, details? } }`).
- Autorização por papel com um middleware já existente e conhecido.

**Negativas**
- O módulo **herda as limitações** dos componentes reaproveitados:
  - códigos fixos em `ValidationError` e `NotFoundError` (I-2);
  - autorização por papel sem escopo de objeto (G-12);
  - cadastro que aceita `ADMIN` (I-3).
- "Sem precisar mudar nada" ([09:29] Bruno) vale para o `errorMiddleware`, mas não para todos os componentes. O `redactPaths` do logger precisará ser revisto para secrets ([ADR-004](ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md)).

## Trade-offs explícitos

- **Consistência com a base existente em troca de herdar suas fragilidades de validação e controle de acesso.** As fragilidades ficam registradas como pendências, sem corrigir o legado nesta decisão.
- **CRUD liberado a qualquer role autenticada (simplicidade imediata) em troca de risco de acesso entre customers.** Aceito "por enquanto" ([09:37] Sofia) e sujeito à revisão de segurança.

## Pendências e riscos

| Tipo | Item | Fonte |
| --- | --- | --- |
| QUESTÃO_ABERTA | **I-2**, emissão de `WEBHOOK_INVALID_URL` | [09:23] Sofia, [09:28] Bruno; código |
| QUESTÃO_ABERTA | Catálogo completo de códigos `WEBHOOK_*` ("etc.") | [09:28] Bruno |
| QUESTÃO_ABERTA | `customer_id` no body ou no path | [09:32] Larissa |
| ADIADA | Endurecer as roles do CRUD ("Mais pra frente") | [09:37] Sofia |
| Risco pré-existente | **I-3**, cadastro público com `ADMIN` | código |
| Risco pré-existente | **G-12**, ausência de vínculo usuário–customer e exposição de eventos entre clientes | código + [09:32] Marcos + [09:37] Sofia |
| Dependência | Revisão de segurança da Sofia antes do deploy | [09:46] Sofia |

## Referências às fontes

- `TRANSCRICAO.md`: [09:23] Sofia, [09:27] Larissa, [09:27] Bruno, [09:28] Diego, [09:28] Bruno, [09:29] Larissa, [09:29] Bruno, [09:30] Larissa, [09:31] Marcos, [09:32] Bruno, [09:32] Marcos, [09:32] Larissa, [09:33] Bruno, [09:36] Marcos, [09:36] Larissa, [09:36] Sofia, [09:37] Sofia, [09:43] Diego, [09:46] Sofia, [09:48] Larissa.
- Código:
  - `src/app.ts`
  - `src/routes/index.ts`
  - `src/shared/errors/app-error.ts`
  - `src/shared/errors/http-errors.ts`
  - `src/middlewares/error.middleware.ts`
  - `src/middlewares/validate.middleware.ts`
  - `src/middlewares/auth.middleware.ts`
  - `src/middlewares/request-logger.middleware.ts`
  - `src/shared/logger/index.ts`
  - `src/modules/auth/auth.routes.ts`
  - `src/modules/auth/auth.schemas.ts`
  - `src/modules/auth/auth.controller.ts`
  - `src/modules/auth/auth.service.ts`
  - `src/modules/users/user.service.ts`
  - `src/modules/users/user.repository.ts`
  - `prisma/schema.prisma`
  - `tests/auth.test.ts`
