# Plano de Implementação: SRE Alert Analyzer

## Visão Geral

Implementação incremental do agente SRE Alert Analyzer em Node.js, integrado ao projeto Express existente. Cada tarefa constrói sobre a anterior, culminando na integração completa de todos os componentes. O agente recebe alertas do CloudWatch, coleta contexto de logs, métricas e eventos Kubernetes, correlaciona os dados e gera sugestões de causa raiz — sem executar nenhuma ação de remediação.

## Tarefas

- [ ] 1. Estrutura base, tipos e interfaces TypeScript
  - Criar diretório `src/sre-alert-analyzer/` com subdiretórios: `types/`, `ingestor/`, `collector/`, `correlation/`, `suggestion/`, `notifier/`, `history/`, `audit/`, `observability/`
  - Criar `src/sre-alert-analyzer/types/index.ts` com todas as interfaces: `RawAlert`, `Alert`, `LogEntry`, `MetricDataPoint`, `KubernetesEvent`, `PodState`, `ContextCollectionResult`, `CorrelationResult`, `Correlation`, `AnomalousMetric`, `CriticalPattern`, `Evidence`, `Suggestion`, `AnalysisResult`, `AnalysisRecord`, `NotificationResult`, `CollectionError`, `NotificationError`, `CollectionConfig`
  - Criar `src/sre-alert-analyzer/types/audit.ts` com a interface de entrada de auditoria
  - Instalar dependências: `@aws-sdk/client-cloudwatch-logs`, `@aws-sdk/client-cloudwatch`, `@aws-sdk/client-s3`, `@aws-sdk/client-secrets-manager`, `@kubernetes/client-node`, `p-queue`, `p-retry`, `fast-check` (devDependency)
  - _Requisitos: 1.2, 2.1, 3.1, 4.1, 5.1, 6.1, 6.2, 7.1, 8.2_

- [ ] 2. Alert Ingestor — validação e enfileiramento
  - [ ] 2.1 Implementar `src/sre-alert-analyzer/ingestor/alertIngestor.ts`
    - Função `validateAlert(raw: RawAlert): Alert` que valida presença de `id`, `severity` e `timestamp`; lança erro descritivo se ausentes
    - Função `parseAlert(raw: RawAlert): Alert` que mapeia o payload CloudWatch para a interface `Alert`, adicionando `receivedAt`
    - Classe `AlertQueue` usando `p-queue` com concorrência configurável via variável de ambiente `ALERT_QUEUE_CONCURRENCY` (padrão: 5)
    - Método `enqueue(raw: RawAlert): Promise<void>` que valida, parseia e enfileira o alerta
    - _Requisitos: 1.2, 1.3, 1.4_

  - [ ]* 2.2 Escrever property test para rejeição de alertas inválidos
    - **Propriedade 6: Rejeição de Alertas Inválidos**
    - **Valida: Requisito 1.3**
    - Usar `fc.record` com campos `id`, `severity`, `timestamp` opcionalmente ausentes para garantir que `validateAlert` sempre lança erro quando qualquer campo obrigatório está faltando e nunca gera Suggestion

  - [ ]* 2.3 Escrever testes unitários para o Alert Ingestor
    - Testar parsing de payload CloudWatch com campos opcionais presentes e ausentes
    - Testar que alertas inválidos são registrados sem interromper o processamento de outros alertas
    - _Requisitos: 1.3, 1.4_

- [ ] 3. Checkpoint — Ingestor funcional
  - Garantir que todos os testes passam. Verificar que `validateAlert` rejeita corretamente entradas inválidas e que `AlertQueue` enfileira alertas válidos. Perguntar ao usuário se há dúvidas antes de continuar.

- [ ] 4. Audit Logger
  - [ ] 4.1 Implementar `src/sre-alert-analyzer/audit/auditLogger.ts`
    - Função `logAuditEntry(entry: AuditEntry): void` que grava no CloudWatch Logs (ou stdout em desenvolvimento) no formato JSON especificado no design
    - Implementar mascaramento automático de credenciais: substituir valores de campos `token`, `password`, `apiKey`, `webhookUrl` por `[REDACTED]` antes de logar
    - _Requisitos: 9.4, 9.5_

  - [ ]* 4.2 Escrever testes unitários para o Audit Logger
    - Testar que credenciais nunca aparecem em texto plano nas entradas de auditoria
    - Testar formato JSON da entrada de auditoria
    - _Requisitos: 9.4, 9.5_

- [ ] 5. Context Collector — CloudWatch Logs
  - [ ] 5.1 Implementar `src/sre-alert-analyzer/collector/logsCollector.ts`
    - Classe `LogsCollector` usando `@aws-sdk/client-cloudwatch-logs`
    - Método `collectLogs(alert: Alert, config: CollectionConfig): Promise<LogEntry[]>` que chama `FilterLogEvents` com `filterPattern: "ERROR OR WARN OR FATAL"` dentro da `contextWindowMinutes`
    - Tratar erro de autenticação/autorização: registrar via `auditLogger` e retornar array vazio (não lançar exceção)
    - Registrar ausência de logs quando a consulta retornar resultado vazio
    - _Requisitos: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ]* 5.2 Escrever property test para filtro de logs por severidade
    - **Propriedade 7: Filtro de Logs por Severidade**
    - **Valida: Requisito 2.4**
    - Usar `fc.array(fc.record({ level: fc.constantFrom('ERROR', 'WARN', 'FATAL', 'INFO', 'DEBUG') }))` para garantir que o filtro retorna exclusivamente entradas com nível `ERROR`, `WARN` ou `FATAL`

  - [ ]* 5.3 Escrever testes unitários para o Logs Collector
    - Testar `contextWindowMinutes` configurável (1 a 1440 minutos)
    - Testar comportamento quando CloudWatch Logs retorna erro
    - _Requisitos: 2.2, 2.3, 2.5_

- [ ] 6. Context Collector — CloudWatch Metrics
  - [ ] 6.1 Implementar `src/sre-alert-analyzer/collector/metricsCollector.ts`
    - Classe `MetricsCollector` usando `@aws-sdk/client-cloudwatch`
    - Método `collectMetrics(alert: Alert, config: CollectionConfig): Promise<MetricDataPoint[]>` que coleta `CPUUtilization`, `MemoryUtilization`, `Latency`, `HTTPCode_ELB_5XX_Count` via `GetMetricData`
    - Calcular média e desvio padrão das últimas 24h para cada métrica; marcar `isAnomalous = true` quando valor atual > 2σ da média
    - Tratar erros retornando array vazio e registrando via `auditLogger`
    - _Requisitos: 3.1, 3.2, 3.3, 3.4_

  - [ ]* 6.2 Escrever testes unitários para o Metrics Collector
    - Testar cálculo de desvio padrão e detecção de anomalias (valor > 2σ)
    - Testar comportamento quando CloudWatch Metrics retorna erro
    - _Requisitos: 3.3, 3.4_

- [ ] 7. Context Collector — Kubernetes Events
  - [ ] 7.1 Implementar `src/sre-alert-analyzer/collector/k8sCollector.ts`
    - Classe `K8sCollector` usando `@kubernetes/client-node` com autenticação via ServiceAccount token (`/var/run/secrets/kubernetes.io/serviceaccount/token`)
    - Método `collectEvents(alert: Alert, config: CollectionConfig): Promise<KubernetesEvent[]>` que chama `listNamespacedEvent` filtrando por `Warning` e `Normal`
    - Método `collectPodStates(alert: Alert): Promise<PodState[]>` que chama `listNamespacedPod`
    - Marcar `isCriticalPattern = true` para eventos com `reason` em: `CrashLoopBackOff`, `OOMKilled`, `ImagePullBackOff`, `Evicted`, `NodeNotReady`
    - Tratar falha de conexão: registrar erro e retornar arrays vazios
    - _Requisitos: 4.1, 4.2, 4.3, 4.4, 4.5_

  - [ ]* 7.2 Escrever property test para detecção de padrões críticos Kubernetes
    - **Propriedade 8: Detecção de Padrões Críticos Kubernetes**
    - **Valida: Requisito 4.3**
    - Usar `fc.array(fc.record({ reason: fc.oneof(fc.constantFrom('CrashLoopBackOff', 'OOMKilled', 'ImagePullBackOff', 'Evicted', 'NodeNotReady'), fc.string()) }))` para garantir que todos os 5 padrões críticos são sempre marcados como `isCriticalPattern = true`

  - [ ]* 7.3 Escrever testes unitários para o K8s Collector
    - Testar coleta de eventos Warning e Normal
    - Testar comportamento quando a API Kubernetes está inacessível
    - _Requisitos: 4.2, 4.4_

- [ ] 8. Context Collector — Orquestrador paralelo
  - [ ] 8.1 Implementar `src/sre-alert-analyzer/collector/contextCollector.ts`
    - Classe `ContextCollector` que orquestra `LogsCollector`, `MetricsCollector` e `K8sCollector`
    - Método `collect(alert: Alert, config: CollectionConfig): Promise<ContextCollectionResult>` usando `Promise.allSettled` para execução paralela
    - Erros parciais de qualquer coletor são capturados em `errors[]` sem interromper a coleta das demais fontes
    - Registrar via `auditLogger` cada consulta externa realizada (fonte, duração, status)
    - _Requisitos: 2.1, 3.1, 4.1, 9.5_

  - [ ]* 8.2 Escrever testes de integração para o Context Collector
    - Testar que falha em uma fonte não interrompe coleta das demais (mock de CloudWatch Logs retornando erro)
    - Testar que `errors[]` é populado corretamente quando fontes falham
    - _Requisitos: 2.3, 3.4, 4.4_

- [ ] 9. Checkpoint — Coleta de contexto funcional
  - Garantir que todos os testes passam. Verificar que `Promise.allSettled` isola falhas parciais corretamente. Perguntar ao usuário se há dúvidas antes de continuar.

- [ ] 10. Correlation Engine
  - [ ] 10.1 Implementar `src/sre-alert-analyzer/correlation/correlationEngine.ts`
    - Função pura `correlate(context: ContextCollectionResult, alert: Alert): CorrelationResult` (sem efeitos colaterais, sem estado externo)
    - Correlacionar por timestamp e `service_id`; identificar sequências temporais nos 10 minutos anteriores ao alerta
    - Marcar correlação como `HIGH_RELEVANCE` quando anomalia de métrica + evento crítico K8s ocorrem em janela de 5 minutos
    - Associar logs `ERROR`/`FATAL` a eventos K8s no mesmo intervalo de tempo
    - Quando nenhuma correlação for encontrada, retornar `CorrelationResult` com `hasCorrelations: false` e `correlations: []`
    - _Requisitos: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [ ]* 10.2 Escrever property test para idempotência da correlação
    - **Propriedade 4: Idempotência da Correlação**
    - **Valida: Requisitos 5.1, 5.2, 5.3, 5.4**
    - Usar `fc.record({ logs: fc.array(...), metrics: fc.array(...), k8sEvents: fc.array(...) })` para garantir que `correlate(data) === correlate(correlate(data))` — mesmos dados produzem sempre o mesmo resultado

  - [ ]* 10.3 Escrever testes unitários para o Correlation Engine
    - Testar identificação de sequência temporal nos 10 minutos anteriores ao alerta
    - Testar marcação de `HIGH_RELEVANCE` quando anomalia de métrica + evento crítico K8s em janela de 5 minutos
    - Testar comportamento quando nenhuma correlação é encontrada
    - _Requisitos: 5.2, 5.3, 5.5_

- [ ] 11. Suggestion Generator
  - [ ] 11.1 Implementar `src/sre-alert-analyzer/suggestion/confidenceScorer.ts`
    - Função pura `calculateConfidenceScore(correlation: CorrelationResult): number` com as regras: +20 por correlação identificada, +15 por padrão crítico K8s, +10 por anomalia de métrica, +10 por correspondência histórica; clampado em [0, 100]
    - _Requisitos: 6.2_

  - [ ]* 11.2 Escrever property test para invariante do Confidence_Score
    - **Propriedade 1: Invariante do Confidence_Score**
    - **Valida: Requisito 6.2**
    - Usar `fc.record({ correlations: fc.array(...), anomalousMetrics: fc.array(...), criticalPatterns: fc.array(...) })` para garantir que `calculateConfidenceScore` sempre retorna valor em [0, 100]

  - [ ] 11.3 Implementar `src/sre-alert-analyzer/suggestion/suggestionGenerator.ts`
    - Classe `SuggestionGenerator` com método `generate(alert: Alert, correlation: CorrelationResult, historical?: HistoricalIncident[]): AnalysisResult`
    - Gerar ao menos uma `Suggestion` para qualquer alerta válido, mesmo sem correlações
    - Ordenar `suggestions[]` por `confidence_score` decrescente
    - Definir `isInconclusive = true` quando o score da Suggestion principal for < 40; incluir indicação explícita de investigação manual na `Suggestion`
    - Incluir `historicalReference` na Suggestion quando `historical` contiver incidente similar
    - `recommendedActions` deve conter apenas strings textuais — nunca comandos executáveis
    - _Requisitos: 6.1, 6.3, 6.4, 6.5, 6.6, 6.7_

  - [ ]* 11.4 Escrever property test para ordenação das Suggestions
    - **Propriedade 2: Ordenação das Suggestions**
    - **Valida: Requisito 6.3**
    - Usar `fc.array(fc.integer({ min: 0, max: 100 }), { minLength: 2 })` para garantir que a lista de Suggestions está sempre ordenada do maior para o menor `confidence_score`

  - [ ]* 11.5 Escrever property test para completude das Suggestions
    - **Propriedade 5: Completude das Suggestions**
    - **Valida: Requisitos 6.1, 6.5**
    - Usar `fc.record(Alert válido)` para garantir que `generate` sempre retorna ao menos uma Suggestion, mesmo quando `correlation.hasCorrelations = false`

  - [ ]* 11.6 Escrever property test para análise inconclusiva
    - **Propriedade 10: Análise Inconclusiva quando Score < 40**
    - **Valida: Requisito 6.4**
    - Usar `fc.integer({ min: 0, max: 39 })` para garantir que quando o score principal é < 40, `isInconclusive = true` e a Suggestion contém indicação de investigação manual

  - [ ]* 11.7 Escrever testes unitários para o Suggestion Generator
    - Testar geração de Suggestion quando não há correlações
    - Testar inclusão de `historicalReference` quando incidente similar existe
    - _Requisitos: 6.1, 6.6_

- [ ] 12. Checkpoint — Motor de análise funcional
  - Garantir que todos os testes passam. Verificar que o pipeline Ingestor → Collector → Correlation → Suggestion funciona de ponta a ponta com dados mockados. Perguntar ao usuário se há dúvidas antes de continuar.

- [ ] 13. History Manager — Amazon S3
  - [ ] 13.1 Implementar `src/sre-alert-analyzer/history/historyManager.ts`
    - Classe `HistoryManager` usando `@aws-sdk/client-s3`
    - Método `save(record: AnalysisRecord): Promise<void>` com `PutObject` no path `analyses/{yyyy}/{MM}/{dd}/{alert_id}.json`; usar `p-retry` com 3 tentativas e intervalo fixo de 5 segundos
    - Método `findById(alertId: string): Promise<AnalysisRecord | null>` com `GetObject`
    - Método `findSimilar(alert: Alert): Promise<HistoricalIncident[]>` para buscar incidentes anteriores similares por `service` e `severity`
    - Registrar via `auditLogger` cada operação S3 (WRITE/READ, status, duração)
    - _Requisitos: 8.1, 8.2, 8.3, 8.4_

  - [ ]* 13.2 Escrever property test para path de armazenamento no S3
    - **Propriedade 9: Path de Armazenamento no S3**
    - **Valida: Requisito 8.1**
    - Usar `fc.record({ date: fc.date(), alertId: fc.string({ minLength: 1 }) })` para garantir que o path gerado segue exatamente o padrão `analyses/{yyyy}/{MM}/{dd}/{alert_id}.json`

  - [ ]* 13.3 Escrever property test para round-trip de serialização do Analysis_Record
    - **Propriedade 3: Round-Trip de Serialização do Analysis_Record**
    - **Valida: Requisitos 8.1, 8.2**
    - Usar `fc.record(AnalysisRecord arbitrário)` para garantir que `JSON.parse(JSON.stringify(record))` produz objeto estruturalmente equivalente ao original, sem perda de campos ou alteração de valores

  - [ ]* 13.4 Escrever testes unitários para o History Manager
    - Testar retry automático: 3 tentativas com intervalo de 5s quando `PutObject` falha
    - Testar que `findById` retorna `null` quando o registro não existe no S3
    - _Requisitos: 8.3, 8.4_

- [ ] 14. Notifier — Jira e Microsoft Teams
  - [ ] 14.1 Implementar `src/sre-alert-analyzer/notifier/secretsProvider.ts`
    - Classe `SecretsProvider` usando `@aws-sdk/client-secrets-manager`
    - Método `getJiraCredentials(): Promise<{ email: string; apiToken: string; baseUrl: string; projectKey: string }>` com cache de 5 minutos
    - Método `getTeamsWebhookUrl(): Promise<string>` com cache de 5 minutos
    - Nunca logar valores de credenciais; usar `[REDACTED]` em qualquer saída de log
    - _Requisitos: 9.3, 9.4_

  - [ ] 14.2 Implementar `src/sre-alert-analyzer/notifier/jiraNotifier.ts`
    - Classe `JiraNotifier` com método `createTicket(alert: Alert, result: AnalysisResult): Promise<{ ticketId: string; ticketUrl: string }>`
    - Mapear severidade do alerta para prioridade Jira: `CRITICAL` → `Highest`, `HIGH` → `High`, `MEDIUM` → `Medium`, `LOW` → `Low`
    - Campos do ticket: `summary`, `description` (análise completa em Markdown), `priority`, `labels: ['sre-alert-analyzer']`
    - Falhas são capturadas e retornadas como `NotificationError` — nunca lançadas
    - _Requisitos: 7.3, 7.4_

  - [ ] 14.3 Implementar `src/sre-alert-analyzer/notifier/teamsNotifier.ts`
    - Classe `TeamsNotifier` com método `sendMessage(alert: Alert, result: AnalysisResult, jiraTicketUrl?: string): Promise<boolean>`
    - Formatar Adaptive Card com: resumo do alerta, `confidence_score` da Suggestion principal, link do Jira Ticket
    - Falhas são capturadas e retornadas como `false` — nunca lançadas
    - _Requisitos: 7.5, 7.6_

  - [ ] 14.4 Implementar `src/sre-alert-analyzer/notifier/notifier.ts`
    - Classe `Notifier` que orquestra `JiraNotifier` e `TeamsNotifier`
    - Método `notify(alert: Alert, result: AnalysisResult): Promise<NotificationResult>` executando Jira e Teams em paralelo (`Promise.allSettled`)
    - Registrar via `auditLogger` cada notificação enviada
    - _Requisitos: 7.3, 7.4, 7.5, 7.6_

  - [ ]* 14.5 Escrever testes unitários para o Notifier
    - Testar mapeamento de severidade para prioridade Jira
    - Testar formatação da Adaptive Card para Teams
    - Testar que falha no Jira não impede envio para Teams e vice-versa
    - _Requisitos: 7.3, 7.4, 7.5, 7.6_

- [ ] 15. Observabilidade — Health Check e Métricas
  - [ ] 15.1 Implementar `src/sre-alert-analyzer/observability/metricsCollector.ts`
    - Classe `AgentMetrics` com contadores: `alerts_processed_total`, `analysis_errors_total`, `inconclusive_analyses_total`
    - Histograma `analysis_duration_seconds` para tempo médio de análise
    - Gauge `external_query_errors_rate` calculado em janela deslizante de 5 minutos
    - Método `toPrometheusFormat(): string` e `toJSON(): object`
    - _Requisitos: 10.1_

  - [ ] 15.2 Implementar `src/sre-alert-analyzer/observability/healthCheck.ts`
    - Função `getHealthStatus(): { status: 'healthy' | 'degraded' | 'unhealthy'; checks: object }`
    - Status `degraded` quando taxa de erros externos > 20% em 5 minutos
    - Status `unhealthy` quando o agente não consegue processar novos alertas
    - _Requisitos: 10.3, 10.4_

  - [ ]* 15.3 Escrever testes unitários para Observabilidade
    - Testar que `/health` retorna JSON válido com campo `status`
    - Testar que `AgentMetrics` incrementa contadores corretamente
    - _Requisitos: 10.1, 10.4_

- [ ] 16. Orquestrador principal e timeout
  - [ ] 16.1 Implementar `src/sre-alert-analyzer/analyzer.ts`
    - Classe `AlertAnalyzer` que orquestra o pipeline completo: Ingestor → ContextCollector → CorrelationEngine → HistoryManager.findSimilar → SuggestionGenerator → Notifier → HistoryManager.save
    - Método `analyze(raw: RawAlert): Promise<AnalysisRecord>` com timeout de 2 minutos via `Promise.race`
    - Quando timeout ocorre: registrar evento de timeout, marcar `AnalysisRecord.status = 'INCOMPLETE'`, registrar via `auditLogger`
    - Emitir alerta operacional para Teams de monitoramento quando taxa de erros externos > 20% em janela de 5 minutos
    - _Requisitos: 7.2, 10.2, 10.3_

  - [ ]* 16.2 Escrever testes de integração para o Analyzer
    - Testar fluxo completo com mocks de CloudWatch, EKS, Jira e Teams
    - Testar comportamento de resiliência quando fontes externas retornam erro
    - Testar que múltiplos alertas simultâneos são processados sem perda de dados
    - _Requisitos: 1.4, 7.2, 10.2_

- [ ] 17. Rotas Express — Webhook e Observabilidade
  - [ ] 17.1 Criar `src/sre-alert-analyzer/routes/alertRoutes.ts`
    - `POST /sre/alerts` — recebe payload CloudWatch, chama `AlertAnalyzer.analyze`, retorna `202 Accepted` imediatamente (processamento assíncrono)
    - Validar `Content-Type: application/json`; retornar `400 Bad Request` com mensagem descritiva para payloads inválidos
    - _Requisitos: 1.1, 1.3_

  - [ ] 17.2 Criar `src/sre-alert-analyzer/routes/observabilityRoutes.ts`
    - `GET /health` — retorna resultado de `getHealthStatus()` com status HTTP 200 (healthy/degraded) ou 503 (unhealthy)
    - `GET /metrics` — retorna `AgentMetrics.toPrometheusFormat()` com `Content-Type: text/plain; version=0.0.4` ou JSON se `Accept: application/json`
    - _Requisitos: 10.4_

  - [ ] 17.3 Registrar rotas no `app.js` existente
    - Importar e montar `alertRoutes` em `/sre/alerts` e `observabilityRoutes` em `/`
    - _Requisitos: 1.1, 10.4_

  - [ ]* 17.4 Escrever smoke tests para os endpoints
    - Testar que `GET /health` retorna JSON com campo `status`
    - Testar que `GET /metrics` retorna campos esperados
    - Testar que `POST /sre/alerts` com payload inválido retorna `400`
    - _Requisitos: 1.3, 10.4_

- [ ] 18. Checkpoint final — Integração completa
  - Garantir que todos os testes passam. Verificar que o fluxo completo funciona: `POST /sre/alerts` → análise → Jira + Teams + S3. Confirmar que nenhuma credencial aparece em logs. Perguntar ao usuário se há dúvidas antes de finalizar.

## Notas

- Tarefas marcadas com `*` são opcionais e podem ser puladas para um MVP mais rápido
- Cada tarefa referencia requisitos específicos para rastreabilidade
- Os checkpoints garantem validação incremental a cada fase
- O Correlation Engine é implementado como função pura para garantir a Propriedade 4 (idempotência) e facilitar property-based testing sem mocks
- Credenciais nunca devem aparecer em logs — o Audit Logger aplica mascaramento automático
- O agente **nunca executa** ações de remediação; `recommendedActions` contém apenas strings textuais
- Usar `Promise.allSettled` (não `Promise.all`) em todas as operações paralelas para garantir resiliência a falhas parciais
