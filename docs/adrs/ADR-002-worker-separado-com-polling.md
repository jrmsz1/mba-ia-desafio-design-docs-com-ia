# ADR-002 — Worker em processo separado com polling de 2 segundos

| Campo | Valor |
| --- | --- |
| Identificador | ADR-002 |
| Status | **Accepted** |
| Data da decisão | Reunião técnica de "quinta-feira, 09:00" (`TRANSCRICAO.md`, cabeçalho). A data de calendário não foi registrada. |
| Data de registro | 2026-11-16 |
| Decisores | Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM, aceite da latência) |
| Base de código | commit `e7f6311` |
| Relacionados | [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-003](ADR-003-retry-backoff-e-dlq.md), [ADR-006](ADR-006-reuso-da-arquitetura-e-componentes-existentes.md) |

**Legenda:** `DECISÃO_CONFIRMADA`, `DETALHE_ACORDADO`, `PROPOSTA_TÉCNICA`, `QUESTÃO_ABERTA`, conforme definidos no [ADR-001](ADR-001-outbox-transacional-no-mysql.md).

---

## Status

**Accepted.** Polling de 2 s fechado em [09:10] Larissa. Processo separado anotado em [09:12] Larissa. Ambos confirmados no resumo de [09:48] Larissa.

## Contexto

- A outbox ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)) precisa de um consumidor que leia os eventos pendentes e faça as chamadas HTTP.
- O requisito de latência é "abaixo de 10 segundos" ([09:02] Marcos).
- MySQL não oferece um mecanismo nativo de notificação a processos externos equivalente ao `NOTIFY/LISTEN` do Postgres ([09:09] Diego).
- Hoje a aplicação tem **um único entry point**, `src/server.ts`. A função `bootstrap()` sobe o Express e trata `SIGINT`/`SIGTERM` com `prisma.$disconnect()`.
- `src/config/database.ts` exporta a fábrica `createPrismaClient()` e um singleton `prisma` criado no carregamento do módulo.
- `src/config/env.ts` valida o ambiente com Zod e exige `DATABASE_URL` e `JWT_SECRET`.

## Decisão

1. **[DECISÃO_CONFIRMADA]** O worker lê a outbox por **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos.
   - Fontes: [09:09] Diego, [09:10] Marcos ("2 segundos serve"), [09:10] Larissa ("Vamos registrar isso como uma decisão. Worker em polling, 2s"), [09:48] Larissa.
2. **[DECISÃO_CONFIRMADA]** O worker roda como **processo separado**, não dentro da instância da API: "se a API reinicia, perde o worker".
   - Fontes: [09:11] Diego, [09:11] Diego ("Só não pode ser o mesmo processo"), [09:12] Larissa ("Anotado"), [09:48] Larissa ("Worker separado").
3. **[DETALHE_ACORDADO]** Novo entry point `src/worker.ts`, ao lado de `src/server.ts`, executado por um script `npm run worker`.
   - Fontes: [09:11] Larissa (sugestão), [09:11] Bruno ("Pode ser"), [09:28] Bruno, [09:28] Diego ("Beleza").
4. **[DETALHE_ACORDADO]** Mesmo banco e mesma stack, mas com **instância própria de `PrismaClient`**, porque o client é por processo; usa a mesma `DATABASE_URL`.
   - Fontes: [09:11] Bruno, [09:11] Diego, [09:30] Bruno (resposta à pergunta de [09:29] Diego, sem objeção).
   - Nota de fonte: em [09:11] Bruno disse "vai precisar conectar no mesmo banco e usar o mesmo Prisma client". Em [09:30] Bruno esclareceu: "Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova". Este ADR segue o esclarecimento de [09:30]. Interpretar a fala de [09:11] como "mesma biblioteca e schema" é inferência.
5. **[DETALHE_ACORDADO]** **Um único worker.** A ordem é garantida apenas por `order_id` (processamento em ordem de `created_at` da outbox) e somente enquanto houver um worker. Não há garantia de ordenação global, e isso é registrado como **limitação conhecida**.
   - Fontes: [09:12] Diego, [09:13] Larissa ("Documentamos como limitação conhecida"), [09:14] Marcos ("Os clientes nunca pediram garantia de ordering global").
   - Classificado como detalhe acordado, e não como decisão confirmada, porque o fechamento de [09:13] Larissa não usa as marcas formais da legenda e o tema não aparece no resumo de [09:48].
6. **[QUESTÃO_ABERTA]** O nome do arquivo com a lógica de processamento dentro do módulo não foi escolhido: `src/modules/webhooks/webhook.worker.ts` **ou** `webhook.processor.ts`.
   - Fonte: [09:28] Bruno.
7. **[PROPOSTA_TÉCNICA]** (inferência a partir do código, para o FDD):
   - `src/worker.ts` pode criar o client via `createPrismaClient()` e espelhar o encerramento gracioso de `src/server.ts`.
   - Ponto de atenção: importar `src/config/env.ts` exige `JWT_SECRET` também no worker, embora ele não use JWT.
   - O tamanho do batch não foi definido ("batch pequeno", [09:08] Diego).

## Alternativas Consideradas

| Alternativa | Quem levantou | Motivo do descarte |
| --- | --- | --- |
| **Trigger de banco** para reação imediata | [09:09] Bruno | MySQL não tem listener nativo. A trigger "só executa SQL" e não notifica processo externo; seria preciso improvisar, por exemplo escrevendo em arquivo ou chamando um endpoint ([09:09] Diego). |
| Worker **dentro do processo da API** | Ninguém propôs; descartada preventivamente | Reiniciar a API derrubaria o worker ([09:11] Diego). |
| **Mesmo `PrismaClient`** da API | [09:29] Diego (pergunta) | `PrismaClient` é por processo; o worker é outro processo Node ([09:30] Bruno). |
| **Vários workers em paralelo** (particionamento por `order_id` ou lock pessimista) | [09:13] Bruno, [09:13] Diego | **Adiada**: "problema do futuro, não agora" ([09:13] Diego). Perderia a garantia de ordem por pedido ([09:12] Diego). |

## Consequências

**Positivas**
- O ciclo de vida da entrega fica independente da API: reinícios ou deploys da API não interrompem o worker ([09:11] Diego).
- Polling de 2 s atende "abaixo de 10 segundos" com folga ([09:09] Diego, [09:10] Marcos).
- É simples: nenhuma dependência nova, mesma stack e mesmo banco ([09:11] Diego).
- Com um único worker, cada cliente recebe os eventos de um mesmo pedido na ordem da outbox, sem retries ([09:12] Diego). Ver o risco de reordenação no ADR-003.

**Negativas**
- O polling soma até ~2 s de atraso. A transcrição registra: "A latência mínima vai ser 2 segundos no pior caso. Aceitamos." ([09:10] Larissa). Nota de precisão: 2 s é o atraso **máximo** introduzido pelo intervalo de polling, não o mínimo; a fala é citada literalmente.
- O banco recebe consultas periódicas mesmo sem eventos. Inferência.
- Um único worker é ponto único de falha e limita a vazão. A escala ficou adiada ([09:13] Diego).

## Trade-offs explícitos

- **Simplicidade e ausência de infraestrutura nova, pagando com latência de até 2 s e consultas constantes.** Aceito porque atende o requisito de menos de 10 s ([09:10] Larissa, [09:10] Marcos).
- **Ordem por pedido com um worker, pagando com vazão e disponibilidade limitadas.** Aceito porque os clientes não exigem ordenação global ([09:14] Marcos).

## Pendências e riscos

| Tipo | Item | Fonte |
| --- | --- | --- |
| Risco (inferência) | Worker sequencial com timeout HTTP de 10 s ([09:42] Diego): um cliente lento pode atrasar a entrega dos demais e ameaçar o objetivo de menos de 10 s ([09:02] Marcos). | análise |
| Risco (inferência) | Retries com backoff ([ADR-003](ADR-003-retry-backoff-e-dlq.md)) podem fazer um evento posterior do mesmo pedido ser entregue antes de um anterior que está aguardando novo envio. | análise |
| Risco (inferência) | `created_at` não representa necessariamente a ordem de commit entre transações concorrentes. | análise |
| QUESTÃO_ABERTA (lacuna) | Falta definir como recuperar linhas presas em "processando" após queda do worker. O status "processando" foi citado em [09:08] Diego. | [09:08] Diego |
| QUESTÃO_ABERTA | Nome do arquivo de processamento (item 6). | [09:28] Bruno |
| ADIADA | Escala com vários workers. | [09:13] Diego |

## Referências às fontes

- `TRANSCRICAO.md`: [09:02] Marcos, [09:08] Diego, [09:09] Diego, [09:09] Bruno, [09:10] Marcos, [09:10] Larissa, [09:11] Diego, [09:11] Larissa, [09:11] Bruno, [09:12] Diego, [09:12] Larissa, [09:13] Bruno, [09:13] Diego, [09:13] Larissa, [09:14] Marcos, [09:28] Bruno, [09:28] Diego, [09:29] Diego, [09:30] Bruno, [09:42] Diego, [09:48] Larissa.
- Código:
  - `src/server.ts` (`bootstrap`, `shutdown`)
  - `src/config/database.ts` (`createPrismaClient`, `prisma`)
  - `src/config/env.ts` (`envSchema`: `DATABASE_URL`, `JWT_SECRET`)
  - `package.json` (`scripts`; não há script `worker` no commit `e7f6311`)
