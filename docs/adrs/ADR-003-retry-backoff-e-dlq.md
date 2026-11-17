# ADR-003 — Retry com backoff exponencial e Dead Letter Queue em tabela separada

| Campo | Valor |
| --- | --- |
| Identificador | ADR-003 |
| Status | **Accepted**, com **pendência de esclarecimento** sobre a semântica da contagem de tentativas (ver "Pendência I-1") |
| Data da decisão | Reunião técnica de "quinta-feira, 09:00" (`TRANSCRICAO.md`, cabeçalho). A data de calendário não foi registrada. |
| Data de registro | 2026-11-16 |
| Decisores | Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Sofia (Eng. de Segurança, autorização do replay), Marcos (PM, aceite da janela) |
| Base de código | commit `e7f6311` |
| Relacionados | [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-da-arquitetura-e-componentes-existentes.md) |

**Legenda:** `DECISÃO_CONFIRMADA`, `DETALHE_ACORDADO`, `PROPOSTA_TÉCNICA`, `QUESTÃO_ABERTA`, conforme definidos no [ADR-001](ADR-001-outbox-transacional-no-mysql.md).

---

## Status

**Accepted** para a escolha arquitetural de retry, backoff e DLQ ([09:17] Larissa, [09:48] Larissa).
- Os valores (5 tentativas; 1m/5m/30m/2h/12h) estão preservados literalmente.
- A **semântica da contagem de tentativas está pendente de esclarecimento** (ver "Pendência I-1").

## Contexto

- Clientes podem ficar indisponíveis, inclusive por longos períodos: já houve um cliente com "indisponibilidade de duas horas em manutenção planejada" ([09:16] Diego).
- A reunião precisava definir o que o worker ([ADR-002](ADR-002-worker-separado-com-polling.md)) faz quando a entrega falha ([09:14] Larissa) e o que fazer com eventos que nunca conseguem ser entregues.

## Decisão

1. **[DECISÃO_CONFIRMADA]** **Retry com backoff exponencial**: o intervalo cresce a cada tentativa e, ao atingir um teto, a falha passa a ser considerada permanente.
   - Fontes: [09:15] Diego (proposta), [09:17] Larissa ("Decidido: …"), [09:48] Larissa ("Retry com backoff exponencial").
2. **[DECISÃO_CONFIRMADA]** Valores aprovados, registrados **literalmente**:
   - **5 tentativas**;
   - **intervalos de 1m / 5m / 30m / 2h / 12h**.
   - Fontes: [09:15] Diego ("Eu sugiro 5"), [09:17] Diego ("1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas"), [09:17] Larissa ("Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h"), [09:48] Larissa ("backoff exponencial 1m/5m/30m/2h/12h, total 5 tentativas, depois DLQ").
3. **[DECISÃO_CONFIRMADA]** **Esgotada a política, o evento vai para a DLQ**, persistida em **tabela separada** da outbox, e não como um status "failed" na própria outbox.
   - Fontes: [09:17] Larissa (pergunta), [09:18] Diego, [09:18] Bruno ("Faz sentido"), [09:48] Larissa ("depois DLQ persistida em tabela separada").
4. **[DETALHE_ACORDADO]** A tabela se chama `webhook_dead_letter` e guarda payload, motivo da falha e timestamp. Serve como evidência para depuração e reprocessamento.
   - Fontes: [09:18] Diego, [09:18] Bruno.
5. **[DECISÃO_CONFIRMADA]** O **reprocessamento é manual**, por endpoint administrativo.
   - Fontes: [09:18] Diego, [09:19] Larissa ("Anota também. Endpoint admin pra replay manual de DLQ").
   - **[DETALHE_ACORDADO]** Rota `POST /admin/webhooks/dead-letter/:id/replay` ([09:18] Diego, reafirmada em [09:35] Diego).
   - **[DETALHE_ACORDADO]** O replay "recoloca na outbox como pendente" ([09:18] Diego).
   - O contrato detalhado pertence ao FDD.
6. **[DECISÃO_CONFIRMADA]** O replay **exige role `ADMIN`**, reaproveitando `requireRole` (`src/middlewares/auth.middleware.ts`).
   - Fontes: [09:36] Sofia, [09:36] Larissa ("Decidido, role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe"), [09:48] Larissa.
7. **[DETALHE_ACORDADO]** O endpoint de replay **registra quem o executou**, para auditoria.
   - Fonte: [09:36] Sofia.
   - O mecanismo (log Pino ou persistência) não foi definido: ver pendências.
8. **[DETALHE_ACORDADO]** **Timeout HTTP de 10 s**: um cliente que não responde nesse prazo é tratado como falha e marcado para retry.
   - Fontes: [09:42] Diego, [09:42] Sofia ("Bom").

## Pendência I-1 — semântica da contagem de tentativas

**Status: QUESTÃO_ABERTA. Encontrada na análise documental; não foi levantada na reunião.**

Os valores do item 2 estão aprovados e **não são alterados por este ADR**. O que não está claro é **o que conta como "tentativa"**.

**Termos:**
- **envio inicial**: a primeira chamada HTTP do evento;
- **retry**: cada nova chamada depois de uma falha;
- **intervalo**: a espera que precede cada retry.

| Leitura | "5 tentativas" significaria | Intervalos efetivamente usados | Janela entre a primeira falha e o último envio | Conflito com a fonte |
| --- | --- | --- | --- | --- |
| A | 5 chamadas HTTP no total | 4 dos 5 (1m, 5m, 30m, 2h) | 2h36min | O intervalo de 12h nunca seria usado. Contradiz "Total de quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). |
| B | 5 retries após o envio inicial | Todos os 5 | 1+5+30+120+720 min = 14h36min ("quase 15 horas") | Contradiz "total 5 tentativas" ([09:48] Larissa). |

- **Este ADR não adota nenhuma das duas leituras.** Não se deve documentar "6 tentativas", "4 retries" ou qualquer outra contagem como se tivesse sido aprovada.
- **Quem deve esclarecer:**
  - Larissa, que declarou a decisão ([09:17], [09:48]);
  - Diego, que propôs os valores e calculou a janela ([09:15], [09:17]).
  - Marcos aceitou a janela de 15 horas como prazo de negócio ([09:17] Marcos) e deve ser consultado se a janela mudar.
- **Impacto na implementação** (até o esclarecimento, o FDD deve marcar `NEEDS_DECISION`):
  - o limite do contador de tentativas que dispara a ida para a DLQ;
  - quantos dos intervalos aprovados são de fato consumidos;
  - a janela máxima até a DLQ, que é a informação a comunicar aos clientes no portal ([09:26] Marcos).
- **Impacto nos testes** (a escrever após o esclarecimento):
  - o caso de fronteira "última tentativa falha → DLQ";
  - o caso "penúltima tentativa falha → agenda um novo envio";
  - a verificação da sequência de intervalos;
  - a janela total.
  - Os testes não devem ser escritos para uma leitura escolhida unilateralmente.

## Alternativas Consideradas

| Alternativa | Quem levantou | Motivo do descarte |
| --- | --- | --- |
| **Retry indefinido** com backoff | Ninguém propôs na reunião. Diego mencionou que "algumas pessoas defendem" ([09:15] Diego). | O evento poderia ficar pendurado para sempre se o cliente sumisse ([09:15] Diego). |
| **3 tentativas** | [09:16] Bruno ("Mais agressivo") | Seriam três retentativas em 30 minutos, insuficientes para indisponibilidades reais de 2 horas ([09:16] Diego). Escolha final por 5 em [09:16] Larissa ("Cinco fica bom"). |
| Marcar como **"failed" na própria outbox**, sem tabela separada | [09:17] Larissa (pergunta) | A tabela separada deixa a leitura da outbox principal "mais limpa" e serve de evidência para depuração e reprocessamento ([09:18] Diego). |

## Consequências

**Positivas**
- Cobre indisponibilidades longas do cliente. A janela citada é de "quase 15 horas", sujeita à Pendência I-1. Marcos a considerou aceitável: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos).
- A falha permanente é finita, e os eventos não ficam pendentes indefinidamente.
- A DLQ preserva evidência (payload e motivo) e permite reprocessamento controlado.
- A outbox principal fica restrita a eventos em trânsito.

**Negativas**
- Um evento pode chegar horas depois da mudança de status, enquanto houver retries.
- O reprocessamento depende de ação humana.
- Um evento em espera de retry pode ser ultrapassado por um evento posterior do mesmo pedido, o que quebra a ordem por `order_id` de [ADR-002](ADR-002-worker-separado-com-polling.md). Inferência da análise, não discutida na reunião.
- O replay reenvia um evento já registrado, o que pode gerar duplicata no cliente. O tratamento está no [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md).

## Trade-offs explícitos

- **Persistência nas tentativas (cobertura de ~15 h) contra entrega tardia.** Aceito para tolerar manutenções planejadas ([09:16] Diego, [09:17] Marcos).
- **Teto finito com intervenção manual contra retry indefinido automático.** Aceito para evitar eventos pendurados ([09:15] Diego).
- **Uma tabela a mais (DLQ) em troca de uma outbox mais enxuta e de evidência isolada** ([09:18] Diego).

## Pendências e riscos

| Tipo | Item | Fonte |
| --- | --- | --- |
| QUESTÃO_ABERTA | **I-1**, semântica da contagem de tentativas (seção acima). | [09:17] Diego, [09:17] Larissa, [09:48] Larissa |
| QUESTÃO_ABERTA (lacuna) | Que resposta HTTP conta como sucesso, e se respostas 4xx também geram retry. Só o timeout de 10 s foi definido. | [09:42] Diego |
| QUESTÃO_ABERTA (lacuna) | Papel do status "falhou" listado para a outbox ([09:08] Diego), já que a falha permanente vai para a DLQ separada. | [09:08] Diego, [09:18] Diego |
| QUESTÃO_ABERTA (lacuna) | Se o replay preserva o `event_id` original. Ver [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md). | [09:18] Diego, [09:25] Diego |
| QUESTÃO_ABERTA | Mecanismo do registro de auditoria do replay. | [09:36] Sofia |
| **Risco de segurança pré-existente (I-3)** | `requireRole('ADMIN')` só protege o replay se a role `ADMIN` não puder ser obtida indevidamente. No commit `e7f6311`, o cadastro público aceita `role: 'ADMIN'`. Detalhes, evidências e mitigações, todas `PROPOSTA_PARA_REVISAO`, estão no [ADR-006](ADR-006-reuso-da-arquitetura-e-componentes-existentes.md#riscos-de-segurança-pré-existentes-i-3-e-g-12). Pendente da revisão de segurança da Sofia ([09:46] Sofia). | código |

## Referências às fontes

- `TRANSCRICAO.md`: [09:08] Diego, [09:14] Larissa, [09:15] Diego, [09:16] Bruno, [09:16] Diego, [09:16] Larissa, [09:17] Diego, [09:17] Marcos, [09:17] Larissa, [09:18] Diego, [09:18] Bruno, [09:19] Larissa, [09:25] Diego, [09:26] Marcos, [09:35] Diego, [09:36] Sofia, [09:36] Larissa, [09:42] Diego, [09:42] Sofia, [09:46] Sofia, [09:48] Larissa.
- Código: `src/middlewares/auth.middleware.ts` (`requireRole`, `authenticate`).
