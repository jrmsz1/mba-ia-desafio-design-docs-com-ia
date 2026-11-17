# ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint e rotação com período de convivência de 24h

| Campo | Valor |
| --- | --- |
| Identificador | ADR-004 |
| Status | **Accepted** |
| Data da decisão | Reunião técnica de "quinta-feira, 09:00" (`TRANSCRICAO.md`, cabeçalho). A data de calendário não foi registrada. |
| Data de registro | 2026-11-16 |
| Decisores | Sofia (Eng. de Segurança, proponente e responsável pelo fechamento), Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Marcos (PM) |
| Base de código | commit `e7f6311` |
| Relacionados | [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-da-arquitetura-e-componentes-existentes.md) |

**Legenda:** `DECISÃO_CONFIRMADA`, `DETALHE_ACORDADO`, `PROPOSTA_TÉCNICA`, `QUESTÃO_ABERTA`, conforme definidos no [ADR-001](ADR-001-outbox-transacional-no-mysql.md).

---

## Status

**Accepted.** Fechado em [09:22] Sofia ("Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h") e confirmado no resumo de [09:48] Larissa. A implementação depende da revisão de segurança da Sofia ([09:46] Sofia).

## Contexto

- Os webhooks expõem dados de pedidos a endpoints fora da infraestrutura da empresa. O cliente precisa conseguir validar que a requisição veio da plataforma e que o payload não foi adulterado no caminho ([09:19] Sofia).
- Os webhooks são apenas de saída ([09:02] Marcos, [09:03] Sofia).
- Já houve um cliente que vazou uma secret em log da própria aplicação ([09:22] Diego).
- No código atual não existe assinatura de mensagens nem armazenamento de secrets de terceiros. A configuração de logs em `src/shared/logger/index.ts` (`redactPaths`) mascara `authorization`, `cookie`, `password`, `passwordHash`, `token` e `accessToken`, mas **não** um campo `secret`.

## Decisão

1. **[DECISÃO_CONFIRMADA]** Assinar o **corpo do request** com **HMAC-SHA256** e enviar a assinatura num header; o cliente verifica do lado dele.
   - Fontes: [09:20] Sofia, [09:20] Sofia ("SHA-256. HMAC-SHA256 é o padrão de mercado"), [09:22] Sofia ("Decidido: HMAC-SHA256 sobre o corpo do request…"), [09:48] Larissa.
   - **[DETALHE_ACORDADO]** O header se chama `X-Signature` ([09:20] Sofia, "um header tipo X-Signature"; confirmado na lista de headers de [09:44] Diego).
2. **[DECISÃO_CONFIRMADA]** **Uma secret por endpoint de webhook**, nunca uma secret global: "Senão se vaza uma, vaza tudo".
   - Fontes: [09:21] Sofia, [09:22] Sofia, [09:48] Larissa.
3. **[DECISÃO_CONFIRMADA]** **A secret é rotacionável**: o cliente solicita uma nova pela API, e a anterior continua válida por **24 horas** em paralelo antes de expirar.
   - Fontes: [09:21] Sofia, [09:22] Sofia ("suporte a rotação com grace period de 24h"), [09:48] Larissa.
4. **[DETALHE_ACORDADO]** A **secret é gerada pela plataforma** e devolvida na criação do webhook.
   - Fonte: [09:31] Marcos.
5. **[DETALHE_ACORDADO]** A configuração de webhook guarda url, secret, customer_id e estado ativo.
   - Fontes: [09:21] Bruno, [09:21] Sofia ("Sim").
6. **[DETALHE_ACORDADO]** **URL obrigatoriamente `https`**. Um cadastro com `http` é recusado com erro de validação no schema Zod. A própria Sofia classificou isso como validação, não como decisão arquitetural.
   - Fonte: [09:23] Sofia.
   - O código de erro correspondente está em [ADR-006](ADR-006-reuso-da-arquitetura-e-componentes-existentes.md), pendência I-2.
7. **[DETALHE_ACORDADO]** Haverá **revisão de segurança pela Sofia**, de no mínimo **dois dias úteis**, sobre o código de HMAC e de geração de secret, antes do deploy.
   - Fontes: [09:46] Sofia, [09:47] Larissa ("Combinado"), [09:49] Sofia.

## Alternativas Consideradas

| Alternativa | Quem levantou | Motivo do descarte |
| --- | --- | --- |
| **Secret global** da plataforma, compartilhada por todos os endpoints | Ninguém propôs. Sofia a descartou explicitamente ([09:21] Sofia). | O vazamento de uma secret comprometeria todos os clientes ([09:21] Sofia). |

Nenhuma outra alternativa de autenticação de mensagens foi discutida na reunião. Este ADR não registra alternativas que não tenham fonte.

## Consequências

**Positivas**
- O cliente consegue verificar autenticidade e integridade de cada entrega ([09:19] Sofia).
- Um vazamento fica restrito a um endpoint ([09:21] Sofia).
- HMAC-SHA256 tem bibliotecas amplamente disponíveis para os clientes ([09:20] Sofia).
- A rotação com 24 h de convivência permite trocar a secret sem interromper as integrações ([09:21] Sofia), o que responde ao histórico de vazamento ([09:22] Diego).

**Negativas**
- Para assinar, a plataforma precisa ter acesso à secret original. Ela não pode ser guardada apenas como hash. Inferência técnica: decorre da própria natureza do HMAC.
- Existe uma nova superfície sensível: secrets em banco, em respostas da API (na criação e na rotação) e potencialmente em logs. O `redactPaths` atual não cobre `secret`.
- A rotação adiciona estado (duas secrets válidas ao mesmo tempo) e complexidade de expiração.
- A verificação da assinatura e a eventual detecção de replay ficam a cargo do cliente.

## Trade-offs explícitos

- **Autenticidade verificável pelo cliente em troca da guarda e da rotação de segredos recuperáveis pela plataforma.**
- **Isolamento por endpoint em troca de mais segredos a gerir.** Aceito para limitar o alcance de um vazamento ([09:21] Sofia).
- **24 h de convivência das secrets em troca de uma janela maior de validade de uma secret possivelmente comprometida.** Aceito para dar tempo de migração ao cliente ([09:21] Sofia).

## Pendências e riscos

As mitigações abaixo são `PROPOSTA_PARA_REVISAO`, sujeitas à revisão de segurança da Sofia ([09:46] Sofia). **Não são decisões da reunião.**

| Tipo | Item |
| --- | --- |
| QUESTÃO_ABERTA (lacuna) | Durante as 24 h de convivência, qual secret assina os envios, ou se são enviadas duas assinaturas. |
| QUESTÃO_ABERTA (lacuna) | Se o `X-Timestamp` ([09:44] Diego, "pra cliente conseguir detectar replay attack se quiser") entra no conteúdo assinado. Hoje a decisão fala em assinar "o corpo do request" ([09:22] Sofia). |
| QUESTÃO_ABERTA (lacuna) | Como a secret é armazenada em repouso (por exemplo, cifrada), dado que precisa ser recuperável. |
| QUESTÃO_ABERTA (lacuna) | O contrato do endpoint de rotação: rota, payload e resposta. A existência foi decidida em [09:21] Sofia; o detalhe pertence ao FDD. |
| PROPOSTA_PARA_REVISAO | Incluir campos de secret no `redactPaths` de `src/shared/logger/index.ts`. |
| PROPOSTA_PARA_REVISAO | Avaliar restrições à URL de destino além de `https`, como endereços internos, para mitigar SSRF. Não foi discutido na reunião. |
| Dependência | A revisão de segurança da Sofia antes do deploy ([09:46] Sofia). |
| Risco relacionado | Um usuário autenticado pode cadastrar webhooks para qualquer customer (G-12). Assim, a assinatura HMAC autentica a origem, mas não impede que um destino indevido receba eventos legitimamente assinados. Ver [ADR-006](ADR-006-reuso-da-arquitetura-e-componentes-existentes.md#riscos-de-segurança-pré-existentes-i-3-e-g-12). |

## Referências às fontes

- `TRANSCRICAO.md`: [09:02] Marcos, [09:03] Sofia, [09:19] Sofia, [09:20] Sofia, [09:20] Bruno, [09:21] Sofia, [09:21] Bruno, [09:22] Diego, [09:22] Sofia, [09:23] Sofia, [09:31] Marcos, [09:44] Diego, [09:46] Sofia, [09:47] Larissa, [09:48] Larissa, [09:49] Sofia.
- Código: `src/shared/logger/index.ts` (`redactPaths`, `createLogger`).
