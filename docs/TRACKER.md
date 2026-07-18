# Tracker de Rastreabilidade

Cada linha mapeia um item registrado nos documentos do pacote à sua origem: a transcrição da reunião (`TRANSCRICAO.md`, formato `[hh:mm] Nome`) ou o código-fonte da aplicação (caminho do arquivo).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de 3 clientes B2B (Atlas, MaxDistribuição, Nova Cargo) por notificação em tempo real; hoje fazem polling no GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Risco de negócio | Atlas pode migrar para o concorrente se não entregarmos até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | Escopo é apenas outbound: webhooks saem da plataforma para o cliente, nunca o contrário | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | "Tempo real" = latência de entrega abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Prazo: fim de novembro, estimado em 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:45] Marcos / [09:46] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook via POST: url + lista de status; secret gerada pelo sistema e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar webhook via PATCH | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover webhook via DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer via GET | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos: lista de status assinados por webhook, aplicado na inserção da outbox | TRANSCRICAO | [09:33] Marcos / [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico de entregas (GET /webhooks/:id/deliveries): sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Evento registrado na outbox dentro da mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Entrega HTTP com headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego / [09:44] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret via API; antiga válida por 24h em paralelo | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ via POST /admin/webhooks/dead-letter/:id/replay, role ADMIN, com log de auditoria | TRANSCRICAO | [09:35] Diego / [09:36] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | URL não-https recusada com erro de validação (schema Zod) | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Após 5 tentativas falhas, evento vai para DLQ em tabela separada (payload, motivo, timestamp) | TRANSCRICAO | [09:17] Larissa / [09:18] Diego |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência mínima de ~2s aceita (pior caso do polling) | TRANSCRICAO | [09:10] Larissa |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Consistência transacional: commit ⇒ evento registrado; rollback ⇒ evento some junto | TRANSCRICAO | [09:06] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | At-least-once: cliente pode receber duplicado e deduplica por X-Event-Id | TRANSCRICAO | [09:24] Diego / [09:25] Diego |
| PRD-RNF-04 | docs/PRD.md | Restrição | Ordering garantido só por order_id e só enquanto single-worker; sem ordering global (limitação documentada) | TRANSCRICAO | [09:12] Diego / [09:13] Larissa |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP; estouro = falha e retry | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB; acima disso gera erro, não trunca | TRANSCRICAO | [09:23] Sofia / [09:24] Diego |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado: restart da API não derruba a entrega | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-08 | docs/PRD.md | Restrição | Transação de mudança de status é pesada (orders + history + estoque); não pode receber HTTP call | TRANSCRICAO | [09:04] Bruno |
| PRD-ESC-01 | docs/PRD.md | Fora de escopo | Email de aviso de falha do webhook: adiado para próxima fase, após medir impacto | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-02 | docs/PRD.md | Fora de escopo | Rate limiting de envio: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-ESC-03 | docs/PRD.md | Fora de escopo | Dashboard visual: projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-04 | docs/PRD.md | Fora de escopo | Arquivamento das linhas entregues (~30 dias) fora do escopo da feature | TRANSCRICAO | [09:08] Diego |
| PRD-ESC-05 | docs/PRD.md | Fora de escopo | Escala para múltiplos workers (particionamento/lock): problema do futuro | TRANSCRICAO | [09:13] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente já vazou secret em log de aplicação (motiva secret por endpoint + rotação) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente já teve indisponibilidade de 2h em manutenção planejada (motiva janela de retry longa) | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-03 | docs/PRD.md | Trade-off | Cliente que ficar >15h fora perde eventos até replay manual; aceito pelo PM | TRANSCRICAO | [09:17] Marcos |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia: mínimo 2 dias úteis antes do deploy (HMAC e geração de secret) | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação da semântica at-least-once em destaque no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Transação changeStatus existente como ponto de captura do evento | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | Arquitetura: outbox + worker polling + retry/DLQ + HMAC + at-least-once (resumo consolidado pela tech lead) | TRANSCRICAO | [09:48] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no service de orders: cliente lento travaria mudanças de status; rollback por cliente fora do ar inaceitável | TRANSCRICAO | [09:04] Bruno / [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila externa: infra nova, overengineering para time pequeno | TRANSCRICAO | [09:07] Larissa / [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger do MySQL para reatividade: não notifica processo externo (sem NOTIFY/LISTEN), improviso esquisito | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry indefinido (evento pendurado para sempre) e 3 tentativas (mata cliente com indisponibilidade de horas) | TRANSCRICAO | [09:15] Diego / [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Exactly-once: exigiria coordenação dos dois lados, complexidade muito maior | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global da plataforma: "se vaza uma, vaza tudo" | TRANSCRICAO | [09:21] Sofia |
| RFC-QA-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar em produção e implementar se virar problema | TRANSCRICAO | [09:38] Diego / [09:39] Larissa |
| RFC-QA-02 | docs/RFC.md | Questão em aberto | Notificação por email de falhas repetidas: próxima fase | TRANSCRICAO | [09:37] Marcos / [09:37] Larissa |
| RFC-QA-03 | docs/RFC.md | Questão em aberto | Escala do worker (particionamento por order_id ou lock pessimista) | TRANSCRICAO | [09:13] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em aberto | Endurecer autorização do CRUD de webhooks no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-05 | docs/RFC.md | Questão em aberto | Arquivamento da outbox após ~30 dias | TRANSCRICAO | [09:08] Diego |
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL existente, inserção na mesma transação SQL | TRANSCRICAO | [09:06] Diego / [09:08] Larissa |
| ADR-001-A | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Detalhe de decisão | Índices em status (pendente/processando/falhou/entregue) e created_at; leitura em batch pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-B | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Chave primária UUID, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-C | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Payload guardado como snapshot renderizado na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker em polling de 2s sobre os pendentes mais antigos | TRANSCRICAO | [09:09] Diego / [09:10] Larissa |
| ADR-002-A | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker como processo separado; entry point src/worker.ts + npm run worker | TRANSCRICAO | [09:11] Diego / [09:11] Larissa |
| ADR-002-B | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Mesmo banco e stack, instância própria de PrismaClient (client é por processo) | TRANSCRICAO | [09:11] Diego / [09:30] Bruno |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h; depois DLQ | TRANSCRICAO | [09:17] Diego / [09:17] Larissa |
| ADR-003-A | docs/adrs/ADR-003-retry-backoff-exponencial-dlq.md | Decisão | DLQ em tabela webhook_dead_letter separada (payload, motivo, timestamp) | TRANSCRICAO | [09:18] Diego |
| ADR-003-B | docs/adrs/ADR-003-retry-backoff-exponencial-dlq.md | Decisão | Replay manual via endpoint admin que recoloca o evento como pendente | TRANSCRICAO | [09:18] Diego / [09:35] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo do request, assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-A | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Secret única por endpoint; rotação com grace period de 24h | TRANSCRICAO | [09:21] Sofia / [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | At-least-once com X-Event-Id (UUID gerado na entrada da outbox) para dedup no cliente | TRANSCRICAO | [09:25] Diego / [09:26] Larissa |
| ADR-005-A | docs/adrs/ADR-005-at-least-once-x-event-id.md | Trade-off | Dedup vira responsabilidade do cliente; padrão de mercado (Stripe, GitHub) | TRANSCRICAO | [09:25] Sofia / [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, schemas Zod, códigos de erro | TRANSCRICAO | [09:30] Larissa |
| ADR-006-A | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Módulo src/modules/webhooks com controller, service, repository, routes e schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-B | docs/adrs/ADR-006-reuso-padroes-existentes.md | Referência de código | Padrão de módulo replicado (controller/service/repository/routes/schemas por domínio) | CODIGO | src/modules/orders/ |
| ADR-006-C | docs/adrs/ADR-006-reuso-padroes-existentes.md | Referência de código | Classe base de erro com statusCode + errorCode simbólico a ser estendida | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-D | docs/adrs/ADR-006-reuso-padroes-existentes.md | Referência de código | Exemplos a seguir: InsufficientStockError, InvalidStatusTransitionError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-FLX-01 | docs/FDD.md | Fluxo | changeStatus estendido: inserção na outbox dentro da transação; falha na inserção ⇒ rollback | TRANSCRICAO | [09:40] Bruno / [09:41] Diego |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Integração via publishWebhookEvent(tx, order, fromStatus, toStatus) — função pura recebendo o tx client | TRANSCRICAO | [09:41] Bruno / [09:41] Diego |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Worker lê pendentes em ordem de created_at, processa, marca como entregue | TRANSCRICAO | [09:08] Diego / [09:12] Diego |
| FDD-FLX-04 | docs/FDD.md | Fluxo | Lógica do worker dentro do módulo (webhook.worker.ts ou webhook.processor.ts) | TRANSCRICAO | [09:28] Bruno |
| FDD-FLX-05 | docs/FDD.md | Fluxo | Transação existente que será estendida: update order + create history dentro de $transaction | CODIGO | src/modules/orders/order.service.ts |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks: url + statuses; customer_id no body/path (JWT é do usuário operador, não do cliente) | TRANSCRICAO | [09:31] Marcos / [09:32] Larissa |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | PATCH /webhooks/:id, DELETE /webhooks/:id, GET de listagem por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries com sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay (ADMIN, auditado) | TRANSCRICAO | [09:35] Diego / [09:36] Sofia |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | Payload do evento: event_id, event_type "order.status_changed", timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Headers do envio: X-Event-Id, X-Signature, X-Timestamp (detecção de replay attack), X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego / [09:44] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | CRUD de configuração com qualquer role autenticada (por enquanto) | TRANSCRICAO | [09:36] Marcos / [09:37] Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Envelope de erro { error: { code, message, details } } reutilizado do middleware central | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Listagens paginadas reutilizando paginated()/PaginatedResponse | CODIGO | src/shared/http/response.ts |
| FDD-ERR-01 | docs/FDD.md | Matriz de erros | Códigos com prefixo WEBHOOK_ (WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc.) | TRANSCRICAO | [09:28] Bruno / [09:29] Larissa |
| FDD-ERR-02 | docs/FDD.md | Matriz de erros | Middleware central já trata AppError, Zod e Prisma — erros novos pegos sem mudança | TRANSCRICAO | [09:29] Bruno |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10s; falha marca para retry | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Backoff 1m/5m/30m/2h/12h, ~15h de janela total | TRANSCRICAO | [09:17] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Limite de 64KB com erro (sem truncar) | TRANSCRICAO | [09:23] Sofia / [09:24] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logger Pino compartilhado com redact de campos sensíveis; base para logs estruturados do worker | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de auditoria do replay com identidade do admin executor | TRANSCRICAO | [09:36] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração | Máquina de estados como fonte dos status assináveis (canTransition/transitions) | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-02 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN') reutilizados nas rotas do módulo | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Schemas Zod aplicados via validate({ body, params, query }) como nos módulos existentes | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Registro do router novo no agregador de rotas (tipo Controllers + router.use) | CODIGO | src/routes/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | src/worker.ts espelha o bootstrap/shutdown gracioso do servidor HTTP | CODIGO | src/server.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Novos modelos seguem convenções do schema: UUID char(36), @@map snake_case, índices explícitos | CODIGO | prisma/schema.prisma |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Configuração de webhook: url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno / [09:21] Sofia |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Outbox com status pendente/processando/falhou/entregue, índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | Dead letter com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | Snapshot do payload renderizado na inserção (estado de quando o status mudou) | TRANSCRICAO | [09:52] Larissa |

## Cobertura

- **Total de linhas:** 98, cobrindo os itens identificáveis de PRD, RFC, FDD e ADRs (> 80%).
- **Fonte TRANSCRICAO:** 84 linhas (~86%), todas com timestamp `[hh:mm] Nome` válido.
- **Fonte CODIGO:** 14 linhas, todas apontando para arquivos reais do repositório.
- Itens dos documentos sem linha própria são elaborações diretas dos itens acima (ex.: nomes de campos do Prisma, textos de exemplo de payload, métricas derivadas dos SLAs), sinalizados como proposta de implementação nos próprios documentos.
