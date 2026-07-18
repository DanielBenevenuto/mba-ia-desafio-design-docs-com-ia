# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Marcos (PM), consolidado a partir da reunião técnica |
| **Status** | Aprovado (decisões fechadas na reunião de quinta-feira, 09:00) |
| **Fontes** | `TRANSCRICAO.md` e código-fonte do OMS |
| **Documentos relacionados** | [RFC](./RFC.md) · [FDD](./FDD.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |

---

## 1. Resumo e contexto da feature

O Order Management System (OMS) hoje não possui nenhum mecanismo de notificação externa: clientes B2B que precisam acompanhar seus pedidos fazem polling contínuo no `GET /orders`. Esta feature adiciona **webhooks outbound**: quando o status de um pedido muda, o sistema notifica ativamente os endpoints HTTP cadastrados pelo cliente, em até 10 segundos, com payload assinado e garantia de entrega *at-least-once*.

A feature será construída sobre o ciclo de vida de pedidos já existente (máquina de estados em `src/modules/orders/order.status.ts` e transação de mudança de status em `src/modules/orders/order.service.ts`), sem alterar seu comportamento funcional.

## 2. Problema e motivação

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — formalizaram o pedido de notificação em tempo real de mudança de status de pedidos. Hoje eles fazem polling no `GET /orders` de tempos em tempos, o que torna a integração **lenta e cara** para eles.

O risco de negócio é concreto: a Atlas sinalizou que, sem essa entrega **até o fim do trimestre**, pode migrar para o concorrente. O prazo alvo comunicado é **fim de novembro**.

## 3. Público-alvo e cenários de uso

**Público-alvo:** times de integração dos clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo e futuros clientes com integração via API).

Cenários principais:

1. **Integração recebe mudanças de status:** o sistema do cliente é notificado quando um pedido dele muda de status (ex.: `PROCESSING → SHIPPED`), sem precisar fazer polling.
2. **Cadastro self-service via API:** um usuário autenticado cadastra, edita, lista e remove endpoints de webhook para um customer, escolhendo quais status quer receber (ex.: "só `SHIPPED` e `DELIVERED`").
3. **Auditoria de entregas:** o cliente consulta o histórico das últimas entregas (sucesso/falha, payload, response, tempo de resposta) para depurar a integração dele.
4. **Segurança da integração:** o cliente valida com HMAC que a requisição veio de nós e não foi adulterada, e rotaciona a secret quando necessário (ex.: vazamento em log).
5. **Operação (admin):** um usuário `ADMIN` reprocessa manualmente eventos que falharam permanentemente (DLQ).

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
|---|---|---|
| Notificar mudanças de status em "tempo real" na percepção do cliente | Latência entre commit da mudança de status e entrega do webhook | **≤ 10 segundos** (piso de ~2s imposto pelo polling do worker) |
| Eliminar o polling dos clientes no `GET /orders` | Volume de polling dos 3 clientes piloto após adoção | Redução observável do polling após a migração dos 3 clientes |
| Entrega confiável mesmo com cliente instável | Janela de retry antes de falha permanente | ~15 horas entre a primeira falha e a última tentativa (5 tentativas) |
| Cumprir o prazo comercial | Data de disponibilização em produção | Fim de novembro (**3 sprints**, incluindo revisão de segurança) |

## 5. Escopo

### Incluso

- Evento `order.status_changed` disparado a cada mudança de status de pedido.
- Registro do evento via **padrão outbox** na mesma transação MySQL da mudança de status.
- **Worker em processo separado** (polling de 2s) que entrega os eventos via HTTP POST.
- CRUD de configuração de webhooks por customer (criar, editar, remover, listar), com filtro por lista de status.
- Assinatura **HMAC-SHA256** com secret única por endpoint, rotação com grace period de 24h.
- Retry com backoff exponencial (5 tentativas) e **DLQ** em tabela separada.
- Endpoint de histórico de entregas por webhook.
- Endpoint admin de replay de DLQ (role `ADMIN`, com log de auditoria).
- Validação de URL `https` obrigatória e limite de payload de 64KB.

### Fora de escopo (descartado ou adiado na reunião)

| Item | Decisão na reunião |
|---|---|
| **Notificação por email quando o webhook do cliente falha repetidamente** | Adiado para próxima fase, "depois que a gente medir o impacto" ([09:37] Larissa) |
| **Rate limiting de envio para o cliente** | Não entra agora; "observar e decidir depois" ([09:39] Diego/Larissa) |
| **Dashboard/painel visual para o cliente** | Fora de escopo; painel é projeto separado do time de frontend ([09:40] Larissa) |
| **Arquivamento de linhas entregues da outbox** (após ~30 dias) | Explicitamente fora do escopo desta feature ([09:08] Diego) |
| **Webhooks inbound (cliente → plataforma)** | Escopo restrito a outbound ([09:02] Marcos) |
| **Escala para múltiplos workers** (particionamento por `order_id` / lock pessimista) | "Problema do futuro, não agora" ([09:13] Diego) |

## 6. Requisitos funcionais

| ID | Requisito |
|---|---|
| RF-01 | Cadastrar webhook via `POST`: informa `url` e lista de status desejados; a **secret é gerada pelo sistema** e devolvida na criação; `customer_id` vem no body/path (não do JWT, pois o JWT é do usuário operador). |
| RF-02 | Editar configuração de webhook via `PATCH`. |
| RF-03 | Remover webhook via `DELETE`. |
| RF-04 | Listar os webhooks de um customer via `GET`. |
| RF-05 | Filtrar eventos por webhook: cada endpoint escolhe a lista de status que quer receber; o filtro é aplicado **na inserção da outbox** (se nenhum webhook do customer assina aquele status, o evento nem é inserido). |
| RF-06 | Consultar histórico de entregas via `GET /webhooks/:id/deliveries`: últimas entregas com sucesso/falha, payload, response e tempo de resposta. |
| RF-07 | Disparar evento `order.status_changed` a cada mudança de status de pedido, registrado na outbox **dentro da mesma transação** que atualiza `orders` e `order_status_history`. |
| RF-08 | Entregar o evento por HTTP POST na URL cadastrada com headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`. |
| RF-09 | Rotacionar secret via endpoint dedicado: nova secret gerada, antiga permanece válida por **24 horas** em paralelo. |
| RF-10 | Reprocessar evento da DLQ via `POST /admin/webhooks/dead-letter/:id/replay` (recoloca na outbox como pendente), restrito à role `ADMIN` e com log de auditoria de quem executou. |
| RF-11 | Recusar cadastro de URL não-`https` com erro de validação (schema Zod). |
| RF-12 | Após 5 tentativas falhas, mover o evento para a DLQ (tabela separada com payload, motivo da falha e timestamp). |

## 7. Requisitos não funcionais

| ID | Requisito |
|---|---|
| RNF-01 | **Latência de entrega ≤ 10s** após a mudança de status (percepção de "tempo real" dos clientes); latência mínima de ~2s no pior caso, imposta pelo intervalo de polling. |
| RNF-02 | **Consistência transacional:** impossível o status mudar sem o evento ser registrado (e vice-versa) — outbox na mesma transação SQL. |
| RNF-03 | **Garantia at-least-once:** o cliente pode receber o mesmo evento mais de uma vez e deduplica pelo `X-Event-Id` (UUID único por evento). |
| RNF-04 | **Ordering:** garantido por `order_id` apenas enquanto houver um único worker (ordem por `created_at` da outbox). **Sem garantia de ordering global** — limitação conhecida e documentada. |
| RNF-05 | **Segurança:** HMAC-SHA256 sobre o corpo do request, secret única por endpoint, TLS obrigatório (`https`). |
| RNF-06 | **Timeout de entrega:** 10 segundos por chamada HTTP; estouro conta como falha e agenda retry. |
| RNF-07 | **Limite de payload:** 64KB; eventos acima disso geram erro (não são truncados nem enviados). |
| RNF-08 | **Isolamento de processo:** o worker roda em processo separado da API (restart da API não derruba a entrega de webhooks). |
| RNF-09 | **Não degradar a transação de pedidos:** nenhuma chamada HTTP dentro da transação de `changeStatus`. |

## 8. Decisões e trade-offs principais

Resumo — cada decisão está detalhada em seu ADR:

1. **Outbox no MySQL existente** em vez de fila externa (Redis Streams): consistência transacional sem infra nova; trade-off: acoplamento ao MySQL e necessidade de worker próprio. → [ADR-001](./adrs/ADR-001-padrao-outbox-no-mysql.md)
2. **Worker separado em polling de 2s** em vez de trigger/notificação do banco: simplicidade, atende o SLA de 10s; trade-off: latência mínima de 2s e queries contínuas. → [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)
3. **Retry: 5 tentativas com backoff 1m/5m/30m/2h/12h + DLQ em tabela separada**: cobre indisponibilidades longas (~15h) sem evento pendurado para sempre. → [ADR-003](./adrs/ADR-003-retry-backoff-exponencial-dlq.md)
4. **HMAC-SHA256 com secret por endpoint e rotação com grace de 24h**: vazamento de uma secret não compromete os demais clientes. → [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
5. **At-least-once com `X-Event-Id`** em vez de exactly-once: padrão de mercado (Stripe, GitHub); trade-off: dedup fica no cliente. → [ADR-005](./adrs/ADR-005-at-least-once-x-event-id.md)
6. **Reuso máximo dos padrões do projeto** (`AppError`, Pino, error middleware, módulos, Zod, prefixo de código de erro): novo módulo indistinguível dos existentes. → [ADR-006](./adrs/ADR-006-reuso-padroes-existentes.md)

## 9. Dependências

- **Código existente:** transação de `changeStatus` em `src/modules/orders/order.service.ts` (ponto de integração crítico); máquina de estados de `src/modules/orders/order.status.ts` (define os status que geram eventos); `authenticate`/`requireRole` de `src/middlewares/auth.middleware.ts`; infraestrutura de erros (`src/shared/errors/`), logger Pino (`src/shared/logger/index.ts`) e error middleware.
- **Banco:** MySQL existente via Prisma (novas tabelas: configuração de webhooks, outbox, entregas, dead letter). Worker usa o mesmo banco/`DATABASE_URL` com instância própria de `PrismaClient` (processo separado).
- **Time:** revisão de segurança da Sofia (mínimo 2 dias úteis antes do deploy, foco em HMAC e geração de secret).
- **Externa:** documentação para os clientes no portal de desenvolvedor (Marcos), incluindo destaque para a semântica at-least-once/dedup.

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Cliente lento/fora do ar degrada a entrega para os demais | Média | Alto | Nenhuma chamada HTTP na transação; timeout de 10s por chamada; retry assíncrono com backoff; DLQ após 5 falhas |
| Cliente recebe evento duplicado e processa duas vezes | Média | Médio | Semântica at-least-once documentada em destaque no portal; `X-Event-Id` (UUID) para dedup no cliente |
| Vazamento de secret do cliente (já ocorreu: cliente logou a secret na aplicação dele) | Média | Alto | Secret única por endpoint (vazamento não afeta outros); rotação self-service com grace de 24h; revisão de segurança dedicada antes do deploy |
| Acúmulo de eventos na outbox degrada o worker | Baixa | Médio | Índices em status e `created_at`; leitura em batch pequeno apenas dos pendentes; arquivamento futuro (~30 dias) já previsto fora deste escopo |
| Perda de ordering se a operação escalar para múltiplos workers | Baixa (fase atual) | Médio | Single-worker nesta fase; limitação documentada; particionamento por `order_id` ou lock pessimista como caminho futuro |
| Prazo (fim de novembro) com risco de churn da Atlas | Média | Alto | Plano de 3 sprints com revisão de segurança embutida; PM confirma prazo com o cliente antecipadamente |

## 11. Critérios de aceitação

1. Ao mudar o status de um pedido, um webhook cadastrado (e que assina aquele status) recebe o evento em até 10 segundos.
2. Se a transação de mudança de status sofre rollback, nenhum evento é entregue; se commita, o evento é registrado (verificável na outbox).
3. O request entregue contém `X-Event-Id`, `X-Signature` (HMAC-SHA256 válido contra a secret do endpoint), `X-Timestamp` e `X-Webhook-Id`; o payload contém `event_id`, `event_type`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents` — sem itens do pedido.
4. Webhook com filtro de status não recebe eventos de status fora da lista (e o evento nem entra na outbox se nenhum webhook do customer o assina).
5. Com o endpoint do cliente fora do ar, o sistema executa exatamente 5 tentativas com espaçamento 1m/5m/30m/2h/12h e então move o evento para a DLQ com payload, motivo e timestamp.
6. `POST /admin/webhooks/dead-letter/:id/replay` só funciona com role `ADMIN`, recoloca o evento como pendente e registra em log quem executou; qualquer outra role recebe 403.
7. Cadastro de URL `http://` é recusado com erro de validação; após rotação de secret, assinaturas com a secret antiga validam por 24h e depois são rejeitadas.
8. `GET /webhooks/:id/deliveries` retorna as entregas com sucesso/falha, payload, response e tempo de resposta.
9. Reinício da API não interrompe o processamento do worker (processos independentes).

## 12. Estratégia de testes e validação

- **Testes de integração** seguindo o padrão existente (`tests/orders.test.ts` com Vitest + Supertest): CRUD de webhooks, validação de URL https, geração/rotação de secret, autorização do endpoint admin.
- **Teste transacional:** mudança de status insere na outbox na mesma transação; falha na inserção da outbox faz rollback da mudança de status (cenário explicitado na reunião: "se a outbox falhar de inserir, rollback").
- **Testes do worker:** processamento de pendentes em ordem de `created_at`, cálculo do agendamento de retry, movimentação para DLQ após a 5ª falha, replay recolocando como pendente.
- **Testes de assinatura:** HMAC-SHA256 verificável com a secret; validade dupla durante o grace de 24h da rotação.
- **Teste ponta a ponta** (previsto no plano de sprints: "integração no order.service e testes ponta a ponta"): mudança de status → outbox → worker → entrega HTTP em servidor fake, medindo latência < 10s.
- **Revisão de segurança manual** pela Sofia (≥ 2 dias úteis) antes do deploy, com foco em HMAC e geração de secret.
