# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | Quinta-feira (reunião técnica, 09:00–09:53) |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |
| **Documentos** | [PRD](./PRD.md) · [FDD](./FDD.md) · [Tracker](./TRACKER.md) |

---

## 1. Resumo executivo (TL;DR)

Propomos adicionar **webhooks outbound** ao OMS via **padrão outbox no MySQL existente**: a mudança de status de pedido insere o evento (payload já renderizado) em uma tabela `webhook_outbox` **dentro da mesma transação** que atualiza o pedido; um **worker em processo separado** faz polling a cada 2 segundos e entrega os eventos por HTTP POST, assinados com **HMAC-SHA256** (secret por endpoint, rotacionável com grace de 24h), com **retry em backoff exponencial (5 tentativas)** e **DLQ** em tabela separada. Garantia **at-least-once** com dedup pelo header `X-Event-Id`. Sem infraestrutura nova: mesmo MySQL, mesmo Prisma, mesmos padrões de módulo do projeto. Estimativa: **3 sprints**, incluindo revisão de segurança.

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram formalmente notificação em tempo real de mudanças de status de pedidos. Hoje eles fazem polling no `GET /orders`, o que torna a integração deles lenta e cara. A Atlas condicionou a permanência na plataforma à entrega até o fim do trimestre (prazo alvo: fim de novembro). Para os clientes, "tempo real" significa **latência abaixo de 10 segundos**.

O OMS não tem hoje nenhum mecanismo de notificação externa, eventos, filas ou webhooks. O ponto natural de captura é a transação de mudança de status em `src/modules/orders/order.service.ts` (`changeStatus`), que já atualiza `orders`, insere em `order_status_history` e ajusta estoque — qualquer solução precisa preservar a atomicidade dessa transação e não adicionar I/O de rede dentro dela.

## 3. Proposta técnica

Visão geral (detalhamento de implementação no [FDD](./FDD.md)):

```
changeStatus (transação MySQL)                    processo separado (npm run worker)
┌─────────────────────────────────┐              ┌──────────────────────────────────┐
│ update orders                   │              │ polling 2s: pendentes por        │
│ insert order_status_history     │              │ created_at, batch pequeno        │
│ ajuste de estoque               │   commit     │   → POST https (HMAC, timeout    │
│ insert webhook_outbox (snapshot)│ ───────────▶ │     10s)                         │
└─────────────────────────────────┘              │   → sucesso: marca entregue      │
                                                 │   → falha: retry 1m/5m/30m/2h/12h│
                                                 │   → 5ª falha: webhook_dead_letter│
                                                 └──────────────────────────────────┘
```

1. **Captura (outbox):** ao mudar o status, se algum webhook ativo do customer assina aquele status, inserimos o evento em `webhook_outbox` na mesma transação — filtragem na inserção (se ninguém assina, nem insere). O payload é um **snapshot** renderizado no momento da inserção. Integração via função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o transaction client atual. → [ADR-001](./adrs/ADR-001-padrao-outbox-no-mysql.md)
2. **Entrega (worker):** processo separado (`src/worker.ts`, `npm run worker`), mesma stack e banco, instância própria de `PrismaClient`. Polling de 2s sobre os pendentes, em ordem de `created_at` — single-worker nesta fase, o que dá ordering implícito por `order_id`. → [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)
3. **Resiliência:** timeout de 10s por chamada; falha agenda retry com backoff 1m/5m/30m/2h/12h (5 tentativas, ~15h de janela); depois, DLQ em `webhook_dead_letter` com replay manual via endpoint admin (role `ADMIN`, auditado). → [ADR-003](./adrs/ADR-003-retry-backoff-exponencial-dlq.md)
4. **Segurança:** HMAC-SHA256 sobre o corpo, header `X-Signature`; secret única por endpoint, gerada por nós, rotacionável com grace de 24h; URL obrigatoriamente `https`; limite de payload de 64KB. → [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
5. **Semântica de entrega:** at-least-once, com `X-Event-Id` (UUID por evento) para dedup no cliente. → [ADR-005](./adrs/ADR-005-at-least-once-x-event-id.md)
6. **API de configuração:** CRUD de webhooks por customer (POST/PATCH/DELETE/GET) + histórico de entregas (`GET /webhooks/:id/deliveries`) + rotação de secret, tudo como módulo padrão em `src/modules/webhooks`, autenticação JWT normal (customer_id no body/path, pois o JWT é do usuário operador). → [ADR-006](./adrs/ADR-006-reuso-padroes-existentes.md)

## 4. Alternativas consideradas

| Alternativa | Trade-off que motivou o descarte |
|---|---|
| **Disparo síncrono no service de orders** | Chamada HTTP dentro da transação: cliente lento travaria mudança de status de outros pedidos, e cliente fora do ar não tem resposta razoável (rollback da mudança de status é inaceitável). |
| **Fila externa (Redis Streams ou similar)** | Exige subir e operar infra nova; para um time pequeno é overengineering — outbox no MySQL existente entrega a mesma garantia sem custo operacional novo. |
| **Trigger do MySQL para reatividade (em vez de polling)** | MySQL não tem NOTIFY/LISTEN; trigger só executa SQL e não notifica processo externo — precisaria de improviso (arquivo, endpoint). Polling de 2s já atende o SLA de 10s. |
| **Retry indefinido / apenas 3 tentativas** | Indefinido deixa evento pendurado para sempre se o cliente sumiu; 3 tentativas (~30min) matariam eventos de clientes com indisponibilidade de horas, caso já ocorrido. 5 tentativas equilibram. |
| **Exactly-once em vez de at-least-once** | Exigiria coordenação dos dois lados e muito mais complexidade; at-least-once com `X-Event-Id` é o padrão de mercado (Stripe, GitHub) e resolve 99% dos casos. |
| **Secret global da plataforma** | Se vaza uma, vaza tudo; secret por endpoint limita o raio de dano de um vazamento (incidente já ocorrido com cliente). |

## 5. Questões em aberto

1. **Rate limiting de saída:** se um cliente tem 50 pedidos mudando de status em um minuto, ele recebe 50 chamadas. Decidido **observar em produção e implementar se virar problema** ([09:38]–[09:39]).
2. **Notificação proativa de falha (email):** avisar o cliente quando o webhook dele falha repetidamente ficou para **próxima fase**, após medir impacto ([09:37]).
3. **Escala do worker:** com múltiplos workers, perde-se o ordering por `order_id`. Caminhos possíveis (particionamento por `order_id`, lock pessimista) ficaram explicitamente como "problema do futuro" ([09:13]).
4. **Endurecimento de autorização do CRUD:** por enquanto qualquer role autenticada configura webhooks; "mais pra frente a gente pode endurecer" ([09:36]–[09:37]).
5. **Arquivamento da outbox:** linhas entregues serão arquivadas após ~30 dias, fora do escopo desta feature ([09:08]).

## 6. Impacto e riscos

- **Código existente:** a única alteração em módulo existente é no `changeStatus` de `src/modules/orders/order.service.ts` (inserção na outbox dentro da transação atual). Sem mudanças no middleware de erro, logger ou autenticação — o módulo novo se encaixa nos padrões vigentes.
- **Banco:** 4 tabelas novas (configuração, outbox, entregas, dead letter); crescimento contínuo da outbox até o arquivamento (fase futura).
- **Operação:** um processo novo para deployar e monitorar (worker); pool de conexões dobra (API + worker).
- **Riscos principais** (matriz completa com probabilidade/impacto no [PRD §10](./PRD.md#10-riscos-e-mitigação)): duplicação de eventos no cliente (mitigada por dedup documentada), vazamento de secret (mitigado por secret por endpoint + rotação), acúmulo na outbox (mitigado por índices e batch pequeno), prazo comercial com risco de churn da Atlas.
- **Segurança:** revisão dedicada da Sofia (≥ 2 dias úteis) sobre HMAC e geração de secret antes do deploy — já embutida na estimativa de 3 sprints.

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL existente](./adrs/ADR-001-padrao-outbox-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling de 2s](./adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](./adrs/ADR-003-retry-backoff-exponencial-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](./adrs/ADR-005-at-least-once-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./adrs/ADR-006-reuso-padroes-existentes.md)
