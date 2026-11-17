# Execução do desafio — Documentação com IA

> Esta seção documenta o processo real de produção do pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos**. O enunciado original do desafio está referenciado na seção [Enunciado original](#enunciado-original).

## Sobre o desafio

O desafio pede transformar a transcrição de uma reunião técnica ([`TRANSCRICAO.md`](TRANSCRICAO.md)) num pacote de design docs para uma nova feature de um Order Management System (OMS) já existente em Node.js, TypeScript, Prisma e MySQL. A feature são webhooks de saída que avisam clientes B2B quando o status de um pedido muda. O pacote reúne PRD, RFC, FDD, de 5 a 8 ADRs, um Tracker de rastreabilidade e este README.

A entrega é **somente documental**. Cada requisito, decisão e restrição precisa ser rastreável à transcrição ou ao código. O trabalho exigiu separar o que a reunião **decidiu** do que foi apenas sugerido, adiado ou descartado. Também exigiu registrar como pendência, e não como decisão, o que a análise do código revelou depois, por exemplo inconsistências e riscos de segurança.

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
|---|---|
| **Claude Code** (CLI, modelo Claude Opus 5.5) | Executor documental. Leu o código e a transcrição, produziu os documentos, rodou as validações mecânicas (citações, caminhos, links, JSON) e aplicou as correções. Atuou sempre dentro do escopo de escrita autorizado em cada etapa. |
| **ChatGPT** | Avaliador independente entre etapas e apoio na preparação dos prompts de cada etapa (informado pelo autor; usado fora do Claude Code) |
| **Universal Engineering Kit v1.5** (`.claude/`) | Conjunto local de regras, skills e agentes de engenharia usado pelo Claude Code. É auxiliar de trabalho, **não faz parte da feature** e **não é versionado** neste repositório (excluído por `.git/info/exclude`). |
| Skills do Kit **efetivamente invocadas** | `/project-discovery` (inventário do OMS), `/source-traceability` (classificação das falas), `/design-docs` (fronteiras entre documentos), `/transactional-outbox` (desenho da outbox no FDD), `/docs-quality-gate` (gates de cada etapa). A matriz de falhas da outbox e o `SKILL.md` de `/plan-test-specs` foram **consultados como referência**, sem invocação. |
| Agente do Kit **efetivamente usado** | `traceability-auditor`, como revisor independente e somente leitura dos ADRs, do RFC, do FDD, do PRD e do Tracker |
| **Context7** (MCP) | Consulta à documentação do Prisma sobre os parâmetros da transação interativa. O FDD registra o valor padrão como pendente de confirmação na documentação da versão 5.22.0. |
| Script Python local (não versionado) | Gerou a matriz do Tracker a partir do inventário e validou cada linha: a fala existe naquele minuto com aquele participante, e o arquivo de código existe |

Os demais agentes e skills do Kit (por exemplo `/decide`, `/contract-compatibility`, `/edge-cases`, `/owasp-top10`) **não foram invocados**. Os temas deles foram tratados com as regras do Kit carregadas na sessão.

## Workflow adotado

Ordem cronológica real. Cada etapa só começou depois de uma autorização explícita, com a lista de arquivos que podiam ser alterados.

A lista numera **atividades**. Os prompts usam a numeração de **etapas** do roteiro do autor, indicada entre parênteses em cada item. As Etapas 1 e 2 desse roteiro antecederam a sessão documentada aqui.

1. **Descoberta e inventário do OMS** (Etapa 3; somente leitura): stack, módulos, transação de `changeStatus`, `requireRole`, `AppError`, `errorMiddleware` e logger Pino. Os testes **não** foram executados, porque `tests/setup.ts` apaga tabelas.
2. **Análise da transcrição e rastreabilidade** (Etapa 3 e sua revisão final): classificação das falas em decidido, requisito, proposta, rejeitado, adiado e questão aberta. Uma **microauditoria** confirmou três achados que acompanharam todo o pacote:
   - **I-1:** "5 tentativas" contra 5 intervalos de backoff;
   - **I-2:** o `validate` atual emite `VALIDATION_ERROR`, nunca `WEBHOOK_INVALID_URL`;
   - **I-3:** o cadastro público aceita `role: ADMIN`.

   A lacuna **G-12** (sem vínculo entre usuário e customer) também foi registrada.
3. **Seis ADRs** (Etapa 4), auditados pelo `traceability-auditor`.
4. **RFC** (Etapa 5), salvo e revisado pelo autor entre as rodadas, com três auditorias até PASS.
5. **FDD** (Etapa 6): 16 seções, contratos, integração com 15 arquivos reais e testes especificados. Auditado até PASS.
6. **PRD** (Etapa 7), auditado até PASS.
7. **Tracker** (Etapa 8): inventário de 499 itens, matriz validada por script e auditoria semântica independente.
8. **Fechamento da divergência DIV-01** entre o FDD e o ADR-001 (fechamento da Etapa 8).
9. **Consolidação** deste README e preparação para a revisão final de entrega (Etapa 9 e ajustes seguintes).

## Prompts customizados

Os trechos abaixo foram copiados literalmente dos prompts usados nesta sessão. Marcas `[…]` indicam trechos omitidos.

**Prompt A: reauditoria final do RFC** (Etapa 5). Prompt do autor ao Claude Code, reproduzido na íntegra:

```text
Execute a reauditoria final da Etapa 5, em modo SOMENTE LEITURA.

O arquivo `docs/RFC.md` foi revisado com base nos 13 achados anteriores: M-5, M-6, M-7 e m-1 a m-10.

Utilize `/docs-quality-gate` e o agente `traceability-auditor`.

Confira o RFC integralmente contra:
- `TRANSCRICAO.md`;
- os seis ADRs locais;
- o código existente;
- os critérios obrigatórios do `README.md`.

Verifique especificamente se:
1. A conclusão da reunião sobre o polling de 2 segundos foi preservada.
2. As seis lacunas identificadas nos ADRs aparecem como pendências da análise, sem serem apresentadas como decisões históricas.
3. A seção "Impacto e riscos" atende à rubrica.
4. Os dez achados menores foram corrigidos.
5. Não existem contradições, afirmações sem fonte ou referências inválidas.
6. O RFC continua conciso, aproximadamente 2 a 4 páginas.
7. Nenhum arquivo fora do escopo documental foi alterado.

Não modifique arquivos, não execute migrations, seeds ou testes destrutivos, e não faça commit nem push.

Entregue PASS, NEEDS_REVIEW ou BLOCKED, com evidências específicas. Se houver algum problema, indique arquivo, linha, causa e correção mínima.

Pare ao concluir a auditoria.
```

**Prompt B: autorização restrita do Tracker** (Etapa 8). Prompt do autor ao Claude Code (excerto):

```text
# MBA Full Cycle — Etapa 8: Tracker de Rastreabilidade

A Etapa 7 (PRD) foi aprovada para continuidade pelo avaliador independente.
[…]
**AUTORIZO EXCLUSIVAMENTE a elaboração de `docs/TRACKER.md`.**
[…]
## 1. Objetivo

Construir uma matriz de rastreabilidade que comprove a origem dos requisitos, decisões, restrições e demais itens identificáveis registrados no pacote documental.

Não invente fontes para satisfazer os critérios quantitativos.

A rastreabilidade precisa ser verificável por um avaliador independente.
[…]
**Nunca manipule a composição do Tracker apenas para alcançar esses limites.**

Se algum percentual não for atingido com dados verdadeiros, apresente NEEDS_REVIEW e explique a situação.
```

**Prompt C: instrução do Claude Code ao agente `traceability-auditor`** na auditoria do FDD (Etapa 6), em excerto:

```text
Foque em julgamento semântico, com arquivo:linha exata:
1. Rótulos: cada `DECIDIDO`/`ACORDADO` tem fonte que sustenta (DECIDIDO só com fechamento explícito ou resumo [09:48]); nada `PROPOSTA_TÉCNICA`/`NEEDS_DECISION` apresentado como decidido; mitigações de segurança como PROPOSTA_PARA_REVISAO ligadas à Sofia.
```

## Iterações e ajustes

Depois da descoberta, cada documento passou por rodadas de geração, auditoria e correção:
- ADRs: uma auditoria e uma reauditoria;
- RFC: três auditorias;
- FDD e Tracker: cada um com uma auditoria e uma reauditoria;
- PRD: uma auditoria (PASS com achados menores, corrigidos em seguida);
- divergência DIV-01: um fechamento próprio.

Em seguida vieram o ajuste editorial do README, a auditoria integral de entrega e o fechamento das oito inconsistências encontradas nela (m-1 a m-8, registradas no Tracker §5), com reauditoria localizada. As iterações mais relevantes:

| # | Documento | Problema identificado | Quem revelou | Correção | Resultado verificado |
|---|---|---|---|---|---|
| 1 | ADRs | O UUID da outbox estava rotulado como decisão confirmada, mas na transcrição é só uma resposta de Larissa, sem marca de fechamento e depois do resumo ([09:51]). Havia também 11 problemas menores: citações parafraseadas entre aspas, fontes de fechamento ausentes e Status só como campo de metadados, sem seção `## Status`. | `traceability-auditor` (NEEDS_REVIEW) | UUID reclassificado como detalhe acordado; citações tornadas literais; fechamentos ([09:17], [09:33], [09:48]) acrescentados | Reauditoria PASS |
| 2 | RFC | A versão salva tinha autor indefinido, G-12 rotulado como "bloqueador" (contrariando o ADR-006), parágrafo obsoleto dizendo que os ADRs eram "inacessíveis", citações de contexto enganosas (SSRF não foi discutido) e omissão das lacunas dos ADRs | `/docs-quality-gate` + `traceability-auditor` (NEEDS_REVIEW em duas rodadas) | O autor revisou o RFC entre as rodadas. Um achado do auditor foi **rejeitado** depois de conferido na transcrição: a citação [09:12] Larissa estava correta, porque ali ela diz "Anotado." | 3ª rodada PASS; o último ajuste removeu "(análise técnica)" da justificativa sobre triggers ([09:09] Diego) |
| 3 | FDD | Quatro itens acordados apareciam como `DECIDIDO`, incluindo HTTPS, que Sofia chamou de "só uma validação no schema Zod" ([09:23]). Um teste e um critério de aceite davam por pronta a sequência de retry, que depende de I-1. | `traceability-auditor` (2 MAJOR, 11 MINOR) | Rótulos corrigidos; T-RTY-01 passou a testar só a configuração; novo T-RTY-05 bloqueado por I-1; P-24 e P-25 criados | Reauditoria apontou um teste citado e não definido (T-ERR-01), que foi criado → PASS |
| 4 | Tracker | A §6 afirmava uma revisão que ainda não tinha terminado; havia 7 duplicatas dentro do FDD, cerca de 20 itens fora do inventário e 3 propostas com fonte forçada | `traceability-auditor` leu as 475 linhas (NEEDS_REVIEW) | Duplicatas removidas; 21 itens incluídos; propostas sem fonte movidas para "sem rastreabilidade"; a cobertura caiu de 97,9% para 97,0% em vez de ser maquiada | Reauditoria PASS |
| 5 | FDD + Tracker | **DIV-01** (chamada de D-01 durante a auditoria): os índices `status`/`created_at` e o "batch pequeno" ([09:08] Diego) estavam como `ACORDADO` no FDD e `PROPOSTA_TÉCNICA` no ADR-001 | Detectada na auditoria do Tracker | FDD alinhado ao ADR-001, conferido na fala literal; histórico preservado no Tracker | Gate PASS; métricas inalteradas |

## Resultados e rastreabilidade

Estes números descrevem **documentação produzida**, não funcionalidade implementada.

| Indicador | Valor | Onde verificar |
|---|---|---|
| ADRs | 6 | [`docs/adrs/`](docs/adrs/README.md) |
| Endpoints documentados no FDD | 7 | [`docs/FDD.md`](docs/FDD.md) §7 |
| Testes especificados (não executados) e pendências | 46 testes e 25 pendências | [`docs/FDD.md`](docs/FDD.md) §14 e §15 |
| Requisitos funcionais / não funcionais | 14 / 12 | [`docs/PRD.md`](docs/PRD.md) §7 e §8 |
| Critérios de aceitação do PRD | 16 | [`docs/PRD.md`](docs/PRD.md) §12 |
| Itens rastreados / identificáveis | 484 / 499 (97,0%) | [`docs/TRACKER.md`](docs/TRACKER.md) §2 |
| Linhas da matriz com fonte `TRANSCRICAO` | 387 / 484 (80,0%) | [`docs/TRACKER.md`](docs/TRACKER.md) §2 |
| Linhas com fonte `CODIGO` e arquivo real | 97 | [`docs/TRACKER.md`](docs/TRACKER.md) §3 |

**Os 15 itens sem rastreabilidade válida** estão listados à parte na §4 do Tracker. São propostas da análise sem fala nem fato de código que as motive, como as restrições contra SSRF, a validação com clientes piloto e a ordem de deploy. Não receberam fonte inventada e nenhum deles aparece como decisão da reunião.

## Como navegar a entrega

Ordem de leitura sugerida:

1. [`docs/PRD.md`](docs/PRD.md): problema, público, escopo, requisitos, riscos e critérios de aceitação
2. [`docs/RFC.md`](docs/RFC.md): proposta arquitetural, alternativas descartadas e questões em aberto
3. [`docs/adrs/README.md`](docs/adrs/README.md): índice das decisões
   - [ADR-001: Outbox transacional no MySQL](docs/adrs/ADR-001-outbox-transacional-no-mysql.md)
   - [ADR-002: Worker separado com polling](docs/adrs/ADR-002-worker-separado-com-polling.md)
   - [ADR-003: Retry, backoff e DLQ](docs/adrs/ADR-003-retry-backoff-e-dlq.md)
   - [ADR-004: HMAC-SHA256 e rotação de secret](docs/adrs/ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md)
   - [ADR-005: At-least-once com X-Event-Id](docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
   - [ADR-006: Reuso da arquitetura e dos componentes](docs/adrs/ADR-006-reuso-da-arquitetura-e-componentes-existentes.md)
4. [`docs/FDD.md`](docs/FDD.md): especificação de implementação, contratos, erros, integração com o código e testes
5. [`docs/TRACKER.md`](docs/TRACKER.md): origem de cada item na transcrição ou no código
6. [`TRANSCRICAO.md`](TRANSCRICAO.md): fonte primária, não alterada

## Limites da entrega

- O desafio é **de documentação**: os webhooks **não foram implementados** e nada foi alterado em `src/`, `prisma/`, `tests/` ou nas configurações.
- **Nenhuma** migration, seed ou teste foi executado. Os testes estão **especificados** no FDD (§14) e marcados como NOT_RUN.
- As pendências `NEEDS_DECISION` continuam abertas e documentadas: I-1, I-2, fan-out, onde aplicar o limite de 64 KB, identidade do evento no replay e as demais do FDD §15.
- Os riscos de segurança **I-3** (cadastro público com ADMIN) e **G-12** (sem autorização por objeto entre usuário e customer) **não foram corrigidos no código**. As mitigações estão como `PROPOSTA_PARA_REVISAO`, sujeitas à revisão de segurança da Sofia.

## Enunciado original

O enunciado completo do desafio, com requisitos e critérios de aceite, está no repositório base: [README original do desafio](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia/blob/main/README.md).
