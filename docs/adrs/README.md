# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante deve ser registrada aqui em arquivos individuais,
nomeados sequencialmente no formato `ADR-NNN-titulo-em-kebab-case.md`
(por exemplo `ADR-001-outbox-transacional-no-mysql.md`).

## Índice

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-transacional-no-mysql.md) | Outbox transacional no MySQL existente | Accepted |
| [ADR-002](ADR-002-worker-separado-com-polling.md) | Worker em processo separado com polling de 2 segundos | Accepted |
| [ADR-003](ADR-003-retry-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ em tabela separada | Accepted (semântica da contagem de tentativas pendente) |
| [ADR-004](ADR-004-assinatura-hmac-sha256-e-rotacao-de-secret.md) | Assinatura HMAC-SHA256 com secret por endpoint e rotação | Accepted |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação via `X-Event-Id` | Accepted |
| [ADR-006](ADR-006-reuso-da-arquitetura-e-componentes-existentes.md) | Reuso da arquitetura modular e dos componentes existentes | Accepted |
