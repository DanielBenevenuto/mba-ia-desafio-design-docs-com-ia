# ADR-002 — Worker em processo separado com polling de 2 segundos

## Status

Aceito — decisão registrada na reunião técnica ([09:10] Larissa: "Vamos registrar isso como uma decisão. Worker em polling, 2s").

## Contexto

Com a outbox no MySQL ([ADR-001](./ADR-001-padrao-outbox-no-mysql.md)), é preciso definir **como** os eventos pendentes são lidos e entregues. O requisito de latência vem do PM: para os clientes, "qualquer coisa abaixo de 10 segundos já é tempo real". Também é preciso decidir **onde** esse processamento roda: dentro da instância da API ou em processo próprio.

## Decisão

- **Polling em loop:** a cada **2 segundos**, o worker busca os eventos pendentes mais antigos (ordem de `created_at`), processa em batch pequeno e marca como entregues. Latência mínima de 2s no pior caso — aceita explicitamente.
- **Processo separado da API:** nova entry point `src/worker.ts` (espelhando o padrão de `src/server.ts`) com script `npm run worker`. Se a API reiniciar, o worker não é afetado.
- Mesmo banco e mesma stack: o worker conecta na mesma `DATABASE_URL`, mas com **instância própria de `PrismaClient`** — o client é por processo.
- **Single-worker nesta fase:** com um único worker processando em ordem de `created_at`, o ordering por `order_id` é implícito. Escalar para múltiplos workers (particionamento por `order_id` ou lock pessimista) fica como problema futuro; a limitação de ordering é documentada.

## Alternativas Consideradas

1. **Trigger do banco para notificação reativa** — descartado. MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres; triggers só executam SQL, não notificam processo externo. Avisar o worker exigiria improvisos (escrever em arquivo, bater em endpoint) — "fica esquisito". Polling de 2s atende o requisito de 10s com folga.
2. **Worker dentro da mesma instância da API** — descartado. Se a API reinicia, perde o worker; o ciclo de vida da entrega ficaria acoplado ao ciclo de vida do servidor HTTP.

## Consequências

**Positivas:**

- Atende o SLA de "tempo real" (< 10s) com implementação trivial e sem dependências novas.
- Isolamento de falhas: restart/crash da API não interrompe a entrega de webhooks, e vice-versa.
- Ordering por `order_id` garantido sem esforço extra enquanto for single-worker.

**Negativas / trade-offs:**

- Latência mínima de 2 segundos no pior caso (aceita).
- Queries contínuas ao banco a cada 2s, mesmo sem eventos pendentes (mitigado pelos índices de status e `created_at`).
- Um processo a mais para operar (deploy, monitoramento, restart).
- Single-worker é um ponto único de processamento; escalar exigirá resolver o ordering (particionamento ou lock), decisão adiada de propósito.
