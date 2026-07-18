# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

## Status

Aceito — decisão fechada na reunião técnica ([09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h").

## Contexto

O endpoint do cliente pode estar fora do ar no momento da entrega. Já houve caso real de cliente com indisponibilidade de duas horas em manutenção planejada. É preciso definir quantas vezes retentar, com qual espaçamento, e o que fazer com eventos que falham permanentemente — sem deixar eventos "pendurados para sempre" nem matar entregas cedo demais.

## Decisão

- **5 tentativas** por evento, com **backoff exponencial**: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Janela total de ~15 horas entre a primeira falha e a última tentativa.
- Esgotadas as 5 tentativas, o evento é considerado **falha permanente** e movido para uma tabela **`webhook_dead_letter` separada**, com payload, motivo da falha e timestamp — mantém a leitura da outbox principal limpa e serve de evidência para debug e reprocessamento.
- Reprocessamento é **manual**, via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Exige role `ADMIN` e loga quem executou (auditoria) — reutilizando o `requireRole` de `src/middlewares/auth.middleware.ts`.
- Timeout de **10 segundos** por chamada HTTP: cliente que não responde em 10s conta como falha e entra no ciclo de retry.

## Alternativas Consideradas

1. **3 tentativas (mais agressivo)** — descartado. Com 3 tentativas em ~30 minutos, um cliente com indisponibilidade de algumas horas (caso real já ocorrido) perderia eventos definitivamente.
2. **Retry indefinido com backoff** — descartado. Evento ficaria pendurado para sempre se o cliente sumiu; 5 tentativas cobrem uma janela de 12–24h, suficiente.
3. **Marcar como "failed" na própria outbox em vez de tabela DLQ separada** — descartado. Tabela separada deixa a leitura da outbox principal mais limpa e concentra a evidência para debug/reprocessamento.

## Consequências

**Positivas:**

- Tolera indisponibilidades longas do cliente (até ~15h) sem perder eventos.
- Falhas permanentes ficam isoladas, auditáveis e reprocessáveis manualmente.
- Nenhum evento fica em retry eterno consumindo o worker.

**Negativas / trade-offs:**

- Cliente indisponível por mais de ~15 horas perde eventos até que um admin faça o replay manual (aceito: "se um cliente meu cair por 15 horas, ele já tá com problema sério dele").
- Replay é manual — sem notificação proativa ao cliente sobre falhas (email foi adiado para fase futura) nem reprocessamento automático.
- Mais uma tabela e um endpoint admin para manter.
