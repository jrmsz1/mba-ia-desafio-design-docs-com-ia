# PRD — Webhooks de Notificação de Pedidos

| Metadado | Valor |
|---|---|
| **Status** | **Draft — para revisão** (não autoriza implementação nem deploy) |
| **Data do documento** | 2026-11-16 |
| **Autor** | [@jrmsz1](https://github.com/jrmsz1) |
| **Responsável de produto na reunião** | Marcos (PM) |
| **Documentos relacionados** | [RFC](RFC.md) (proposta arquitetural), [FDD](FDD.md) (especificação de implementação), [ADRs](adrs/README.md) (decisões) |

**Legenda de estado:**
- `DECIDIDO`: fechado na reunião.
- `ACORDADO`: proposto e aceito sem objeção.
- `PENDENTE`: decisão ainda necessária.
- `PROPOSTA_PARA_REVISAO`: sugestão deste documento, não aprovada.

Citações no formato `[hh:mm] Nome` referem-se a [`TRANSCRICAO.md`](../TRANSCRICAO.md).

---

## 1. Resumo e contexto da feature

O OMS passará a **avisar automaticamente os clientes B2B quando o status de um pedido deles mudar**, por meio de chamadas HTTPS de saída (webhooks) para um endereço cadastrado pelo cliente.

- Cada aviso é **assinado**, para que o cliente confirme a origem e a integridade.
- Cada aviso traz um **identificador único**, para que o cliente descarte repetições.
- Avisos que falham são **reenviados automaticamente** e, se ainda assim não forem entregues, ficam **guardados para reprocessamento manual** por um administrador.

Hoje o OMS não tem nenhum mecanismo de notificação externa. É fato do código, verificado por varredura de `src/`, sem chamadas HTTP de saída, filas ou eventos (ver [FDD §1](FDD.md#1-contexto-e-motivação-técnica)). A decisão técnica foi tomada em reunião entre Larissa (Tech Lead), Marcos (PM), Bruno (Pedidos), Diego (Plataforma) e Sofia (Segurança). A estimativa é de **três sprints**, incluindo a revisão de segurança ([09:47] Larissa).

## 2. Problema e motivação

| Aspecto | Descrição | Fonte |
|---|---|---|
| Quem pediu | Pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo | [09:00] Marcos |
| Dor atual | Os clientes consultam a lista de pedidos de tempos em tempos para descobrir mudanças, o que deixa a integração "lenta e cara pra eles" | [09:00] Marcos |
| O que querem | Ser notificados em "tempo real" quando o status muda, sem precisar ficar atualizando manualmente | [09:00] Marcos, [09:02] Marcos |
| Risco comercial | A Atlas sugeriu que pode migrar para um concorrente se a entrega não ocorrer até o fim do trimestre | [09:00] Marcos |
| Prazo pedido | A Atlas quer a funcionalidade para o fim de novembro | [09:45] Marcos |

## 3. Público-alvo e cenários de uso

| Público | Papel | Fonte |
|---|---|---|
| Clientes B2B integrados | Recebem os avisos em seus sistemas; validam a assinatura e descartam duplicatas | [09:00] Marcos, [09:25] Diego |
| Usuários do OMS que representam o cliente | Cadastram e gerenciam webhooks pela API do OMS, autenticados com o JWT do sistema | [09:32] Marcos |
| Administradores do OMS (role ADMIN) | Reprocessam manualmente avisos que esgotaram as tentativas | [09:36] Sofia, [09:36] Larissa |
| Time de produto e portal do desenvolvedor | Documenta a integração e a possibilidade de avisos repetidos | [09:26] Marcos, [09:40] Marcos |

**Cenários**

- **C-01:** Um cliente cadastra seu endereço e escolhe receber só os status SHIPPED e DELIVERED. Quando o pedido é enviado, recebe o aviso em poucos segundos ([09:33] Marcos para o filtro; [09:02] Marcos para a meta de tempo).
- **C-02:** O sistema do cliente fica fora do ar por duas horas em manutenção planejada. Os avisos são reenviados automaticamente e chegam quando ele volta ([09:16] Diego).
- **C-03:** O cliente recebe o mesmo aviso duas vezes e descarta o repetido pelo identificador do evento ([09:25] Diego).
- **C-04:** Um cliente suspeita que sua secret vazou, pede uma nova e tem 24 horas para migrar seus sistemas ([09:21] Sofia, [09:22] Diego). Como os avisos são assinados durante essa janela ainda depende da FDD P-06.
- **C-05:** Um aviso esgota as tentativas. Um administrador investiga e solicita o reenvio, e fica registrado quem fez isso ([09:18] Diego, [09:36] Sofia).
- **C-06:** O cliente consulta os últimos envios feitos para ele, com sucesso ou falha, conteúdo, resposta e tempo de resposta, para diagnosticar a integração ([09:34] Marcos).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica e meta | Estado | Fonte |
|---|---|---|---|---|
| OBJ-01 | Substituir o polling dos clientes por avisos quase imediatos | **Aviso entregue em menos de 10 segundos** após a mudança de status, em operação normal (cliente disponível) | Requisito de negócio dos clientes, relatado pelo PM, sem fechamento formal específico. A equipe considerou o desenho suficiente para ele ([09:09] Diego, [09:10] Marcos). | [09:02] Marcos |
| OBJ-02 | Atender o pedido dos três clientes no prazo | Funcionalidade disponível até o fim de novembro, estimada em três sprints | Prazo informado pelo cliente; estimativa da Tech Lead; confirmação com a Atlas pendente ([09:47] Marcos) | [09:45] Marcos, [09:47] Larissa |
| OBJ-03 | Não perder avisos por indisponibilidade temporária do cliente | Avisos não entregues são reenviados automaticamente e, se ainda falharem, ficam disponíveis para reprocessamento manual | `DECIDIDO` | [09:17] Larissa, [09:19] Larissa |

**Como a meta do OBJ-01 será verificada**

| Item | Situação |
|---|---|
| Meta estabelecida | Menos de 10 segundos ([09:02] Marcos) |
| Estratégia de medição | `PROPOSTA_PARA_REVISAO`: medir o tempo entre a mudança de status e a entrega bem-sucedida, usando o registro de entregas, em ambiente de teste e depois em produção ([FDD §11](FDD.md#11-observabilidade), AC-T-02) |
| Resultado comprovado | **Ainda não existe.** A funcionalidade não foi implementada. |

**Indicadores adicionais** (`PROPOSTA_PARA_REVISAO`, não são compromissos):
- quantidade de avisos que chegam à fila de falhas permanentes;
- adesão dos três clientes solicitantes.

Nenhum percentual, SLA, volume ou disponibilidade foi aprovado na reunião, e este PRD não fixa nenhum.

## 5. Escopo incluído

- Aviso automático de mudança de status de pedido para clientes B2B (PRD-FR-01).
- Cadastro, edição, listagem e remoção de webhooks por customer, com escolha dos status de interesse (PRD-FR-02 a 07).
- Secret gerada pela plataforma, assinatura de cada aviso e rotação com 24 horas de convivência (PRD-FR-03, 09 e 10).
- Identificador único por evento para deduplicação (PRD-FR-11).
- Reenvio automático, retenção de falhas permanentes e reprocessamento manual auditado (PRD-FR-12 a 14).
- Histórico das últimas entregas (PRD-FR-08).
- Documentação no portal do desenvolvedor sobre a integração e os avisos repetidos ([09:26] Marcos, [09:40] Marcos).

## 6. Fora de escopo

| Item | Classificação | Fonte |
|---|---|---|
| Email ao cliente quando o webhook falha repetidamente | **Adiado** para a próxima fase, após medir o impacto | [09:37] Larissa, [09:38] Marcos |
| Dashboard visual para o cliente ver seus webhooks | **Fora do escopo**: projeto separado do time de frontend | [09:40] Larissa |
| Webhooks de entrada (cliente enviando para o OMS) | **Fora do escopo**: o fluxo é só de saída | [09:02] Marcos, [09:03] Sofia |
| Vários workers em paralelo (escala) | **Adiado**: "problema do futuro" | [09:13] Diego |
| Arquivamento de avisos já entregues (~30 dias) | **Fora desta feature** | [09:08] Diego |
| Itens do pedido no aviso | **Excluído do conteúdo**: o cliente consulta o pedido se quiser detalhes | [09:43] Diego |
| Garantia de entrega exatamente uma vez (exactly-once) | **Alternativa rejeitada**: exigiria coordenação dos dois lados | [09:25] Diego, [09:26] Larissa |
| Ordenação global de avisos | **Não prometida**: só por pedido, com um único worker | [09:13] Larissa, [09:14] Marcos |

**Em avaliação, não excluído:** o **limite de ritmo de envio** (rate limiting) a cada cliente. A decisão foi "observar e decidir depois" ([09:38] Diego, [09:39] Larissa).

## 7. Requisitos funcionais

Os critérios de aceitação de cada requisito estão na §12.

| ID | Requisito | Fonte | Estado |
|---|---|---|---|
| PRD-FR-01 | Quando o status de um pedido muda, o OMS envia um aviso aos webhooks do customer que escutam aquele status | [09:00] Marcos, [09:40] Bruno | `DECIDIDO` ([09:48] Larissa) |
| PRD-FR-02 | Um usuário autenticado cadastra um webhook informando o endereço, os status de interesse e o customer. O customer é informado explicitamente, não deduzido do login. | [09:31] Marcos, [09:32] Larissa, [09:33] Bruno | Cadastro e campos: `ACORDADO`. Customer não deduzido do JWT: `DECIDIDO` ([09:32] Larissa "Não vem do JWT"; [09:33] Bruno "Anotado"; ADR-006 item 8). Customer no caminho ou no corpo: `PENDENTE` ([09:32] Larissa). Isso **não** resolve a autorização por objeto entre usuário e customer, que continua em aberto (R-02, G-12). |
| PRD-FR-03 | A secret do webhook é **gerada pela plataforma** e devolvida ao usuário no cadastro | [09:31] Marcos | `ACORDADO` |
| PRD-FR-04 | O usuário edita um webhook existente | [09:33] Bruno | `ACORDADO` |
| PRD-FR-05 | O usuário remove um webhook | [09:33] Bruno | `ACORDADO`; destino de avisos ainda pendentes do webhook removido `PENDENTE` (FDD P-16) |
| PRD-FR-06 | O usuário lista os webhooks de um customer | [09:33] Bruno | `ACORDADO` |
| PRD-FR-07 | Cada webhook define a lista de status que deseja receber; status fora da lista não geram aviso para ele | [09:33] Marcos, [09:34] Bruno | `ACORDADO` |
| PRD-FR-08 | O cliente consulta o histórico das **últimas 100** entregas: sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta | [09:34] Marcos | `ACORDADO`; unidade dos "100" (tentativas ou eventos) `PENDENTE` (FDD P-25) |
| PRD-FR-09 | O cliente solicita uma nova secret pela API; a anterior continua válida por **24 horas** e depois deixa de valer | [09:21] Sofia, [09:22] Sofia | `DECIDIDO` |
| PRD-FR-10 | Cada aviso é **assinado com HMAC-SHA256** usando uma secret **exclusiva do webhook**, para o cliente verificar origem e integridade | [09:19] Sofia, [09:20] Sofia, [09:21] Sofia | `DECIDIDO` ([09:22] Sofia) |
| PRD-FR-11 | Cada aviso traz um **identificador único do evento** (`X-Event-Id`) para que o cliente descarte repetições | [09:25] Diego | `DECIDIDO` ([09:26] Larissa) |
| PRD-FR-12 | Avisos não entregues são **reenviados automaticamente**: 5 tentativas, com intervalos de 1m, 5m, 30m, 2h e 12h | [09:15] Diego, [09:17] Diego | `DECIDIDO` ([09:17] Larissa). A contagem exata está `PENDENTE` (I-1, §11). |
| PRD-FR-13 | Avisos que esgotam as tentativas ficam guardados com o motivo da falha e podem ser **reprocessados manualmente apenas por administradores** | [09:18] Diego, [09:19] Larissa, [09:36] Sofia | `DECIDIDO` ([09:36] Larissa) |
| PRD-FR-14 | Todo reprocessamento manual **registra quem o executou**, para auditoria | [09:36] Sofia | `ACORDADO` |

**Total:** 14 requisitos funcionais.

Nenhum deles vem de proposta técnica do FDD. Detalhes como caminhos de API, formato da resposta e nomes de erro estão no [FDD §7](FDD.md#7-contratos-públicos-http) e não fazem parte deste PRD.

## 8. Requisitos não funcionais

| ID | Requisito | Fonte | Estado |
|---|---|---|---|
| PRD-NFR-01 | Entrega do aviso em **menos de 10 segundos** em operação normal | [09:02] Marcos | Meta (OBJ-01) |
| PRD-NFR-02 | Uma mudança de status **nunca** fica registrada sem os avisos devidos aos webhooks que escutam aquele status, e um aviso nunca existe sem a mudança | [09:06] Diego, [09:40] Bruno, [09:41] Diego | `DECIDIDO` ([09:08] Larissa, [09:48] Larissa) |
| PRD-NFR-03 | A lentidão ou indisponibilidade de um cliente **não pode** atrasar nem bloquear a mudança de status de pedidos | [09:04] Bruno, [09:06] Diego | `DECIDIDO` |
| PRD-NFR-04 | Os avisos são enviados **somente para endereços HTTPS**; endereços HTTP são recusados no cadastro | [09:23] Sofia | `ACORDADO` |
| PRD-NFR-05 | Avisos maiores que **64 KB** não são enviados nem truncados: geram erro | [09:23] Sofia, [09:24] Diego, [09:24] Larissa | `ACORDADO`. A etapa em que o erro ocorre está `PENDENTE` (FDD P-10). |
| PRD-NFR-06 | Um cliente que não responde em **10 segundos** é tratado como falha e entra no reenvio | [09:42] Diego, [09:42] Sofia | `ACORDADO` |
| PRD-NFR-07 | Garantia de entrega **pelo menos uma vez**: o cliente pode receber o mesmo aviso mais de uma vez | [09:24] Diego, [09:26] Larissa | `DECIDIDO` |
| PRD-NFR-08 | O aviso é **enxuto** (dados do pedido e da mudança, sem itens) e reflete o estado **no momento da mudança** | [09:43] Diego, [09:44] Bruno, [09:52] Larissa | `ACORDADO` (conteúdo); `DECIDIDO` (momento: [09:52] Bruno) |
| PRD-NFR-09 | A ordem dos avisos é garantida apenas **por pedido** e enquanto houver um único processador | [09:12] Diego, [09:13] Larissa | Limitação aceita. O impacto dos reenvios na ordem está `PENDENTE` (FDD P-21). |
| PRD-NFR-10 | Secret **exclusiva por webhook**: o vazamento de uma não compromete as demais | [09:21] Sofia | `DECIDIDO` |
| PRD-NFR-11 | Sem nova infraestrutura de mensageria; reaproveitar banco, logs e padrões já existentes no OMS | [09:07] Diego, [09:30] Larissa | `DECIDIDO` |
| PRD-NFR-12 | Revisão de segurança da Sofia, de **pelo menos dois dias úteis**, antes do deploy | [09:46] Sofia, [09:47] Larissa | `ACORDADO` |

**Total:** 12 requisitos não funcionais.

## 9. Decisões e trade-offs principais

| Decisão | O que se ganha | O que se aceita | ADR |
|---|---|---|---|
| Registrar o aviso junto com a mudança de status e enviá-lo depois, em vez de chamar o cliente na hora | Nenhum aviso se perde e clientes lentos não afetam o OMS ([09:04] Bruno) | O aviso chega com algum atraso | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Usar o banco já existente, sem Redis ou fila nova | Menos infraestrutura para um time pequeno ([09:07] Diego) | Verificação periódica do banco | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Processador separado, verificando a cada 2 segundos | Atende a meta de menos de 10 s ([09:10] Marcos) e não cai quando a API reinicia ([09:11] Diego) | Até ~2 s de espera; um único processador | [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) |
| 5 tentativas com intervalos crescentes (até ~15 horas segundo [09:17] Diego; a janela final depende da I-1, R-03), depois retenção para reprocessamento manual | Cobre indisponibilidades longas ([09:16] Diego), e Marcos considera aceitável ([09:17] Marcos) | Avisos podem chegar horas depois; o reprocessamento final é manual | [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) |
| Assinatura HMAC-SHA256 com secret por webhook e rotação de 24 h | O cliente confirma a origem; o vazamento fica isolado ([09:21] Sofia) | O cliente precisa implementar a verificação | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md) |
| Entrega "pelo menos uma vez" com identificador do evento | Solução simples e padrão de mercado ([09:25] Diego) | Duplicatas possíveis; a deduplicação fica com o cliente ([09:25] Sofia) | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Seguir os padrões existentes do OMS | Consistência e menor custo de manutenção ([09:30] Larissa) | Herda as fragilidades de acesso existentes (riscos R-01 e R-02) | [ADR-006](adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md) |

## 10. Dependências

| ID | Dependência | Responsável | Fonte |
|---|---|---|---|
| D-01 | Revisão de segurança antes do deploy, com no mínimo dois dias úteis, cobrindo HMAC e geração de secret | Sofia | [09:46] Sofia, [09:49] Sofia |
| D-02 | Sessão de revisão do design antes de começar a codificação | Larissa, Bruno, Diego | [09:50] Larissa |
| D-03 | Documentação no portal do desenvolvedor: como integrar via API e a semântica de avisos repetidos (deduplicação). Incluir também a verificação da assinatura, necessidade que vem de [09:20] Sofia, é `PROPOSTA_PARA_REVISAO`. | Marcos | [09:26] Marcos, [09:40] Marcos |
| D-04 | Confirmação do prazo com a Atlas e atualização dos clientes | Marcos | [09:47] Marcos, [09:49] Marcos |
| D-05 | Os clientes precisam implementar a verificação da assinatura e a deduplicação por identificador | Clientes B2B | [09:20] Sofia, [09:25] Diego |
| D-06 | Decisões pendentes listadas na §11 e no [FDD §15](FDD.md#15-riscos-pendências-e-decisões-ainda-necessárias) | Ver cada item | — |

## 11. Riscos e mitigação

As probabilidades e impactos são avaliação qualitativa deste documento. Nenhum controle de segurança citado está implementado. As mitigações são `PROPOSTA_PARA_REVISAO` e dependem da revisão da Sofia quando envolvem segurança ([09:46] Sofia).

| ID | Risco | Prob. | Justificativa da probabilidade | Impacto | Mitigação proposta | Origem | Quem decide |
|---|---|---|---|---|---|---|---|
| R-01 (I-3) | **Cadastro público aceita a role ADMIN.** Qualquer pessoa pode se tornar administrador e reprocessar avisos, anulando a restrição do replay. | Média | A rota de cadastro é pública e aceita `ADMIN` hoje (fato do código); depende apenas de alguém tentar | Alto: reenvio indevido de dados de pedidos e auditoria comprometida | Impedir a escolha de ADMIN no cadastro público (correção fora desta feature); revisar administradores existentes antes do deploy | Código: `src/modules/auth/auth.schemas.ts`, `src/modules/auth/auth.routes.ts`; [ADR-006](adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md) | Sofia (revisão), Larissa (escopo) |
| R-02 (G-12) | **Sem isolamento entre clientes.** Como o cadastro é liberado a qualquer usuário autenticado "por enquanto" ([09:37] Sofia) e não há vínculo entre usuário e customer, alguém pode cadastrar o próprio endereço no customer de outra empresa e receber os avisos dela. | Média | O cadastro de usuários é público e não há verificação por objeto no modelo atual (fato do código) | Alto: exposição de dados de pedidos de outro cliente | Vincular usuários a customers e verificar a permissão em cada operação, **ou** aceite formal e temporário do risco pela Segurança | Código: `prisma/schema.prisma` (model `User` sem relação com `Customer`); [09:32] Marcos, [09:37] Sofia | Sofia |
| R-03 (I-1) | **Contagem das tentativas ambígua.** "5 tentativas" com 5 intervalos pode significar 5 envios (janela de ~2h36) ou 1 envio mais 5 reenvios (janela de ~14h36, "quase 15 horas"). | Alta | A ambiguidade está na própria fonte e não foi resolvida | Médio: muda a janela comunicada aos clientes e quando um aviso vai para reprocessamento manual | Larissa e Diego esclarecem antes de implementar; consultar Marcos se a janela mudar | [09:17] Diego, [09:17] Larissa, [09:48] Larissa; [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) | Larissa, Diego |
| R-04 | **Avisos duplicados** processados duas vezes por clientes que não deduplicam | Média | Duplicatas são esperadas pela garantia escolhida ([09:24] Diego); depende da implementação de cada cliente | Médio: efeito repetido no sistema do cliente | Documentação destacada no portal ([09:26] Marcos); identificador estável por evento, inclusive no reprocessamento (FDD P-08) | [09:25] Sofia, [09:26] Marcos | Marcos, Diego |
| R-05 | **Atraso causado por cliente lento.** Com um único processador e espera de até 10 s por cliente, um cliente lento pode atrasar os avisos dos demais e comprometer a meta de menos de 10 s. | Média | Inferência do desenho aprovado (processador único + timeout de 10 s) | Alto para o OBJ-01 | Avaliar processamento com concorrência limitada ou lotes menores, preservando a ordem por pedido (PRD-NFR-09); exige decisão de Diego e Larissa. Medir o atraso do aviso pendente mais antigo. ([FDD §10–11](FDD.md#10-estratégias-de-resiliência)) | [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) | Diego |
| R-06 | **Falha prolongada de entrega:** avisos esgotam as tentativas e ficam parados até ação manual | Baixa na leitura B da I-1 (~14h36 de janela); **Média** na leitura A (~2h36), já que houve indisponibilidade de 2 horas ([09:16] Diego) | Depende da janela efetiva de reenvio (R-03) | Médio: o cliente fica sem a informação até o reprocessamento | Reprocessamento manual por ADMIN; indicador de avisos retidos (`PROPOSTA_PARA_REVISAO`); email adiado para a próxima fase | [09:18] Diego, [09:37] Larissa | Larissa |
| R-07 | **Prazo:** três sprints contra o pedido de fim de novembro, com revisão de segurança obrigatória | Média | Prazo comercial fixado pelo cliente e estimativa ainda não confirmada com a Atlas | Alto comercialmente (risco de churn da Atlas, [09:00] Marcos) | Confirmar o prazo com a Atlas ([09:47] Marcos); reservar a revisão da Sofia no planejamento ([09:47] Larissa) | [09:45] Marcos, [09:46] Larissa | Marcos, Larissa |

**Pendências de decisão que afetam o produto** (detalhes técnicos no [FDD §15](FDD.md#15-riscos-pendências-e-decisões-ainda-necessárias)):

| Pendência | Por que afeta o produto | Quem decide |
|---|---|---|
| I-1: contagem das tentativas (R-03) | Janela de reenvio prometida ao cliente | Larissa, Diego |
| I-2 (FDD P-02): código de erro do endereço não-HTTPS | Mensagem de erro que o usuário recebe no cadastro (AC-03) | Bruno, Larissa |
| I-3 e G-12: acesso administrativo e isolamento (R-01, R-02) | Segurança e confiança dos clientes | Sofia |
| FDD P-09: aviso na **criação** do pedido | No desenho aprovado, só mudanças de status geram aviso; o status inicial (PENDING) nunca seria notificado | Marcos, Bruno |
| FDD P-10: onde aplicar o limite de 64 KB | Se o limite for aplicado no registro, um aviso grande impediria a mudança de status | Larissa, Sofia |
| FDD P-08: identificador no reprocessamento | Deduplicação pelo cliente após um replay | Diego, Larissa |
| FDD P-21: ordem dos avisos com reenvios | O cliente pode receber "enviado" antes de "em processamento" | Diego, Marcos |
| FDD P-06 e P-07: qual secret assina durante a rotação; timestamp na assinatura | O que o cliente precisa implementar para verificar os avisos | Sofia |
| FDD P-18: campos básicos do aviso além do valor total | Conteúdo do contrato com o cliente | Marcos, Diego |
| Rate limiting de saída | Possível excesso de chamadas ao cliente | Em observação ([09:39] Larissa) |

## 12. Critérios de aceitação

**Estados:**
- **Definido:** verificável com as decisões atuais.
- **Depende de decisão:** aguarda uma pendência da §11.
- **Proposta:** forma de validação ainda não aprovada.

Nenhum critério foi verificado, porque a funcionalidade não existe.

| ID | Requisito | Critério (dado / quando / então) | Estado |
|---|---|---|---|
| AC-01 | FR-01, NFR-02 | Dado um webhook que escuta "PROCESSING", quando um pedido do customer passa de PAID para PROCESSING, então um aviso é registrado; se a mudança falhar, nenhum aviso existe | Definido |
| AC-02 | FR-07 | Dado um webhook que escuta só SHIPPED e DELIVERED, quando o pedido vai para PROCESSING, então nenhum aviso é gerado para ele | Definido |
| AC-03 | FR-02, FR-03, NFR-04 | Ao cadastrar com endereço HTTPS, o webhook é criado e a secret é devolvida; com endereço HTTP, o cadastro é recusado com erro de validação | Definido (o código de erro depende da I-2 / FDD P-02) |
| AC-04 | FR-04, FR-05, FR-06 | Um webhook cadastrado pode ser alterado, aparece na listagem do customer e, após removido, deixa de receber avisos | Definido (avisos já pendentes: depende da FDD P-16) |
| AC-05 | FR-10, NFR-10 | O cliente recalcula o HMAC-SHA256 do corpo recebido com a secret do seu webhook e obtém a assinatura enviada; a secret de um webhook não valida avisos de outro | Definido para HMAC sobre o corpo ([09:22] Sofia); a inclusão do timestamp depende da FDD P-07 |
| AC-06 | FR-09 | Após a rotação, a nova secret é devolvida; a anterior permanece válida por 24 horas e depois deixa de valer | Definido quanto à janela; assinatura durante a janela depende da FDD P-06 |
| AC-07 | FR-11, NFR-07 | Um aviso reenviado chega com o mesmo identificador de evento do envio original | Definido para reenvios automáticos; replay manual depende da FDD P-08 |
| AC-08 | FR-12, NFR-06 | Com o cliente fora do ar ou sem responder em 10 s, o aviso é reenviado seguindo os intervalos 1m, 5m, 30m, 2h e 12h | Depende de decisão (I-1) quanto ao número de envios |
| AC-09 | FR-13 | Ao esgotar a política, o aviso fica retido com motivo e data; um ADMIN consegue reprocessá-lo; um não-ADMIN recebe recusa | Definido quanto à regra; a eficácia depende de R-01 (I-3) |
| AC-10 | FR-14 | Cada reprocessamento manual registra o usuário que o executou | Definido |
| AC-11 | FR-08 | O cliente consulta e obtém até 100 entregas recentes com resultado, conteúdo, resposta e tempo de resposta | Definido (unidade dos "100": FDD P-25) |
| AC-12 | NFR-01, OBJ-01 | Em ambiente de teste, com cliente disponível, o aviso chega em menos de 10 s após a mudança de status | Proposta (forma de medição: §4) |
| AC-13 | NFR-03 | Com um cliente indisponível, mudanças de status de pedidos continuam sendo concluídas normalmente | Definido |
| AC-14 | NFR-05 | Um aviso maior que 64 KB não é enviado nem truncado e gera erro | Depende de decisão (FDD P-10) |
| AC-15 | R-02 | Um usuário não consegue cadastrar ou ver webhooks de um customer ao qual não pertence | Depende de decisão (G-12) |
| AC-16 | NFR-12 | A revisão de segurança da Sofia foi concluída antes do deploy | Definido |

## 13. Estratégia de testes e validação

A estratégia descreve validação futura; **nada foi executado**. O detalhamento técnico (46 cenários, níveis e bloqueios) está no [FDD §14](FDD.md#14-estratégia-de-testes-e-critérios-de-aceite-técnicos).

| Camada | O que valida | Estado |
|---|---|---|
| Testes automatizados com banco real | Registro do aviso junto com a mudança de status e ausência de aviso quando a mudança falha (AC-01, AC-02, AC-13) | Definido; segue o padrão de testes de integração já existente no OMS (`tests/orders.test.ts`) |
| Testes de entrega com servidor de teste | Assinatura, identificador, timeout, reenvios e retenção (AC-05, AC-07, AC-08, AC-09) | Definido, exceto o que depende de I-1 e da FDD P-08 |
| Testes de acesso | Recusa sem login, replay só para ADMIN, isolamento entre customers (AC-09, AC-15) | Isolamento depende de G-12 |
| Revisão de segurança | HMAC, geração e rotação de secret, riscos R-01 e R-02 | Obrigatória, a cargo da Sofia ([09:46] Sofia) |
| Revisão do design | Coerência entre PRD, RFC, ADRs e FDD antes de codar | Prevista ([09:50] Larissa) |
| Validação com clientes piloto | Integração real com Atlas, MaxDistribuição e Nova Cargo, e medição da meta de 10 s | `PROPOSTA_PARA_REVISAO` (não discutida na reunião) |
| Regressão | Os testes atuais de pedidos e autenticação continuam passando | Definido |

## 14. Rastreabilidade das informações

- **Transcrição:** todo requisito, objetivo, exclusão e decisão deste PRD cita a fala de origem no formato `[hh:mm] Nome`.
- **Código:** os fatos de código usados aqui aparecem com o caminho do arquivo:
  - ausência de notificação externa;
  - cadastro público com ADMIN;
  - falta de vínculo entre usuário e customer;
  - padrão de testes.
- **Análise documental:** o que vem de análise posterior (I-1, I-3, G-12 e as pendências da FDD) está rotulado como tal e não como decisão da reunião.
- **Documentos derivados:**
  - decisões arquiteturais nos [ADRs](adrs/README.md);
  - proposta e alternativas no [RFC](RFC.md);
  - especificação técnica, contratos e testes no [FDD](FDD.md).
- **Tracker:** a rastreabilidade está consolidada em [`docs/TRACKER.md`](TRACKER.md), com a correspondência item a item (`PRD-FR-NN`, `PRD-NFR-NN`, riscos e critérios) com a fonte.
