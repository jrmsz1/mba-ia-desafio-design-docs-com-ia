# Tracker de Rastreabilidade — Webhooks de Notificação de Pedidos

| Metadado | Valor |
|---|---|
| **Data** | 2026-11-16 |
| **Autor** | [@jrmsz1](https://github.com/jrmsz1) |
| **Base de código** | commit `e7f6311` |
| **Documentos rastreados** | [PRD](PRD.md), [RFC](RFC.md), [FDD](FDD.md), [ADR-001 a ADR-006](adrs/README.md) |
| **Fontes primárias aceitas** | `TRANSCRICAO` ([`TRANSCRICAO.md`](../TRANSCRICAO.md)) e `CODIGO` (arquivos do repositório) |

## 1. Método

1. **Inventário antes da matriz.** Cada documento foi percorrido e os itens identificáveis foram listados. Contam como itens identificáveis:
   - requisitos, objetivos e metas;
   - decisões, alternativas, exclusões e adiamentos;
   - restrições e trade-offs;
   - contratos, erros, integrações, regras de segurança e estados;
   - pendências, riscos, critérios de aceitação e testes especificados;
   - propostas técnicas.
2. **Regras de contagem** (para não inflar nem esconder itens):
   - **Uma afirmação, uma linha.** Explicações repetidas da mesma afirmação **dentro do mesmo documento** contam uma vez, no local principal. Exemplos: polling de 2 s repetido no FDD (§2, §6.7 e §10); a ressalva de I-3 repetida no ADR-003 e no ADR-004 como ponteiro para o ADR-006.
   - O mesmo fato em **documentos diferentes** conta em cada documento, porque cada documento é auditado por si.
   - **Não são contados:**
     - metadados (autor, data, revisores);
     - listas de links e referências;
     - legendas;
     - as seções de navegação: PRD §13 (exceto a proposta nova de validação com pilotos) e §14, e FDD §0 e §16;
     - a tabela "Pendências de decisão que afetam o produto" do PRD §11, que só aponta para riscos e pendências do FDD já contados;
     - as consequências e trade-offs narrativos dos ADRs, que explicam decisões já contadas.
3. **Validação das fontes:**
   - **`TRANSCRICAO`:** a localização tem exatamente um par `[hh:mm] Nome`, e o script confirma que esse participante fala naquele minuto. Quando a sustentação usa mais de uma fala, a principal está em *Localização* e as complementares aparecem no resumo.
   - **`CODIGO`:** caminho real do commit `e7f6311`, com símbolo entre parênteses; o script confirma que o arquivo existe. Módulos futuros (`src/worker.ts`, `src/modules/webhooks/*`) **nunca** aparecem como fonte `CODIGO`.
4. **Propostas e análises:**
   - O prefixo `[PROPOSTA]`, `[PROPOSTA_PARA_REVISAO]` ou `[ANÁLISE]` no resumo indica que o item **não** é decisão da reunião.
   - A localização aponta a evidência primária que motivou a proposta.
   - Propostas sem evidência primária ficam **fora da matriz**, na §4, e contam como não rastreadas.
5. **Critério de "evidência motivadora":** uma proposta ou análise só entra na matriz quando uma fala ou um fato de código **coloca o problema** que ela responde. Exemplo: o fan-out responde ao `X-Webhook-Id` pedido em [09:44] Sofia "pra cliente que tem vários". Propostas sem esse vínculo (SSRF, indicadores, validação com pilotos, ordem de deploy) vão para a §4.
6. **Tipos usados:**
   - Nos ADRs, no FDD e no PRD, o campo *Tipo* preserva a classificação do documento de origem: "Decisão" para DECIDIDO e "Detalhe acordado" para ACORDADO.
   - O **RFC não rotula seus itens**; nas linhas do RFC, o Tipo foi inferido a partir da transcrição e dos ADRs correspondentes.

## 2. Resumo quantitativo

| Métrica | Numerador / Denominador | Resultado | Exigido |
|---|---|---|---|
| Cobertura (itens com rastreabilidade válida / itens identificáveis) | 484 / 499 | **97.0%** | ≥ 80% |
| Linhas com Fonte = `TRANSCRICAO` válida / total de linhas da matriz | 387 / 484 | **80.0%** | ≥ 70% |
| Linhas com Fonte = `CODIGO` e caminho real | 97 | **97** | ≥ 5 |

**Inventário por documento**

| Documento | Itens identificáveis | Rastreados | Sem rastreabilidade válida | Cobertura |
|---|---|---|---|---|
| PRD | 90 | 88 | 2 | 97.8% |
| RFC | 59 | 57 | 2 | 96.6% |
| ADRs | 117 | 116 | 1 | 99.1% |
| &nbsp;&nbsp;↳ ADR-001 | 19 | 19 | 0 | 100.0% |
| &nbsp;&nbsp;↳ ADR-002 | 17 | 17 | 0 | 100.0% |
| &nbsp;&nbsp;↳ ADR-003 | 17 | 17 | 0 | 100.0% |
| &nbsp;&nbsp;↳ ADR-004 | 17 | 16 | 1 | 94.1% |
| &nbsp;&nbsp;↳ ADR-005 | 10 | 10 | 0 | 100.0% |
| &nbsp;&nbsp;↳ ADR-006 | 37 | 37 | 0 | 100.0% |
| FDD | 233 | 223 | 10 | 95.7% |
| **Total** | **499** | **484** | **15** | **97.0%** |

## 3. Matriz de rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| -- | --------- | ---- | ----------------- | ----- | ----------- |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Atlas Comercial, MaxDistribuição e Nova Cargo pediram notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | `docs/PRD.md` | Problema | Clientes fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | `docs/PRD.md` | Risco de negócio | Atlas pode migrar para concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | `docs/PRD.md` | Fato do código | OMS sem notificação externa: o app monta apenas auth, users, customers, products e orders | CODIGO | `src/app.ts` (buildControllers) |
| PRD-CTX-05 | `docs/PRD.md` | Restrição | Estimativa de três sprints incluindo a revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| PRD-PUB-01 | `docs/PRD.md` | Público-alvo | Usuários do OMS que representam o cliente cadastram webhooks via API com JWT do sistema | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | `docs/PRD.md` | Público-alvo | Administradores (role ADMIN) reprocessam avisos esgotados | TRANSCRICAO | [09:36] Sofia |
| PRD-SCN-01 | `docs/PRD.md` | Cenário de uso | Cliente filtra só SHIPPED e DELIVERED (meta de tempo em [09:02] Marcos) | TRANSCRICAO | [09:33] Marcos |
| PRD-SCN-02 | `docs/PRD.md` | Cenário de uso | Indisponibilidade de 2 h em manutenção coberta pelo reenvio | TRANSCRICAO | [09:16] Diego |
| PRD-SCN-03 | `docs/PRD.md` | Cenário de uso | Cliente descarta aviso repetido pelo identificador do evento | TRANSCRICAO | [09:25] Diego |
| PRD-SCN-04 | `docs/PRD.md` | Cenário de uso | Rotação de secret com 24 h para o cliente migrar (motivação: vazamento em [09:22] Diego) | TRANSCRICAO | [09:21] Sofia |
| PRD-SCN-05 | `docs/PRD.md` | Cenário de uso | Replay por administrador com registro de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-SCN-06 | `docs/PRD.md` | Cenário de uso | Cliente consulta últimas entregas (sucesso/falha, payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo/Métrica | Aviso entregue em menos de 10 segundos (meta; resultado ainda não comprovado) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo | Funcionalidade disponível até o fim de novembro (pedido da Atlas) | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo | Não perder avisos por indisponibilidade: reenvio automático e reprocessamento manual | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Aviso aos webhooks interessados quando o status de um pedido muda | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Cadastro de webhook com endereço, status de interesse e customer explícito (campos em [09:31] Marcos) | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Secret gerada pela plataforma e devolvida no cadastro | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Edição de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Listagem dos webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Filtro por lista de status de interesse | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Histórico das últimas 100 entregas com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com a anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Aviso assinado com HMAC-SHA256 e secret exclusiva do webhook | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Identificador único do evento (X-Event-Id) para deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Reenvio automático: 5 tentativas, 1m/5m/30m/2h/12h (contagem pendente, I-1) | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | Falhas esgotadas retidas com motivo e reprocessáveis só por ADMIN | TRANSCRICAO | [09:36] Larissa |
| PRD-FR-14 | `docs/PRD.md` | Requisito Funcional | Reprocessamento manual registra quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Entrega em menos de 10 s em operação normal | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Mudança de status e aviso devido são atômicos | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Cliente lento ou fora do ar não bloqueia mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Somente endereços HTTPS; HTTP recusado no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Aviso acima de 64 KB gera erro, sem truncar (etapa pendente) | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Sem resposta em 10 s conta como falha e entra no reenvio | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Entrega pelo menos uma vez; duplicatas possíveis | TRANSCRICAO | [09:26] Larissa |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Aviso enxuto, sem itens (estado no momento da mudança: [09:52] Bruno) | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Ordem garantida só por pedido e com processador único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Secret exclusiva por webhook isola vazamentos | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-11 | `docs/PRD.md` | Requisito Não Funcional | Sem nova infraestrutura de mensageria; reuso do existente (também [09:07] Diego) | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-12 | `docs/PRD.md` | Restrição | Revisão de segurança da Sofia de pelo menos dois dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-EXC-01 | `docs/PRD.md` | Adiamento | Email de alerta ao cliente adiado para a próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-EXC-02 | `docs/PRD.md` | Exclusão | Dashboard visual fora do escopo (time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-EXC-03 | `docs/PRD.md` | Exclusão | Webhooks de entrada fora do escopo (somente saída) | TRANSCRICAO | [09:02] Marcos |
| PRD-EXC-04 | `docs/PRD.md` | Adiamento | Vários workers em paralelo adiados | TRANSCRICAO | [09:13] Diego |
| PRD-EXC-05 | `docs/PRD.md` | Exclusão | Arquivamento de avisos entregues (~30 dias) fora desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-EXC-06 | `docs/PRD.md` | Exclusão | Itens do pedido fora do aviso | TRANSCRICAO | [09:43] Diego |
| PRD-EXC-07 | `docs/PRD.md` | Alternativa rejeitada | Exactly-once descartado por exigir coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| PRD-EXC-08 | `docs/PRD.md` | Restrição | Ordenação global não prometida | TRANSCRICAO | [09:13] Larissa |
| PRD-EXC-09 | `docs/PRD.md` | Questão aberta | Rate limiting de saída em avaliação: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-RSK-01 | `docs/PRD.md` | Risco | R-01 (I-3): cadastro público aceita role ADMIN | CODIGO | `src/modules/auth/auth.schemas.ts` (registerSchema.role) |
| PRD-RSK-02 | `docs/PRD.md` | Risco | R-02 (G-12): sem vínculo usuário-customer; exposição entre clientes | CODIGO | `prisma/schema.prisma` (model User) |
| PRD-RSK-03 | `docs/PRD.md` | Risco | R-03 (I-1): ambiguidade de 5 tentativas x 5 intervalos | TRANSCRICAO | [09:48] Larissa |
| PRD-RSK-04 | `docs/PRD.md` | Risco | R-04: duplicatas processadas por clientes sem deduplicação | TRANSCRICAO | [09:24] Diego |
| PRD-RSK-05 | `docs/PRD.md` | Risco | R-05 [ANÁLISE]: cliente lento atrasa os demais (worker único + timeout) | TRANSCRICAO | [09:42] Diego |
| PRD-RSK-06 | `docs/PRD.md` | Risco | R-06: falha prolongada leva à retenção até ação manual | TRANSCRICAO | [09:16] Diego |
| PRD-RSK-07 | `docs/PRD.md` | Risco | R-07: prazo de fim de novembro com três sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Revisão de segurança da Sofia (HMAC e geração de secret) | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Documentação no portal do desenvolvedor (integração e duplicatas) | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Confirmação do prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | Clientes implementam verificação da assinatura | TRANSCRICAO | [09:20] Sofia |
| PRD-TRD-01 | `docs/PRD.md` | Trade-off | Registrar e enviar depois em vez de chamada síncrona | TRANSCRICAO | [09:04] Bruno |
| PRD-TRD-02 | `docs/PRD.md` | Trade-off | Banco existente em vez de Redis: menos infra, verificação periódica | TRANSCRICAO | [09:07] Diego |
| PRD-TRD-03 | `docs/PRD.md` | Trade-off | Processador separado a cada 2 s atende a meta de 10 s | TRANSCRICAO | [09:10] Marcos |
| PRD-TRD-04 | `docs/PRD.md` | Trade-off | Janela de ~15 h aceita pelo PM (depende de I-1) | TRANSCRICAO | [09:17] Marcos |
| PRD-TRD-05 | `docs/PRD.md` | Trade-off | Secret por webhook isola vazamentos; cliente implementa verificação | TRANSCRICAO | [09:21] Sofia |
| PRD-TRD-06 | `docs/PRD.md` | Trade-off | At-least-once transfere a deduplicação ao cliente | TRANSCRICAO | [09:25] Sofia |
| PRD-TRD-07 | `docs/PRD.md` | Trade-off | Reuso dos padrões do OMS; [ANÁLISE] herda fragilidades de acesso (I-3, G-12) | TRANSCRICAO | [09:30] Larissa |
| PRD-AC-01 | `docs/PRD.md` | Critério de aceitação | Aviso registrado na transição; nenhum aviso se a mudança falhar | TRANSCRICAO | [09:40] Bruno |
| PRD-AC-02 | `docs/PRD.md` | Critério de aceitação | Status fora do filtro não gera aviso | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-03 | `docs/PRD.md` | Critério de aceitação | HTTPS aceito e secret devolvida; HTTP recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-04 | `docs/PRD.md` | Critério de aceitação | Edição, listagem e remoção funcionam; [ANÁLISE] removido deixa de receber (pendentes: P-16) | TRANSCRICAO | [09:33] Bruno |
| PRD-AC-05 | `docs/PRD.md` | Critério de aceitação | Cliente recalcula HMAC-SHA256 do corpo e confere a assinatura | TRANSCRICAO | [09:22] Sofia |
| PRD-AC-06 | `docs/PRD.md` | Critério de aceitação | Secret anterior válida por 24 h após a rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-07 | `docs/PRD.md` | Critério de aceitação | Reenvio mantém o identificador do evento | TRANSCRICAO | [09:25] Diego |
| PRD-AC-08 | `docs/PRD.md` | Critério de aceitação | Reenvios seguem 1m/5m/30m/2h/12h (número de envios depende de I-1) | TRANSCRICAO | [09:17] Larissa |
| PRD-AC-09 | `docs/PRD.md` | Critério de aceitação | Retenção com motivo; só ADMIN reprocessa | TRANSCRICAO | [09:36] Larissa |
| PRD-AC-10 | `docs/PRD.md` | Critério de aceitação | Reprocessamento registra o usuário executor | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-11 | `docs/PRD.md` | Critério de aceitação | Consulta retorna até 100 entregas recentes | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-12 | `docs/PRD.md` | Critério de aceitação | [PROPOSTA de medição] Aviso em menos de 10 s em ambiente de teste | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-13 | `docs/PRD.md` | Critério de aceitação | Mudanças de status concluem com cliente indisponível | TRANSCRICAO | [09:04] Bruno |
| PRD-AC-14 | `docs/PRD.md` | Critério de aceitação | Aviso > 64 KB gera erro sem truncar (depende de P-10) | TRANSCRICAO | [09:24] Larissa |
| PRD-AC-15 | `docs/PRD.md` | Critério de aceitação | Usuário não acessa webhooks de customer alheio (depende de G-12) | CODIGO | `prisma/schema.prisma` (model User) |
| PRD-AC-16 | `docs/PRD.md` | Critério de aceitação | Revisão de segurança concluída antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-PUB-03 | `docs/PRD.md` | Público-alvo | Clientes B2B integrados recebem, validam assinatura e deduplicam | TRANSCRICAO | [09:00] Marcos |
| PRD-PUB-04 | `docs/PRD.md` | Público-alvo | Time de produto e portal do desenvolvedor documentam a integração | TRANSCRICAO | [09:26] Marcos |
| ADR-001-P05 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Risco | Crescimento da outbox sem arquivamento nesta fase | TRANSCRICAO | [09:08] Diego |
| ADR-002-C02 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Fato do código | createPrismaClient e singleton prisma criado na importação | CODIGO | `src/config/database.ts` (createPrismaClient, prisma) |
| ADR-006-I2N | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Proposta técnica | [PROPOSTA] Não alterar o validate compartilhado (contrato global) | CODIGO | `src/middlewares/validate.middleware.ts` (validate) |
| ADR-006-I2Q1 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Questão aberta | I-2: status 400 ou 422 para URL inválida | CODIGO | `src/shared/errors/http-errors.ts` (ValidationError, UnprocessableEntityError) |
| ADR-006-I2Q2 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Questão aberta | I-2: classes em src/shared/errors ou dentro do módulo | CODIGO | `src/shared/errors/http-errors.ts` (BadRequestError) |
| ADR-006-I2Q3 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Questão aberta | Semântica de WEBHOOK_SECRET_REQUIRED com secret gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| ADR-006-C06 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Fato do código | Testes de registro só usam role OPERATOR (sem teste negativo de ADMIN) | CODIGO | `tests/auth.test.ts` (Auth flow) |
| FDD-FL-41 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Secret anterior descartada após previousSecretExpiresAt | TRANSCRICAO | [09:21] Sofia |
| FDD-FL-42 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] X-Timestamp em ISO 8601 | TRANSCRICAO | [09:44] Diego |
| FDD-FL-43 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Histórico limitado a 100 itens, sem paginação | TRANSCRICAO | [09:34] Marcos |
| FDD-API-12 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] E2 com envelope data e campo secretRotationPending | CODIGO | `src/shared/http/response.ts` (paginated) |
| FDD-API-13 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] E7 responde 200 (202 defensável) | TRANSCRICAO | [09:35] Diego |
| FDD-OBS-05 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Catálogo de eventos de log em snake_case | CODIGO | `src/middlewares/request-logger.middleware.ts` (requestLogger) |
| FDD-OBS-06 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Métricas emitidas como log estruturado periódico, sem coletor novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-07 | `docs/FDD.md` | Fato do código | Não há OpenTelemetry nem biblioteca de métricas no projeto | CODIGO | `package.json` (dependencies) |
| FDD-INT-16 | `docs/FDD.md` | Risco (código) | Rotas /customers/:customerId/webhooks no espaço do router de customers: ordem de montagem a testar | CODIGO | `src/modules/customers/customer.routes.ts` (buildCustomerRouter) |
| FDD-INT-17 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Injetar ou importar o publisher no OrderService | CODIGO | `src/app.ts` (buildControllers) |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Três clientes B2B pediram notificação; hoje consultam a API repetidamente | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | `docs/RFC.md` | Requisito Não Funcional | Entrega aceitável em menos de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | `docs/RFC.md` | Restrição | Fluxo exclusivamente outbound | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-04 | `docs/RFC.md` | Fato do código | Stack MySQL + Prisma | CODIGO | `prisma/schema.prisma` (datasource provider mysql) |
| RFC-CTX-05 | `docs/RFC.md` | Fato do código | changeStatus valida transição e atualiza estoque, pedido e histórico numa transação | CODIGO | `src/modules/orders/order.service.ts` (OrderService.changeStatus) |
| RFC-CTX-06 | `docs/RFC.md` | Contexto | Envio síncrono comprometeria a transação de pedidos | TRANSCRICAO | [09:04] Bruno |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Outbox no MySQL na transação de changeStatus | TRANSCRICAO | [09:08] Larissa |
| RFC-PROP-02 | `docs/RFC.md` | Decisão | Falha ao gravar na outbox desfaz a mudança do pedido | TRANSCRICAO | [09:40] Bruno |
| RFC-PROP-03 | `docs/RFC.md` | Detalhe acordado | Eventos inseridos só para filtros cadastrados | TRANSCRICAO | [09:34] Bruno |
| RFC-PROP-04 | `docs/RFC.md` | Decisão | Payload como snapshot do momento da alteração | TRANSCRICAO | [09:52] Bruno |
| RFC-PROP-05 | `docs/RFC.md` | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| RFC-PROP-06 | `docs/RFC.md` | Decisão | Polling a cada 2 segundos | TRANSCRICAO | [09:10] Larissa |
| RFC-PROP-07 | `docs/RFC.md` | Detalhe acordado | Worker com PrismaClient próprio no mesmo MySQL | TRANSCRICAO | [09:30] Bruno |
| RFC-PROP-08 | `docs/RFC.md` | Restrição | Worker único com limites de ordenação; escala adiada | TRANSCRICAO | [09:13] Larissa |
| RFC-PROP-09 | `docs/RFC.md` | Avaliação da equipe | 2 s considerado adequado à meta (ressalva de análise rotulada) | TRANSCRICAO | [09:09] Diego |
| RFC-PROP-10 | `docs/RFC.md` | Detalhe acordado | Timeout HTTP de 10 segundos | TRANSCRICAO | [09:42] Diego |
| RFC-PROP-11 | `docs/RFC.md` | Decisão | 5 tentativas com 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-12 | `docs/RFC.md` | Decisão | DLQ persistida em tabela separada | TRANSCRICAO | [09:18] Diego |
| RFC-PROP-13 | `docs/RFC.md` | Decisão | Replay manual exige ADMIN via requireRole (auditoria: [09:36] Sofia) | TRANSCRICAO | [09:36] Larissa |
| RFC-PROP-14 | `docs/RFC.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, convivência de 24 h | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-15 | `docs/RFC.md` | Detalhe acordado | Secret gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| RFC-PROP-16 | `docs/RFC.md` | Restrição | HTTPS obrigatório | TRANSCRICAO | [09:23] Sofia |
| RFC-PROP-17 | `docs/RFC.md` | Requisito Não Funcional | Payload acima de 64 KB gera erro, sem truncamento | TRANSCRICAO | [09:24] Larissa |
| RFC-PROP-18 | `docs/RFC.md` | Decisão | At-least-once com deduplicação via X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-19 | `docs/RFC.md` | Decisão | Módulo src/modules/webhooks reutilizando AppError, Pino, middleware de erro e Zod (Prisma e JWT: componentes já existentes) | TRANSCRICAO | [09:30] Larissa |
| RFC-PROP-20 | `docs/RFC.md` | Decisão | Códigos de erro prefixados por WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| RFC-PROP-21 | `docs/RFC.md` | Decisão | customer_id não vem do JWT atual | TRANSCRICAO | [09:32] Larissa |
| RFC-PROP-22 | `docs/RFC.md` | Decisão | CRUD autenticado para qualquer role, por enquanto | TRANSCRICAO | [09:37] Sofia |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa rejeitada | Notificação síncrona dentro da transação | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa rejeitada | Redis Streams ou fila adicional | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa rejeitada | Trigger MySQL para sinalizar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa rejeitada | Exactly-once ponta a ponta | TRANSCRICAO | [09:25] Diego |
| RFC-QA-01 | `docs/RFC.md` | Questão aberta | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-QA-02 | `docs/RFC.md` | Questão aberta | Endurecimento futuro do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-03 | `docs/RFC.md` | Questão aberta | customer_id no body ou no path | TRANSCRICAO | [09:32] Larissa |
| RFC-EXC-01 | `docs/RFC.md` | Adiamento | Email adiado | TRANSCRICAO | [09:37] Larissa |
| RFC-EXC-02 | `docs/RFC.md` | Exclusão | Dashboard fora do escopo | TRANSCRICAO | [09:40] Larissa |
| RFC-EXC-03 | `docs/RFC.md` | Exclusão | Webhooks de entrada fora do escopo | TRANSCRICAO | [09:02] Marcos |
| RFC-EXC-04 | `docs/RFC.md` | Exclusão | Arquivamento da outbox fora do escopo | TRANSCRICAO | [09:08] Diego |
| RFC-EXC-05 | `docs/RFC.md` | Adiamento | Múltiplos workers adiados | TRANSCRICAO | [09:13] Diego |
| RFC-PEND-01 | `docs/RFC.md` | Pendência (análise) | I-1: 5 tentativas x 5 intervalos; NEEDS_DECISION Larissa/Diego | TRANSCRICAO | [09:48] Larissa |
| RFC-PEND-02 | `docs/RFC.md` | Pendência (análise) | I-2: validate converte Zod em VALIDATION_ERROR, não WEBHOOK_INVALID_URL | CODIGO | `src/middlewares/validate.middleware.ts` (validate) |
| RFC-PEND-03 | `docs/RFC.md` | Risco (análise) | I-3: registro público aceita ADMIN, propagado ao JWT | CODIGO | `src/modules/auth/auth.schemas.ts` (registerSchema) |
| RFC-PEND-04 | `docs/RFC.md` | Risco (análise) | G-12: sem vínculo usuário-customer; risco elevado à revisão da Sofia | CODIGO | `prisma/schema.prisma` (model User) |
| RFC-LAC-01 | `docs/RFC.md` | Pendência (análise) | Qual secret assina durante as 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| RFC-LAC-02 | `docs/RFC.md` | Pendência (análise) | X-Timestamp dentro ou fora do HMAC | TRANSCRICAO | [09:44] Diego |
| RFC-LAC-03 | `docs/RFC.md` | Pendência (análise) | Replay preserva ou não o event_id original | TRANSCRICAO | [09:18] Diego |
| RFC-LAC-04 | `docs/RFC.md` | Pendência (análise) | OrderService.create gera ou não evento | CODIGO | `src/modules/orders/order.service.ts` (OrderService.create) |
| RFC-LAC-05 | `docs/RFC.md` | Pendência (análise) | Limite de 64 KB aplicado na inserção ou no envio | TRANSCRICAO | [09:24] Larissa |
| RFC-LAC-06 | `docs/RFC.md` | Pendência (análise) | Retries podem reordenar avisos do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| RFC-RSK-01 | `docs/RFC.md` | Risco (análise) | Critério de sucesso da entrega não definido (só timeout) | TRANSCRICAO | [09:42] Diego |
| RFC-RSK-02 | `docs/RFC.md` | Risco (análise) | Fan-out para vários endpoints do cliente | TRANSCRICAO | [09:44] Sofia |
| RFC-RSK-03 | `docs/RFC.md` | Risco (análise) | Recuperação de registros presos em processamento | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | Tabelas Outbox/DLQ, módulo, worker, integração com changeStatus, sem mensageria nova | TRANSCRICAO | [09:06] Diego |
| RFC-DEP-01 | `docs/RFC.md` | Dependência | Pelo menos dois dias úteis de revisão da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| RFC-DEP-02 | `docs/RFC.md` | Restrição | Estimativa de três sprints | TRANSCRICAO | [09:46] Larissa |
| RFC-NEXT-01 | `docs/RFC.md` | Próximo passo | Sessão de revisão do design com Bruno, Diego e Larissa antes de codar | TRANSCRICAO | [09:50] Larissa |
| ADR-001-D01 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Decisão | Outbox em tabela no MySQL existente | TRANSCRICAO | [09:08] Larissa |
| ADR-001-D02 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Decisão | Inserção na mesma transação de changeStatus; falha desfaz tudo | TRANSCRICAO | [09:40] Bruno |
| ADR-001-D03 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Detalhe acordado | publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno |
| ADR-001-D04 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Detalhe acordado | Filtro aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-001-D05 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Decisão | Payload snapshot na inserção | TRANSCRICAO | [09:52] Bruno |
| ADR-001-D06 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Detalhe acordado | UUID como identificador da outbox | TRANSCRICAO | [09:51] Larissa |
| ADR-001-D07 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Proposta técnica | [PROPOSTA] Índices em status e created_at; leitura em batch pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-C01 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Fato do código | Etapas da transação de changeStatus e tipo TxClient | CODIGO | `src/modules/orders/order.service.ts` (OrderService.changeStatus) |
| ADR-001-C02 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Fato do código | Débito só em PENDING->PAID; reposição no cancelamento | CODIGO | `src/modules/orders/order.status.ts` (shouldDebitStock, shouldReplenishStock) |
| ADR-001-A01 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Alternativa rejeitada | Disparo HTTP síncrono | TRANSCRICAO | [09:04] Bruno |
| ADR-001-A02 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Alternativa rejeitada | Redis Streams: overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-A03 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Alternativa rejeitada | Injetar repository inteiro no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-001-A04 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Alternativa rejeitada | Filtrar na hora do envio (pergunta de [09:34] Diego) | TRANSCRICAO | [09:34] Bruno |
| ADR-001-A05 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Alternativa rejeitada | Renderizar o payload no envio | TRANSCRICAO | [09:52] Larissa |
| ADR-001-P01 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Pendência (análise) | Criação do pedido grava PENDING sem changeStatus | CODIGO | `src/modules/orders/order.service.ts` (OrderService.create) |
| ADR-001-P02 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Pendência (análise) | Fan-out: linha por evento ou por webhook | TRANSCRICAO | [09:44] Sofia |
| ADR-001-P03 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Pendência (análise) | Onde aplicar o limite de 64 KB | TRANSCRICAO | [09:24] Larissa |
| ADR-001-P04 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Risco (análise) | findUnique sem lock permite transições concorrentes | CODIGO | `src/modules/orders/order.service.ts` (OrderService.changeStatus) |
| ADR-002-D01 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Polling a cada 2 segundos | TRANSCRICAO | [09:10] Larissa |
| ADR-002-D02 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Worker em processo separado (anotado) | TRANSCRICAO | [09:12] Larissa |
| ADR-002-D03 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Detalhe acordado | Entry point src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D04 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Detalhe acordado | PrismaClient próprio com a mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-D05 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Detalhe acordado | Worker único; ordem só por order_id como limitação conhecida | TRANSCRICAO | [09:13] Larissa |
| ADR-002-D06 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Questão aberta | Nome do arquivo de processamento: webhook.worker.ts ou webhook.processor.ts | TRANSCRICAO | [09:28] Bruno |
| ADR-002-D07 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Proposta técnica | [PROPOSTA] Worker via createPrismaClient; env exige JWT_SECRET | CODIGO | `src/config/env.ts` (envSchema) |
| ADR-002-C01 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Fato do código | Único entry point atual com tratamento de SIGINT/SIGTERM | CODIGO | `src/server.ts` (bootstrap) |
| ADR-002-A01 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Alternativa rejeitada | Trigger de banco: MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-A02 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Alternativa rejeitada | Worker dentro do processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-A03 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Alternativa rejeitada | Mesmo PrismaClient da API | TRANSCRICAO | [09:30] Bruno |
| ADR-002-A04 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Adiamento | Vários workers (particionamento ou lock pessimista) | TRANSCRICAO | [09:13] Diego |
| ADR-002-P01 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Risco (análise) | Worker sequencial com timeout de 10 s atrasa outros clientes | TRANSCRICAO | [09:42] Diego |
| ADR-002-P02 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Risco (análise) | Retries podem reordenar eventos do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| ADR-002-P03 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Risco (análise) | created_at não representa ordem de commit | TRANSCRICAO | [09:12] Diego |
| ADR-002-P04 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Pendência (análise) | Recuperação de linhas presas em processando | TRANSCRICAO | [09:08] Diego |
| ADR-003-D01 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Decisão | Retry com backoff exponencial e teto de tentativas (proposta em [09:15] Diego) | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D02 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Decisão | 5 tentativas; intervalos 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D03 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Decisão | DLQ em tabela separada (confirmado em [09:48] Larissa) | TRANSCRICAO | [09:18] Diego |
| ADR-003-D04 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Detalhe acordado | Tabela webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-003-D05 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Decisão | Reprocessamento manual por endpoint admin | TRANSCRICAO | [09:19] Larissa |
| ADR-003-D06 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Detalhe acordado | POST /admin/webhooks/dead-letter/:id/replay recoloca como pendente | TRANSCRICAO | [09:35] Diego |
| ADR-003-D07 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Decisão | Replay exige ADMIN com requireRole | TRANSCRICAO | [09:36] Larissa |
| ADR-003-D08 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Detalhe acordado | Replay registra quem executou | TRANSCRICAO | [09:36] Sofia |
| ADR-003-D09 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Detalhe acordado | Timeout HTTP de 10 s tratado como falha | TRANSCRICAO | [09:42] Diego |
| ADR-003-I1 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Pendência (análise) | I-1: leitura A (5 envios, 2h36) x leitura B (6 envios, 14h36); esclarecer com Larissa e Diego | TRANSCRICAO | [09:48] Larissa |
| ADR-003-A01 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Alternativa rejeitada | Retry indefinido (citado sem proponente) | TRANSCRICAO | [09:15] Diego |
| ADR-003-A02 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Alternativa rejeitada | 3 tentativas propostas por Bruno, insuficientes para 2 h de indisponibilidade | TRANSCRICAO | [09:16] Diego |
| ADR-003-A03 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Alternativa rejeitada | Marcar failed na própria outbox (pergunta de [09:17] Larissa) | TRANSCRICAO | [09:18] Diego |
| ADR-003-P01 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Pendência (análise) | O que conta como sucesso e retry de 4xx | TRANSCRICAO | [09:42] Diego |
| ADR-003-P02 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Pendência (análise) | Papel do status falhou na outbox | TRANSCRICAO | [09:08] Diego |
| ADR-003-P03 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Pendência (análise) | Replay preserva o event_id? | TRANSCRICAO | [09:18] Diego |
| ADR-003-P04 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Pendência (análise) | Mecanismo do registro de auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| ADR-004-D01 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Decisão | HMAC-SHA256 sobre o corpo do request | TRANSCRICAO | [09:22] Sofia |
| ADR-004-D02 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Detalhe acordado | Assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D03 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Decisão | Secret única por endpoint, nunca global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D04 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Decisão | Rotação pela API com a anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D05 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Detalhe acordado | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| ADR-004-D06 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Detalhe acordado | Configuração guarda url, secret, customer_id e ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-D07 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Detalhe acordado | URL obrigatoriamente https (validação Zod) | TRANSCRICAO | [09:23] Sofia |
| ADR-004-D08 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Detalhe acordado | Revisão de segurança de no mínimo dois dias úteis | TRANSCRICAO | [09:46] Sofia |
| ADR-004-C01 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Fato do código | redactPaths do logger não cobre secret | CODIGO | `src/shared/logger/index.ts` (redactPaths) |
| ADR-004-C02 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Contexto | Cliente já vazou secret em log da própria aplicação | TRANSCRICAO | [09:22] Diego |
| ADR-004-A01 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Alternativa rejeitada | Secret global (descartada sem proponente) | TRANSCRICAO | [09:21] Sofia |
| ADR-004-P01 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Pendência (análise) | Qual secret assina durante a convivência de 24 h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-P02 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Pendência (análise) | X-Timestamp entra no conteúdo assinado? | TRANSCRICAO | [09:44] Diego |
| ADR-004-P03 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Pendência (análise) | Armazenamento da secret em repouso (precisa ser recuperável) | TRANSCRICAO | [09:22] Diego |
| ADR-004-P04 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Pendência (análise) | Contrato do endpoint de rotação | TRANSCRICAO | [09:21] Sofia |
| ADR-004-P05 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Incluir secret no redactPaths | CODIGO | `src/shared/logger/index.ts` (redactPaths) |
| ADR-005-D01 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Decisão | Garantia de entrega at-least-once | TRANSCRICAO | [09:26] Larissa |
| ADR-005-D02 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Decisão | X-Event-Id com UUID gerado na entrada na outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-D03 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Detalhe acordado | event_id também no corpo do payload | TRANSCRICAO | [09:43] Diego |
| ADR-005-D04 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Detalhe acordado | Semântica documentada no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-005-D05 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Trade-off | Deduplicação é responsabilidade do cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-D06 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Proposta técnica | [PROPOSTA] Causas de duplicata: timeout após processamento, queda antes do ack, replay | TRANSCRICAO | [09:24] Diego |
| ADR-005-A01 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Alternativa rejeitada | Exactly-once: exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-P01 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Pendência (análise) | Replay preserva o event_id original? | TRANSCRICAO | [09:18] Diego |
| ADR-005-P02 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Pendência (análise) | Mesmo X-Event-Id para vários webhooks do customer? | TRANSCRICAO | [09:44] Sofia |
| ADR-005-P03 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Risco | Clientes sem deduplicação; mitigação no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-006-D01 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-D02 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Decisão | Módulo src/modules/webhooks com controller, service, repository, routes, schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-D03 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Decisão | Prefixo WEBHOOK_ em todos os códigos de erro | TRANSCRICAO | [09:29] Larissa |
| ADR-006-D04 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Detalhe acordado | Exemplos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| ADR-006-D05 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Decisão | Nenhum logger novo: Pino | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D06 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Detalhe acordado | Error middleware trata os novos erros sem mudança | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D07 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Decisão | Replay reaproveita requireRole com ADMIN | TRANSCRICAO | [09:36] Larissa |
| ADR-006-D08 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Decisão | CRUD com autenticação normal e qualquer role, por enquanto | TRANSCRICAO | [09:37] Sofia |
| ADR-006-D09 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Decisão | customer_id não vem do JWT (anotado) | TRANSCRICAO | [09:33] Bruno |
| ADR-006-D10 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Detalhe acordado | Lógica de processamento dentro do módulo | TRANSCRICAO | [09:28] Bruno |
| ADR-006-C01 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Fato do código | AppError com statusCode, errorCode e details | CODIGO | `src/shared/errors/app-error.ts` (AppError) |
| ADR-006-C02 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Fato do código | errorMiddleware serializa AppError, ZodError e Prisma P2002/P2025 | CODIGO | `src/middlewares/error.middleware.ts` (errorMiddleware) |
| ADR-006-C03 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Fato do código | NotFoundError fixa NOT_FOUND; WEBHOOK_NOT_FOUND exige classe própria | CODIGO | `src/shared/errors/http-errors.ts` (NotFoundError) |
| ADR-006-C04 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Fato do código | Composição manual de dependências e rotas sob /api/v1 | CODIGO | `src/app.ts` (buildControllers, buildApp) |
| ADR-006-C05 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Fato do código | Log de requisição com X-Request-Id | CODIGO | `src/middlewares/request-logger.middleware.ts` (requestLogger) |
| ADR-006-I2 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Pendência (análise) | I-2: validate fixa VALIDATION_ERROR; WEBHOOK_INVALID_URL nunca emitido | CODIGO | `src/middlewares/validate.middleware.ts` (validate) |
| ADR-006-I2P | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Proposta técnica | [PROPOSTA] Opções P1/P2/P3 para emitir WEBHOOK_INVALID_URL | TRANSCRICAO | [09:28] Bruno |
| ADR-006-R01 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Risco (código) | I-3: registerSchema aceita role ADMIN | CODIGO | `src/modules/auth/auth.schemas.ts` (registerSchema) |
| ADR-006-R02 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Risco (análise) | requireRole confia no role do token; replay sem proteção efetiva | CODIGO | `src/middlewares/auth.middleware.ts` (authenticate, requireRole) |
| ADR-006-R03 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Risco (código) | G-12: User sem relação com Customer | CODIGO | `prisma/schema.prisma` (model User) |
| ADR-006-R04 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Risco (análise) | Exposição de eventos entre clientes via CRUD liberado | TRANSCRICAO | [09:37] Sofia |
| ADR-006-M01 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Impedir ADMIN no registro público | CODIGO | `src/modules/auth/auth.routes.ts` (buildAuthRouter /register) |
| ADR-006-M02 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Revisar ADMIN existentes e tokens emitidos | CODIGO | `src/modules/auth/auth.service.ts` (AuthService.login, signToken) |
| ADR-006-M03 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Autorização por objeto ou aceite formal do risco | TRANSCRICAO | [09:37] Sofia |
| ADR-006-M04 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Testes negativos de replay com OPERATOR | TRANSCRICAO | [09:36] Larissa |
| ADR-006-M05 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Redação de secrets nos logs | CODIGO | `src/shared/logger/index.ts` (redactPaths) |
| ADR-006-A01 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Alternativa rejeitada | Introduzir componentes novos como outro logger | TRANSCRICAO | [09:29] Bruno |
| ADR-006-A02 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Alternativa (análise) | Usar só códigos genéricos (plausível, não proposta na reunião) | TRANSCRICAO | [09:29] Larissa |
| ADR-006-A03 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Alternativa rejeitada | Derivar customer_id do JWT | TRANSCRICAO | [09:31] Marcos |
| ADR-006-P01 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Questão aberta | Catálogo completo de códigos WEBHOOK_ (etc.) | TRANSCRICAO | [09:28] Bruno |
| ADR-006-P02 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Questão aberta | customer_id no body ou no path | TRANSCRICAO | [09:32] Larissa |
| ADR-006-P03 | `docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md` | Adiamento | Endurecer roles do CRUD mais pra frente | TRANSCRICAO | [09:37] Sofia |
| FDD-CTX-01 | `docs/FDD.md` | Fato do código | changeStatus executa validação, estoque, update e histórico em $transaction | CODIGO | `src/modules/orders/order.service.ts` (OrderService.changeStatus) |
| FDD-CTX-02 | `docs/FDD.md` | Fato do código | Nenhum módulo de notificação montado no app | CODIGO | `src/app.ts` (buildControllers) |
| FDD-CTX-03 | `docs/FDD.md` | Alternativa rejeitada | Chamada síncrona descartada | TRANSCRICAO | [09:04] Bruno |
| FDD-OT-01 | `docs/FDD.md` | Objetivo técnico | Evento registrado atomicamente com a mudança | TRANSCRICAO | [09:08] Larissa |
| FDD-OT-02 | `docs/FDD.md` | Objetivo técnico | Worker separado com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| FDD-OT-03 | `docs/FDD.md` | Objetivo técnico | Meta de entrega em menos de 10 s com 2 s de polling | TRANSCRICAO | [09:10] Marcos |
| FDD-OT-04 | `docs/FDD.md` | Objetivo técnico | HMAC-SHA256 com secret por endpoint e rotação de 24 h | TRANSCRICAO | [09:22] Sofia |
| FDD-OT-05 | `docs/FDD.md` | Objetivo técnico | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| FDD-OT-06 | `docs/FDD.md` | Objetivo técnico | Retry 5 tentativas, DLQ e replay ADMIN | TRANSCRICAO | [09:17] Larissa |
| FDD-OT-07 | `docs/FDD.md` | Objetivo técnico | Reuso de arquitetura, AppError, Pino, Zod e requireRole | TRANSCRICAO | [09:30] Larissa |
| FDD-NG-01 | `docs/FDD.md` | Não-objetivo | Exactly-once não prometido | TRANSCRICAO | [09:25] Diego |
| FDD-NG-02 | `docs/FDD.md` | Não-objetivo | Ordenação global não prometida | TRANSCRICAO | [09:13] Larissa |
| FDD-EXC-01 | `docs/FDD.md` | Adiamento | Email adiado | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-02 | `docs/FDD.md` | Exclusão | Dashboard fora do escopo | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-03 | `docs/FDD.md` | Exclusão | Webhooks de entrada fora | TRANSCRICAO | [09:02] Marcos |
| FDD-EXC-04 | `docs/FDD.md` | Exclusão | Arquivamento fora desta feature | TRANSCRICAO | [09:08] Diego |
| FDD-EXC-05 | `docs/FDD.md` | Adiamento | Vários workers adiados | TRANSCRICAO | [09:13] Diego |
| FDD-EXC-06 | `docs/FDD.md` | Questão aberta | Rate limiting: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| FDD-EXC-07 | `docs/FDD.md` | Exclusão | Itens do pedido fora do payload | TRANSCRICAO | [09:43] Diego |
| FDD-COMP-01 | `docs/FDD.md` | Componente | Módulo src/modules/webhooks (a criar) | TRANSCRICAO | [09:27] Bruno |
| FDD-COMP-02 | `docs/FDD.md` | Componente | publishWebhookEvent com tx (a criar) | TRANSCRICAO | [09:41] Bruno |
| FDD-COMP-03 | `docs/FDD.md` | Componente | Processador webhook.worker.ts ou webhook.processor.ts (a criar) | TRANSCRICAO | [09:28] Bruno |
| FDD-COMP-04 | `docs/FDD.md` | Componente | Entry point src/worker.ts e npm run worker (a criar) | TRANSCRICAO | [09:11] Larissa |
| FDD-DM-01 | `docs/FDD.md` | Modelo de dados | [PROPOSTA] WebhookEndpoint com url, secret, customerId, active | TRANSCRICAO | [09:21] Bruno |
| FDD-DM-02 | `docs/FDD.md` | Modelo de dados | [PROPOSTA] WebhookOutbox com status PENDING/PROCESSING/FAILED/DELIVERED e índices | TRANSCRICAO | [09:08] Diego |
| FDD-DM-03 | `docs/FDD.md` | Modelo de dados | [PROPOSTA] WebhookDeadLetter com payload, reason, failedAt | TRANSCRICAO | [09:18] Diego |
| FDD-DM-04 | `docs/FDD.md` | Modelo de dados | [PROPOSTA] WebhookDelivery para o histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-DM-05 | `docs/FDD.md` | Fato do código | Convenções: uuid() Char(36), @@map snake_case, @@index | CODIGO | `prisma/schema.prisma` |
| FDD-DM-06 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Uma linha de outbox por evento x webhook (P-11) | TRANSCRICAO | [09:44] Sofia |
| FDD-DM-07 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] orderId sem FK: exclusão de pedido permitida em PENDING/CANCELLED | CODIGO | `src/modules/orders/order.service.ts` (OrderService.delete) |
| FDD-DM-08 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] onDelete Cascade do customer para evitar P2003/500 | CODIGO | `src/modules/customers/customer.service.ts` (CustomerService.delete) |
| FDD-DM-09 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Migração apenas aditiva sobre o schema atual | CODIGO | `prisma/schema.prisma` |
| FDD-FL-01 | `docs/FDD.md` | Regra de autorização | CRUD aceita qualquer role autenticada (por enquanto) | TRANSCRICAO | [09:37] Sofia |
| FDD-FL-02 | `docs/FDD.md` | Fato do código | Padrão de validação z.nativeEnum(OrderStatus) | CODIGO | `src/modules/orders/order.schemas.ts` (updateOrderStatusSchema) |
| FDD-FL-03 | `docs/FDD.md` | Restrição | Somente URLs https | TRANSCRICAO | [09:23] Sofia |
| FDD-FL-04 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Secret devolvida só na criação e rotação | TRANSCRICAO | [09:31] Marcos |
| FDD-FL-05 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] PATCH edita url, statuses e active | TRANSCRICAO | [09:33] Bruno |
| FDD-FL-06 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Listagem sem secret | TRANSCRICAO | [09:22] Diego |
| FDD-FL-07 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] customerId no path (P-13) | TRANSCRICAO | [09:32] Larissa |
| FDD-FL-09 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] P1: checagem https no service com WEBHOOK_INVALID_URL | TRANSCRICAO | [09:28] Bruno |
| FDD-FL-10 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Secret via crypto.randomBytes(32), sujeita à revisão | TRANSCRICAO | [09:46] Sofia |
| FDD-FL-11 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Rotação move secret para previousSecret com expiração de 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-FL-12 | `docs/FDD.md` | Fato do código | Passos atuais da transação: findUnique, validação, estoque, update, histórico, refresh | CODIGO | `src/modules/orders/order.service.ts` (OrderService.changeStatus) |
| FDD-FL-13 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] publishWebhookEvent após o histórico e antes do refresh | TRANSCRICAO | [09:41] Bruno |
| FDD-FL-14 | `docs/FDD.md` | Regra de negócio | Sem webhook interessado, nada é inserido | TRANSCRICAO | [09:34] Bruno |
| FDD-FL-15 | `docs/FDD.md` | Regra de negócio | Um eventId UUID por evento | TRANSCRICAO | [09:25] Diego |
| FDD-FL-16 | `docs/FDD.md` | Invariante | Exceção desfaz status e evento | TRANSCRICAO | [09:40] Bruno |
| FDD-FL-17 | `docs/FDD.md` | Risco (código) | Prisma 5.22.0 em uso; tempo-limite padrão da transação interativa a confirmar na documentação externa | CODIGO | `package.json` (@prisma/client 5.22.0) |
| FDD-FL-18 | `docs/FDD.md` | Contrato | Campos do payload: event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-FL-19 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] timestamp do payload = instante da mudança | TRANSCRICAO | [09:43] Diego |
| FDD-FL-20 | `docs/FDD.md` | Fato do código | order_number no formato ORD-NNNNNN | CODIGO | `src/modules/orders/order.service.ts` (reserveOrderNumber) |
| FDD-FL-21 | `docs/FDD.md` | Fato do código | PENDING nunca é destino de changeStatus | CODIGO | `src/modules/orders/order.status.ts` (transitions) |
| FDD-FL-22 | `docs/FDD.md` | Regra de negócio | [ANÁLISE] Filtro avaliado na inserção; webhook posterior não recebe eventos antigos | TRANSCRICAO | [09:34] Bruno |
| FDD-FL-23 | `docs/FDD.md` | Fato do código | Padrão de bootstrap e encerramento gracioso para o worker | CODIGO | `src/server.ts` (bootstrap, shutdown) |
| FDD-FL-24 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Ticks de 2 s sem sobreposição | TRANSCRICAO | [09:09] Diego |
| FDD-FL-25 | `docs/FDD.md` | Detalhe acordado | Busca dos pendentes mais antigos a cada ciclo | TRANSCRICAO | [09:09] Diego |
| FDD-FL-26 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Reivindicação condicional PENDING->PROCESSING | TRANSCRICAO | [09:12] Diego |
| FDD-FL-27 | `docs/FDD.md` | Estado/transição | Estados pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-FL-28 | `docs/FDD.md` | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp e Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-FL-29 | `docs/FDD.md` | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-FL-30 | `docs/FDD.md` | Regra de segurança | HMAC sobre os mesmos bytes enviados | TRANSCRICAO | [09:22] Sofia |
| FDD-FL-31 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Codificação sha256=<hex> no X-Signature | TRANSCRICAO | [09:20] Sofia |
| FDD-FL-32 | `docs/FDD.md` | Fato do código | Node >= 20 exigido; disponibilidade de fetch e AbortSignal.timeout é fato de plataforma | CODIGO | `package.json` (engines.node) |
| FDD-FL-33 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Falha incrementa attempts e reagenda nextAttemptAt | TRANSCRICAO | [09:17] Diego |
| FDD-FL-34 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Mover para DLQ e marcar outbox numa única transação | TRANSCRICAO | [09:18] Diego |
| FDD-FL-36 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Falhas sem retry (64 KB no envio, webhook inativo) | TRANSCRICAO | [09:23] Sofia |
| FDD-FL-37 | `docs/FDD.md` | Regra de autorização | Replay com authenticate + requireRole(ADMIN) | TRANSCRICAO | [09:36] Larissa |
| FDD-FL-38 | `docs/FDD.md` | Detalhe acordado | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FL-39 | `docs/FDD.md` | Detalhe acordado | Replay registra autor (replayedById e log Pino) | TRANSCRICAO | [09:36] Sofia |
| FDD-FL-40 | `docs/FDD.md` | Detalhe acordado | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| FDD-API-01 | `docs/FDD.md` | Contrato | E1 POST /customers/:customerId/webhooks (caminho proposto) | TRANSCRICAO | [09:31] Marcos |
| FDD-API-02 | `docs/FDD.md` | Contrato | E2 GET /customers/:customerId/webhooks (caminho proposto) | TRANSCRICAO | [09:33] Bruno |
| FDD-API-03 | `docs/FDD.md` | Contrato | E3 PATCH /webhooks/:id (caminho proposto) | TRANSCRICAO | [09:33] Bruno |
| FDD-API-04 | `docs/FDD.md` | Contrato | E4 DELETE /webhooks/:id (caminho proposto) | TRANSCRICAO | [09:33] Bruno |
| FDD-API-05 | `docs/FDD.md` | Contrato | E5 POST /webhooks/:id/secret/rotate (caminho proposto) | TRANSCRICAO | [09:21] Sofia |
| FDD-API-06 | `docs/FDD.md` | Contrato | E6 GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-API-07 | `docs/FDD.md` | Contrato | E7 POST /admin/webhooks/dead-letter/:id/replay (ADMIN) | TRANSCRICAO | [09:35] Diego |
| FDD-API-08 | `docs/FDD.md` | Fato do código | Prefixo /api/v1 para todas as rotas | CODIGO | `src/app.ts` (buildApp) |
| FDD-API-09 | `docs/FDD.md` | Fato do código | Formato de erro { error: { code, message, details } } | CODIGO | `src/middlewares/error.middleware.ts` (errorMiddleware) |
| FDD-API-10 | `docs/FDD.md` | Fato do código | Exclusão responde 204 sem corpo | CODIGO | `src/modules/orders/order.controller.ts` (OrderController.delete) |
| FDD-API-11 | `docs/FDD.md` | Fato do código | Campos da API em camelCase (customerId, toStatus) | CODIGO | `src/modules/orders/order.schemas.ts` (createOrderSchema, updateOrderStatusSchema) |
| FDD-ERR-01 | `docs/FDD.md` | Contrato de erro | WEBHOOK_NOT_FOUND 404 | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | `docs/FDD.md` | Contrato de erro | WEBHOOK_INVALID_URL 400 ou 422 (I-2) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | `docs/FDD.md` | Contrato de erro | WEBHOOK_SECRET_REQUIRED com semântica pendente (P-04) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] WEBHOOK_CUSTOMER_NOT_FOUND 404 | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-05 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] WEBHOOK_ROTATION_IN_PROGRESS 409 | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-06 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] WEBHOOK_DEAD_LETTER_NOT_FOUND 404 | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-07 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED 409 | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-08 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] WEBHOOK_INACTIVE 422 (estado ativo da configuração) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-09 | `docs/FDD.md` | Contrato de erro | WEBHOOK_PAYLOAD_TOO_LARGE para payload > 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-10 | `docs/FDD.md` | Motivo de falha | WEBHOOK_DELIVERY_TIMEOUT após 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-11 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] WEBHOOK_DELIVERY_NETWORK_ERROR (cliente offline) | TRANSCRICAO | [09:14] Larissa |
| FDD-ERR-12 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] WEBHOOK_DELIVERY_HTTP_ERROR | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-14 | `docs/FDD.md` | Motivo de falha | WEBHOOK_RETRIES_EXHAUSTED leva à DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-ERR-15 | `docs/FDD.md` | Fato do código | UNAUTHORIZED/FORBIDDEN vindos do middleware compartilhado | CODIGO | `src/middlewares/auth.middleware.ts` (requireRole) |
| FDD-SEC-03 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Impedir role ADMIN no registro público | CODIGO | `src/modules/auth/auth.schemas.ts` (registerSchema) |
| FDD-SEC-04 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Vínculo usuário-customer ou aceite formal | TRANSCRICAO | [09:37] Sofia |
| FDD-SEC-05 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Cifrar secret em repouso (P-05) | TRANSCRICAO | [09:22] Diego |
| FDD-SEC-07 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Redação de secret e assinatura nos logs | CODIGO | `src/shared/logger/index.ts` (redactPaths) |
| FDD-SEC-09 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Comparação em tempo constante no cliente | TRANSCRICAO | [09:20] Sofia |
| FDD-SEC-10 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] X-Timestamp na assinatura (P-07) | TRANSCRICAO | [09:44] Diego |
| FDD-SEC-11 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] event_id estável inclusive no replay | TRANSCRICAO | [09:25] Diego |
| FDD-SEC-12 | `docs/FDD.md` | Proposta de segurança | [PROPOSTA_PARA_REVISAO] Truncar responseBody armazenado | TRANSCRICAO | [09:34] Marcos |
| FDD-RES-01 | `docs/FDD.md` | Restrição | Sem fallback nesta fase: email adiado; DLQ e replay como recuperação | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-03 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Batch pequeno para conter carga (sem fechamento formal; ADR-001 item 7) | TRANSCRICAO | [09:08] Diego |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Logs estruturados com Pino, sem nada novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] Métricas de pendência, latência, falhas e DLQ (meta de 10 s) | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-03 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] eventId como correlação ponta a ponta | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-04 | `docs/FDD.md` | Proposta técnica | [PROPOSTA] correlationId a partir do X-Request-Id | CODIGO | `src/middlewares/request-logger.middleware.ts` (requestLogger) |
| FDD-INT-01 | `docs/FDD.md` | Integração | changeStatus recebe publishWebhookEvent; contrato do PATCH preservado | CODIGO | `src/modules/orders/order.service.ts` (OrderService.changeStatus) |
| FDD-INT-02 | `docs/FDD.md` | Integração | Máquina de estados inalterada; OrderStatus valida filtros | CODIGO | `src/modules/orders/order.status.ts` (transitions, canTransition) |
| FDD-INT-03 | `docs/FDD.md` | Integração | authenticate nas rotas E1-E7 e requireRole(ADMIN) em E7 | CODIGO | `src/middlewares/auth.middleware.ts` (authenticate, requireRole) |
| FDD-INT-04 | `docs/FDD.md` | Integração | validate usado sem alteração | CODIGO | `src/middlewares/validate.middleware.ts` (validate) |
| FDD-INT-05 | `docs/FDD.md` | Integração | errorMiddleware sem alteração; P2003 cairia em 500 | CODIGO | `src/middlewares/error.middleware.ts` (errorMiddleware) |
| FDD-INT-06 | `docs/FDD.md` | Integração | Novas classes WEBHOOK_ sobre AppError e subclasses | CODIGO | `src/shared/errors/http-errors.ts` (BadRequestError, ConflictError) |
| FDD-INT-07 | `docs/FDD.md` | Integração | redactPaths do logger a revisar | CODIGO | `src/shared/logger/index.ts` (createLogger) |
| FDD-INT-08 | `docs/FDD.md` | Integração | Novos models e migração aditiva | CODIGO | `prisma/schema.prisma` |
| FDD-INT-09 | `docs/FDD.md` | Integração | buildControllers instancia controller de webhooks | CODIGO | `src/app.ts` (buildControllers) |
| FDD-INT-10 | `docs/FDD.md` | Integração | buildApiRouter monta routers de webhooks e admin | CODIGO | `src/routes/index.ts` (buildApiRouter) |
| FDD-INT-11 | `docs/FDD.md` | Integração | Worker usa createPrismaClient; singleton criado na importação | CODIGO | `src/config/database.ts` (createPrismaClient) |
| FDD-INT-12 | `docs/FDD.md` | Integração | env exige JWT_SECRET também no worker | CODIGO | `src/config/env.ts` (envSchema) |
| FDD-INT-13 | `docs/FDD.md` | Integração | src/worker.ts segue o padrão de bootstrap | CODIGO | `src/server.ts` (bootstrap) |
| FDD-INT-14 | `docs/FDD.md` | Integração | Script worker a adicionar; não existe hoje | CODIGO | `package.json` (scripts) |
| FDD-INT-15 | `docs/FDD.md` | Integração | Limpeza das novas tabelas no setup de testes | CODIGO | `tests/setup.ts` (beforeEach) |
| FDD-DEP-01 | `docs/FDD.md` | Dependência | Nenhuma biblioteca nova | CODIGO | `package.json` (dependencies) |
| FDD-DEP-02 | `docs/FDD.md` | Dependência | Prisma 5.22.0 | CODIGO | `package.json` (@prisma/client) |
| FDD-DEP-03 | `docs/FDD.md` | Dependência | MySQL 8.0 | CODIGO | `docker-compose.yml` (image mysql:8.0) |
| FDD-DEP-04 | `docs/FDD.md` | Compatibilidade | PATCH /orders/:id/status preservado, salvo P-10 | CODIGO | `src/modules/orders/order.routes.ts` (buildOrderRouter) |
| FDD-DEP-05 | `docs/FDD.md` | Dependência | Contrato externo documentado no portal | TRANSCRICAO | [09:26] Marcos |
| FDD-DEP-06 | `docs/FDD.md` | Dependência | Novo processo a operar junto da API | TRANSCRICAO | [09:11] Diego |
| FDD-DEP-07 | `docs/FDD.md` | Restrição | Três sprints com revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| FDD-T-OUT-01 | `docs/FDD.md` | Teste especificado | Transição gera linha PENDING com snapshot | TRANSCRICAO | [09:40] Bruno |
| FDD-T-OUT-02 | `docs/FDD.md` | Teste especificado | Falha na outbox desfaz status, histórico e estoque | TRANSCRICAO | [09:40] Bruno |
| FDD-T-OUT-03 | `docs/FDD.md` | Teste especificado | Sem interessado, nenhuma linha | TRANSCRICAO | [09:34] Bruno |
| FDD-T-OUT-04 | `docs/FDD.md` | Teste especificado | Transição inválida: 409 e nenhuma linha | CODIGO | `src/modules/orders/order.status.ts` (canTransition) |
| FDD-T-OUT-05 | `docs/FDD.md` | Teste especificado | Sem estoque: 422 e nenhuma linha | CODIGO | `src/modules/orders/order.service.ts` (debitStock) |
| FDD-T-OUT-06 | `docs/FDD.md` | Teste especificado | Envio usa o snapshot original | TRANSCRICAO | [09:52] Larissa |
| FDD-T-OUT-07 | `docs/FDD.md` | Teste especificado | Dois webhooks: N linhas com o mesmo eventId (bloqueado P-11) | TRANSCRICAO | [09:44] Sofia |
| FDD-T-OUT-08 | `docs/FDD.md` | Teste especificado | Caracterização de mudanças concorrentes (P-17) | CODIGO | `src/modules/orders/order.service.ts` (OrderService.changeStatus) |
| FDD-T-WRK-01 | `docs/FDD.md` | Teste especificado | Worker com PrismaClient próprio encerra no SIGTERM | TRANSCRICAO | [09:30] Bruno |
| FDD-T-WRK-02 | `docs/FDD.md` | Teste especificado | Entrega 200 marca DELIVERED com os cinco headers | TRANSCRICAO | [09:44] Diego |
| FDD-T-WRK-03 | `docs/FDD.md` | Teste especificado | Servidor de teste confere o HMAC do corpo | TRANSCRICAO | [09:22] Sofia |
| FDD-T-WRK-04 | `docs/FDD.md` | Teste especificado | Corpo adulterado falha na verificação | TRANSCRICAO | [09:19] Sofia |
| FDD-T-WRK-05 | `docs/FDD.md` | Teste especificado | Timeout de 10 s reagenda | TRANSCRICAO | [09:42] Diego |
| FDD-T-WRK-06 | `docs/FDD.md` | Teste especificado | 500 seguido de 200: duas entregas registradas | TRANSCRICAO | [09:15] Diego |
| FDD-T-WRK-07 | `docs/FDD.md` | Teste especificado | Retry de 4xx (bloqueado P-20) | TRANSCRICAO | [09:42] Diego |
| FDD-T-WRK-08 | `docs/FDD.md` | Teste especificado | Reenvio após crash com o mesmo X-Event-Id | TRANSCRICAO | [09:24] Diego |
| FDD-T-WRK-09 | `docs/FDD.md` | Teste especificado | Recuperação de PROCESSING órfão (bloqueado P-12) | TRANSCRICAO | [09:08] Diego |
| FDD-T-WRK-10 | `docs/FDD.md` | Teste especificado | Eventos do mesmo pedido em ordem de createdAt | TRANSCRICAO | [09:12] Diego |
| FDD-T-WRK-11 | `docs/FDD.md` | Teste especificado | Ordem com retry (bloqueado P-21) | TRANSCRICAO | [09:12] Diego |
| FDD-T-RTY-01 | `docs/FDD.md` | Teste especificado | Configuração contém 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| FDD-T-RTY-02 | `docs/FDD.md` | Teste especificado | Última tentativa falha leva à DLQ (bloqueado I-1) | TRANSCRICAO | [09:48] Larissa |
| FDD-T-RTY-03 | `docs/FDD.md` | Teste especificado | Penúltima tentativa reagenda (bloqueado I-1) | TRANSCRICAO | [09:48] Larissa |
| FDD-T-RTY-04 | `docs/FDD.md` | Teste especificado | Movimentação para DLQ atômica | TRANSCRICAO | [09:18] Diego |
| FDD-T-RTY-05 | `docs/FDD.md` | Teste especificado | Intervalos efetivamente consumidos (bloqueado I-1) | TRANSCRICAO | [09:17] Diego |
| FDD-T-DLQ-01 | `docs/FDD.md` | Teste especificado | Replay por ADMIN cria PENDING e registra autor | TRANSCRICAO | [09:36] Sofia |
| FDD-T-DLQ-02 | `docs/FDD.md` | Teste especificado | Replay por OPERATOR retorna 403 | TRANSCRICAO | [09:36] Larissa |
| FDD-T-DLQ-03 | `docs/FDD.md` | Teste especificado | Replay mantém X-Event-Id (bloqueado P-08) | TRANSCRICAO | [09:25] Diego |
| FDD-T-DLQ-04 | `docs/FDD.md` | Teste especificado | Segundo replay retorna 409 | TRANSCRICAO | [09:35] Diego |
| FDD-T-DLQ-05 | `docs/FDD.md` | Teste especificado | DLQ inexistente retorna 404 | TRANSCRICAO | [09:35] Diego |
| FDD-T-CRUD-01 | `docs/FDD.md` | Teste especificado | Ciclo E1-E4 com secret só na criação | TRANSCRICAO | [09:33] Bruno |
| FDD-T-CRUD-02 | `docs/FDD.md` | Teste especificado | URL http retorna WEBHOOK_INVALID_URL (bloqueado I-2) | TRANSCRICAO | [09:23] Sofia |
| FDD-T-CRUD-03 | `docs/FDD.md` | Teste especificado | statuses inválido retorna VALIDATION_ERROR | TRANSCRICAO | [09:33] Marcos |
| FDD-T-CRUD-04 | `docs/FDD.md` | Teste especificado | Filtro com PENDING (bloqueado P-09) | CODIGO | `src/modules/orders/order.status.ts` (transitions) |
| FDD-T-CRUD-05 | `docs/FDD.md` | Teste especificado | Customer inexistente retorna 404 | TRANSCRICAO | [09:29] Larissa |
| FDD-T-CRUD-06 | `docs/FDD.md` | Teste especificado | Exclusão de customer sem pedidos não gera 500 | CODIGO | `prisma/migrations/20260519182739_init/migration.sql` (orders_customerId_fkey) |
| FDD-T-SEC-01 | `docs/FDD.md` | Teste especificado | Rotação: anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-T-SEC-02 | `docs/FDD.md` | Teste especificado | Rotas sem token retornam 401 | CODIGO | `src/middlewares/auth.middleware.ts` (authenticate) |
| FDD-T-SEC-03 | `docs/FDD.md` | Teste especificado | Acesso a customer alheio retorna 403 (bloqueado G-12) | CODIGO | `prisma/schema.prisma` (model User) |
| FDD-T-SEC-04 | `docs/FDD.md` | Teste especificado | Registro com ADMIN rejeitado (bloqueado I-3) | CODIGO | `src/modules/auth/auth.schemas.ts` (registerSchema) |
| FDD-T-SEC-06 | `docs/FDD.md` | Teste especificado | Logs sem secret | CODIGO | `src/shared/logger/index.ts` (redactPaths) |
| FDD-T-LIM-01 | `docs/FDD.md` | Teste especificado | Fronteira de 64 KB (bloqueado P-10) | TRANSCRICAO | [09:24] Larissa |
| FDD-T-LIM-02 | `docs/FDD.md` | Teste especificado | Snapshot típico bem abaixo de 64 KB | TRANSCRICAO | [09:43] Diego |
| FDD-T-ERR-01 | `docs/FDD.md` | Teste especificado | Cada código WEBHOOK_ gera status e corpo da matriz | TRANSCRICAO | [09:29] Larissa |
| FDD-ACT-01 | `docs/FDD.md` | Critério de aceite técnico | Nenhum status sem evento quando há interessado; nenhum evento após rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-ACT-02 | `docs/FDD.md` | Critério de aceite técnico | Latência commit-entrega abaixo de 10 s em teste | TRANSCRICAO | [09:02] Marcos |
| FDD-ACT-03 | `docs/FDD.md` | Critério de aceite técnico | Cinco headers e assinatura verificável | TRANSCRICAO | [09:44] Diego |
| FDD-ACT-04 | `docs/FDD.md` | Critério de aceite técnico | Reenvios preservam X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-ACT-05 | `docs/FDD.md` | Critério de aceite técnico | Configuração 1m/5m/30m/2h/12h; limiar segue I-1 | TRANSCRICAO | [09:17] Larissa |
| FDD-ACT-06 | `docs/FDD.md` | Critério de aceite técnico | E7 só por ADMIN com autoria registrada | TRANSCRICAO | [09:36] Larissa |
| FDD-ACT-07 | `docs/FDD.md` | Critério de aceite técnico | Erros com prefixo WEBHOOK_ e formato do errorMiddleware | TRANSCRICAO | [09:29] Larissa |
| FDD-ACT-08 | `docs/FDD.md` | Critério de aceite técnico | Suíte atual de pedidos e auth continua verde | CODIGO | `tests/orders.test.ts` |
| FDD-ACT-09 | `docs/FDD.md` | Critério de aceite técnico | Nenhuma secret em logs ou respostas fora de E1/E5 | TRANSCRICAO | [09:22] Diego |
| FDD-ACT-10 | `docs/FDD.md` | Critério de aceite técnico | Revisão da Sofia concluída antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-P-01 | `docs/FDD.md` | Pendência | I-1 contagem das tentativas | TRANSCRICAO | [09:48] Larissa |
| FDD-P-02 | `docs/FDD.md` | Pendência | I-2 VALIDATION_ERROR x WEBHOOK_INVALID_URL | CODIGO | `src/middlewares/validate.middleware.ts` (validate) |
| FDD-P-03 | `docs/FDD.md` | Pendência | I-3 registro público com ADMIN | CODIGO | `src/modules/auth/auth.schemas.ts` (registerSchema) |
| FDD-P-04 | `docs/FDD.md` | Pendência | Semântica de WEBHOOK_SECRET_REQUIRED com secret gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| FDD-P-05 | `docs/FDD.md` | Pendência | Armazenamento da secret em repouso | TRANSCRICAO | [09:22] Diego |
| FDD-P-06 | `docs/FDD.md` | Pendência | Qual secret assina durante as 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-P-07 | `docs/FDD.md` | Pendência | X-Timestamp no HMAC | TRANSCRICAO | [09:44] Diego |
| FDD-P-08 | `docs/FDD.md` | Pendência | event_id após replay | TRANSCRICAO | [09:18] Diego |
| FDD-P-09 | `docs/FDD.md` | Pendência | Evento na criação do pedido | CODIGO | `src/modules/orders/order.service.ts` (OrderService.create) |
| FDD-P-10 | `docs/FDD.md` | Pendência | Onde aplicar o limite de 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-P-11 | `docs/FDD.md` | Pendência | Fan-out para vários endpoints | TRANSCRICAO | [09:44] Sofia |
| FDD-P-12 | `docs/FDD.md` | Pendência | Recuperação de PROCESSING órfão | TRANSCRICAO | [09:08] Diego |
| FDD-P-13 | `docs/FDD.md` | Pendência | customer_id no path ou no body | TRANSCRICAO | [09:32] Larissa |
| FDD-P-14 | `docs/FDD.md` | Pendência | Nome do arquivo do processador | TRANSCRICAO | [09:28] Bruno |
| FDD-P-15 | `docs/FDD.md` | Pendência | Papel do status FAILED na outbox | TRANSCRICAO | [09:08] Diego |
| FDD-P-16 | `docs/FDD.md` | Pendência | Eventos pendentes de webhook removido ou inativo | TRANSCRICAO | [09:33] Bruno |
| FDD-P-17 | `docs/FDD.md` | Pendência | Concorrência em changeStatus sem lock | CODIGO | `src/modules/orders/order.service.ts` (OrderService.changeStatus) |
| FDD-P-18 | `docs/FDD.md` | Pendência | Campos básicos além de total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-P-19 | `docs/FDD.md` | Pendência | Tamanho do batch | TRANSCRICAO | [09:08] Diego |
| FDD-P-20 | `docs/FDD.md` | Pendência | Critério de sucesso e retry de 4xx | TRANSCRICAO | [09:42] Diego |
| FDD-P-21 | `docs/FDD.md` | Pendência | Ordenação com retries | TRANSCRICAO | [09:12] Diego |
| FDD-P-22 | `docs/FDD.md` | Pendência | G-12 autorização por objeto | CODIGO | `prisma/schema.prisma` (model User) |
| FDD-P-24 | `docs/FDD.md` | Pendência | Segunda rotação dentro das 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-P-25 | `docs/FDD.md` | Pendência | Unidade dos últimos 100 do histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-RR-01 | `docs/FDD.md` | Risco | Cliente lento atrasa os demais | TRANSCRICAO | [09:42] Diego |
| FDD-RR-02 | `docs/FDD.md` | Risco | Replay indevido por ADMIN autoatribuível (I-3); exposição G-12 em FDD-P-22 | CODIGO | `src/modules/auth/auth.routes.ts` (buildAuthRouter) |
| FDD-RR-03 | `docs/FDD.md` | Risco | Crescimento da outbox sem arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-RR-04 | `docs/FDD.md` | Risco | Transação de changeStatus mais longa | CODIGO | `src/modules/orders/order.service.ts` (OrderService.changeStatus) |
| FDD-RR-05 | `docs/FDD.md` | Risco | Cliente sem deduplicação | TRANSCRICAO | [09:26] Marcos |

## 4. Itens sem rastreabilidade válida

Estes itens existem nos documentos e entram no denominador da cobertura, mas **não têm fonte primária** na transcrição nem no código. Não recebem localização inventada.

| ID | Documento | Item | Motivo |
|---|---|---|---|
| PRD-OBJ-04 | `docs/PRD.md` | [PROPOSTA] Indicadores de avisos retidos e adesão dos três clientes | Nenhuma fala trata de indicadores além da meta de 10 s. |
| PRD-VAL-01 | `docs/PRD.md` | [PROPOSTA] Validação com os três clientes piloto | O próprio PRD registra que não foi discutida na reunião. |
| RFC-NEXT-02 | `docs/RFC.md` | [PROPOSTA] Responsáveis por I-1, contratos, segurança e escopo | [09:46] Sofia sustenta só a revisão de segurança; a distribuição dos demais responsáveis é sugestão do documento. |
| FDD-DEP-08 | `docs/FDD.md` | [PROPOSTA] schema_version opcional no payload | O FDD registra que não foi discutido; sem fato de código motivador. |
| FDD-DEP-09 | `docs/FDD.md` | [PROPOSTA] Migration aplicada antes da nova versão da API | Prática de deploy sem fala ou fato de código específico. |
| RFC-RSK-04 | `docs/RFC.md` | SSRF em URLs HTTPS | Tema não discutido na reunião e sem fato de código que o sustente; análise de segurança externa (OWASP). |
| ADR-004-P06 | `docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md` | [PROPOSTA_PARA_REVISAO] Restrições de destino contra SSRF | Mesmo motivo de RFC-RSK-04; o próprio ADR registra que não foi discutido. |
| FDD-NG-03 | `docs/FDD.md` | Alta disponibilidade do worker não é objetivo desta fase | Afirmação de escopo sem fala correspondente; [09:13] Diego trata só de vários workers. |
| FDD-ERR-13 | `docs/FDD.md` | [PROPOSTA] WEBHOOK_DELIVERY_REDIRECT_BLOCKED | Deriva da proposta de SSRF, sem fonte primária. |
| FDD-SEC-08 | `docs/FDD.md` | [PROPOSTA_PARA_REVISAO] Bloqueio de IPs privados, redirect manual, DNS no envio | SSRF: sem fonte primária. |
| FDD-RES-02 | `docs/FDD.md` | [PROPOSTA] Banco indisponível no worker: registrar erro e tentar no próximo tick | Prática de resiliência sem fala ou fato de código específico. |
| FDD-T-WRK-12 | `docs/FDD.md` | Teste de banco indisponível durante o tick | Verifica FDD-RES-02, que não tem fonte primária. |
| FDD-T-WRK-13 | `docs/FDD.md` | Teste de resposta 3xx não seguida | Verifica a proposta de SSRF. |
| FDD-T-SEC-05 | `docs/FDD.md` | Teste de URL que resolve para IP privado | Verifica a proposta de SSRF. |
| FDD-P-23 | `docs/FDD.md` | Pendência SSRF | Registrada como pendência para a Sofia, mas sem fonte primária. |

Recomendação: manter esses itens rotulados como análise ou proposta e submetê-los à revisão de segurança da Sofia ([09:46] Sofia). A outra opção é removê-los dos documentos, decisão que cabe ao autor. Nenhum deles aparece como decisão da reunião nos documentos auditados.

## 5. Divergências e observações

**Divergência encontrada e resolvida:**

- **DIV-01: rótulo da fala de [09:08] Diego** (índices em `status`/`created_at` e "batch pequeno"). DIV-01 foi anteriormente identificada como D-01 no processo de auditoria; o ID mudou para não colidir com a dependência D-01 do PRD.
  - O `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` (Decisão, item 7) a classifica como `PROPOSTA_TÉCNICA`, sem fechamento formal.
  - O `docs/FDD.md` §5 a lista entre os campos `ACORDADO` ("status e `created_at` da outbox, em [09:08] Diego").
  - O ADR-002 (item 7) e o FDD P-19 confirmam que o tamanho do batch não foi definido.
  - Neste Tracker, as linhas correspondentes (ADR-001-D07, FDD-DM-02, FDD-RES-03) seguem a classificação do ADR.
  - **Status: RESOLVIDA (2026-11-16)**, com autorização específica.
    - O `docs/FDD.md` §5 deixou de listar "status e `created_at` da outbox" entre os itens `ACORDADO`.
    - O FDD passou a registrar o índice e a leitura em batch pequeno como `PROPOSTA_TÉCNICA`, alinhados ao ADR-001 item 7.
    - Evidência: em [09:08] Diego descreve a tabela; o fechamento seguinte, [09:08] Larissa ("Tá decidido então: outbox em MySQL"), trata da outbox em geral, sem marca específica para índices ou batch.
    - Nenhum valor técnico, contrato ou outra classificação foi alterado, e nenhuma linha da matriz mudou: as linhas ADR-001-D07, FDD-DM-02 e FDD-RES-03 já seguiam o ADR.

**Inconsistências da auditoria integral (2026-11-16):** a auditoria cruzada de entrega encontrou oito inconsistências menores entre documentos (m-1 a m-8), todas corrigidas com autorização específica, sem alterar valores técnicos, decisões ou a matriz:

| ID | Inconsistência | Correção |
|---|---|---|
| m-1 | Rate limiting de saída listado como fora de escopo no FDD §3 | Registrado no FDD como "em avaliação, não excluído" ([09:39] Larissa), alinhado ao PRD §6 e ao RFC §5 |
| m-2 | ADR-001 dizia que o limite de 64 KB foi "decidido" | Classificado como `ACORDADO` (requisito não funcional), conforme [09:24] Larissa |
| m-3 | PRD-FR-02 tratava "customer não deduzido do JWT" como `ACORDADO` | Classificado como `DECIDIDO` ([09:32] Larissa; [09:33] Bruno "Anotado"), alinhado ao ADR-006; G-12 preservado |
| m-4 | Meta de 10 s rotulada `DECIDIDO` no PRD OBJ-01 | Rótulo removido: é requisito de negócio dos clientes ([09:02] Marcos), sem fechamento formal |
| m-5 | Numeração do workflow no README diferente das "Etapas" citadas nos prompts | Etapa correspondente indicada em cada atividade |
| m-6 | ID "D-01" usado para a dependência do PRD e para a divergência | Divergência renomeada para DIV-01 no README e neste Tracker |
| m-7 | README citava entre aspas um texto que não é literal do FDD; ADR-001 prometia confirmar o timeout "durante o FDD" | Aspas removidas; ADR-001 passou a dizer "antes da implementação" |
| m-8 | PRD dizia que a rastreabilidade "será consolidada" | PRD afirma que a rastreabilidade está consolidada neste Tracker |

**Reauditoria localizada (`traceability-auditor`, 2026-11-16): PASS.**
- Os oito itens foram conferidos com a transcrição e estão resolvidos, sem regressão e sem alteração de valores técnicos ou decisões.
- Depois dessas correções, não há outras contradições de significado entre PRD, RFC, FDD e ADRs.
- Alcance: a reauditoria cobriu os temas de m-1 a m-8 e as classificações relacionadas, não uma releitura integral do pacote.

Registram-se as observações abaixo:

1. **I-1 está na própria fonte.** "5 tentativas" ([09:17] Larissa, [09:48] Larissa) e os cinco intervalos com janela de "quase 15 horas" ([09:17] Diego) admitem duas leituras. Os quatro documentos preservam os valores e tratam a contagem como pendente (ADR-003, RFC §6, FDD P-01, PRD R-03).
2. **Status `FAILED` da outbox.** Foi citado em [09:08] Diego e mantido no modelo proposto do FDD, mas seu papel com a DLQ separada continua em aberto (ADR-003, FDD P-15).
3. **Listas de revisores com propósitos diferentes.** O RFC lista os cinco participantes como revisores do documento, conforme a rubrica. O FDD distingue a revisão do design (Larissa, Bruno e Diego, [09:50] Larissa) da revisão de segurança do código (Sofia, [09:46] Sofia). Não há contradição.
4. **Inconsistência interna da transcrição.** Em [09:27] Larissa diz "Estamos a uma hora de reunião", mas a reunião começou às 09:00 e dura ~55 min pelo cabeçalho. Não afeta nenhuma decisão.
5. **Decisões após o resumo.** UUID ([09:51] Larissa) e snapshot ([09:52] Bruno) foram fechados depois de Marcos e Sofia saírem ([09:50]). Os documentos classificam o UUID como detalhe acordado e o snapshot como decisão.
6. **Gap entre transcrição e código.** A transcrição diz que a transação de mudança de status "decrementa stock_quantity" ([09:04] Bruno), mas o código só debita em PENDING→PAID (`src/modules/orders/order.status.ts`, `shouldDebitStock`). O ADR-001 registra a nuance.

## 6. Validação executada

- Script local (fora do repositório) que lê `TRANSCRICAO.md` e confere:
  - unicidade dos 499 IDs;
  - existência dos 9 documentos rastreados;
  - para cada linha `TRANSCRICAO`, o formato `[hh:mm] Nome` e a fala daquele participante naquele minuto;
  - para cada linha `CODIGO`, a existência do arquivo.
- **Reprodutibilidade:** o script de geração e validação não foi versionado, porque esta etapa autoriza alterar apenas `docs/TRACKER.md`. Ele pode ser incluído no repositório mediante autorização.
- **Revisão semântica independente** (agente `traceability-auditor`, 2026-11-16):
  - **1ª rodada: NEEDS_REVIEW.** Foram verificadas todas as 90 linhas `CODIGO` e as 385 linhas `TRANSCRICAO` da versão anterior. Achados: 2 MAJOR e 7 grupos MINOR (tipos inflados, âncoras forçadas, duplicatas no mesmo documento, cerca de 20 itens não inventariados). Nenhuma fonte inventada.
  - **Correções aplicadas nesta versão:**
    - reclassificação de tipos;
    - 3 itens movidos para a §4;
    - 7 duplicatas removidas;
    - 21 itens incluídos no inventário;
    - divergência DIV-01 registrada.
  - **Reauditoria das linhas alteradas:** PASS (2026-11-16). Todos os achados resolvidos, sem regressão.
- **Encerramento da DIV-01** (2026-11-16): o rótulo do `docs/FDD.md` §5 foi alinhado ao ADR-001 item 7. A matriz, os denominadores e os percentuais não mudaram.
- Nenhum teste, migration ou seed foi executado.
