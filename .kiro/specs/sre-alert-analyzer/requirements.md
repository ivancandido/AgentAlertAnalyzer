# Documento de Requisitos

## Introdução

O **SRE Alert Analyzer** é um agente inteligente de análise automática de alertas para ambientes cloud, atuando como um Site Reliability Engineer (SRE) virtual. O agente recebe alertas provenientes do **CloudWatch (AWS)** como fonte principal, correlaciona automaticamente com logs, métricas e eventos recentes do cluster Kubernetes (EKS), e **sugere** a causa raiz do problema — sem executar nenhuma ação automaticamente. O objetivo é reduzir o tempo médio de resolução (MTTR) e o esforço manual das equipes de operações, mantendo o controle humano sobre todas as decisões de remediação.

## Glossário

- **Agent**: O sistema SRE Alert Analyzer responsável por orquestrar a análise de alertas.
- **Alert**: Notificação de evento anômalo recebida do CloudWatch.
- **CloudWatch**: Serviço de monitoramento e observabilidade da AWS — fonte principal de alertas.
- **EKS**: Amazon Elastic Kubernetes Service — serviço gerenciado de Kubernetes na AWS.
- **Cluster**: Conjunto de nós Kubernetes gerenciados pelo EKS.
- **Log**: Registro textual de eventos gerado por aplicações ou infraestrutura.
- **Metric**: Medida quantitativa de desempenho ou estado de um recurso (ex: CPU, memória, latência).
- **Event**: Ocorrência registrada no Kubernetes (ex: CrashLoopBackOff, OOMKilled, pod eviction).
- **Root Cause**: Causa raiz identificada como origem do problema que gerou o alerta.
- **Correlation_Engine**: Componente do Agent responsável por correlacionar dados de múltiplas fontes.
- **Context_Window**: Janela de tempo configurável usada para buscar logs, métricas e eventos relevantes ao alerta.
- **Suggestion**: Hipótese de causa raiz gerada pelo Agent com base na correlação de dados. O Agent **nunca executa** ações — apenas gera Suggestions para revisão humana.
- **Confidence_Score**: Pontuação de 0 a 100 que indica o grau de confiança do Agent na Suggestion apresentada.
- **Analysis_Record**: Registro persistente de uma análise completa, armazenado no Amazon S3.
- **Jira_Ticket**: Ticket criado no Jira para registrar o alerta e a análise, garantindo rastreabilidade pelo time de operações.
- **Teams_Channel**: Canal do Microsoft Teams configurado para receber notificações de análise.

---

## Requisitos

### Requisito 1: Ingestão de Alertas

**User Story:** Como SRE, quero que o agente receba alertas do CloudWatch automaticamente, para que eu não precise iniciar manualmente a análise de cada incidente.

#### Critérios de Aceitação

1. WHEN um alerta é publicado no CloudWatch, THE Agent SHALL ingeri-lo em até 30 segundos após a publicação.
2. THE Agent SHALL extrair do alerta os campos: identificador, severidade, timestamp, serviço afetado, região AWS e descrição textual.
3. IF o alerta recebido não contiver os campos obrigatórios (identificador, severidade, timestamp), THEN THE Agent SHALL registrar o alerta como inválido e emitir uma mensagem de erro descritiva.
4. THE Agent SHALL suportar ingestão simultânea de múltiplos alertas sem perda de dados.

---

### Requisito 2: Coleta de Contexto — Logs

**User Story:** Como SRE, quero que o agente colete automaticamente os logs relevantes ao alerta, para que a análise seja baseada em evidências reais do ambiente.

#### Critérios de Aceitação

1. WHEN um alerta é ingerido, THE Agent SHALL consultar o CloudWatch Logs para recuperar entradas de log do serviço afetado dentro da Context_Window configurada.
2. THE Agent SHALL suportar uma Context_Window configurável entre 1 minuto e 24 horas, com valor padrão de 30 minutos.
3. WHEN logs do serviço afetado não estiverem disponíveis no CloudWatch Logs, THE Agent SHALL registrar a ausência e prosseguir a análise com os dados disponíveis.
4. THE Agent SHALL filtrar logs por nível de severidade (ERROR, WARN, FATAL) para priorizar entradas relevantes.
5. IF a consulta ao CloudWatch Logs retornar erro de autenticação ou autorização, THEN THE Agent SHALL registrar o erro e notificar o operador sem interromper a análise de outros alertas.

---

### Requisito 3: Coleta de Contexto — Métricas

**User Story:** Como SRE, quero que o agente colete métricas de infraestrutura e aplicação correlacionadas ao alerta, para que anomalias de desempenho sejam identificadas.

#### Critérios de Aceitação

1. WHEN um alerta é ingerido, THE Agent SHALL consultar o CloudWatch Metrics para recuperar métricas do serviço e dos nós EKS afetados dentro da Context_Window configurada.
2. THE Agent SHALL coletar, no mínimo, as métricas: utilização de CPU, utilização de memória, latência de requisições e taxa de erros HTTP.
3. WHEN uma métrica coletada exceder o limiar de 2 desvios padrão em relação à média histórica das últimas 24 horas, THE Agent SHALL marcar a métrica como anômala.
4. IF a consulta ao CloudWatch Metrics retornar erro, THEN THE Agent SHALL registrar o erro e prosseguir a análise sem as métricas indisponíveis.
5. WHERE a integração com o Amazon Managed Service for Prometheus estiver habilitada, THE Agent SHALL também coletar métricas do Prometheus para os pods do cluster EKS afetado.

---

### Requisito 4: Coleta de Contexto — Eventos do Cluster Kubernetes

**User Story:** Como SRE, quero que o agente colete eventos recentes do cluster EKS relacionados ao serviço afetado, para que falhas de orquestração sejam consideradas na análise.

#### Critérios de Aceitação

1. WHEN um alerta é ingerido, THE Agent SHALL consultar a API do Kubernetes para recuperar eventos do namespace e dos pods do serviço afetado dentro da Context_Window configurada.
2. THE Agent SHALL coletar eventos dos tipos: Warning e Normal, priorizando os do tipo Warning.
3. THE Agent SHALL identificar e registrar os seguintes padrões de eventos críticos: CrashLoopBackOff, OOMKilled, ImagePullBackOff, Evicted e NodeNotReady.
4. IF a conexão com a API do Kubernetes falhar, THEN THE Agent SHALL registrar o erro e prosseguir a análise com os dados disponíveis de outras fontes.
5. THE Agent SHALL coletar o estado atual dos pods (Running, Pending, Failed, Unknown) do serviço afetado no momento da análise.

---

### Requisito 5: Correlação de Dados

**User Story:** Como SRE, quero que o agente correlacione automaticamente logs, métricas e eventos coletados, para que padrões causais sejam identificados sem intervenção manual.

#### Critérios de Aceitação

1. WHEN todos os dados de contexto forem coletados, THE Correlation_Engine SHALL correlacionar logs, métricas e eventos pelo timestamp e pelo identificador do serviço afetado.
2. THE Correlation_Engine SHALL identificar sequências temporais de eventos que precederam o alerta em até 10 minutos.
3. WHEN uma anomalia de métrica e um evento crítico do Kubernetes ocorrerem dentro de uma janela de 5 minutos, THE Correlation_Engine SHALL registrar a correlação como de alta relevância.
4. THE Correlation_Engine SHALL associar entradas de log com nível ERROR ou FATAL a eventos do cluster que ocorreram no mesmo intervalo de tempo.
5. IF nenhuma correlação for identificada entre as fontes de dados, THEN THE Correlation_Engine SHALL registrar a ausência de correlação e prosseguir para a geração de sugestões com base nos dados individuais.

---

### Requisito 6: Sugestão de Causa Raiz

**User Story:** Como SRE, quero que o agente sugira a causa raiz do problema com base nos dados correlacionados, para que eu possa agir rapidamente na resolução do incidente.

> **Princípio fundamental:** O Agent **nunca executa** ações de remediação. Toda Suggestion é uma recomendação para revisão e aprovação humana.

#### Critérios de Aceitação

1. WHEN a correlação de dados for concluída, THE Agent SHALL gerar ao menos uma Suggestion de causa raiz para cada alerta analisado.
2. THE Agent SHALL atribuir um Confidence_Score entre 0 e 100 a cada Suggestion, baseado na quantidade e qualidade das evidências correlacionadas.
3. THE Agent SHALL apresentar as Suggestions ordenadas do maior para o menor Confidence_Score.
4. WHEN o Confidence_Score da Suggestion principal for inferior a 40, THE Agent SHALL indicar explicitamente que a análise é inconclusiva e recomendar investigação manual.
5. THE Agent SHALL incluir em cada Suggestion: descrição da causa hipotética, evidências de suporte (logs, métricas e eventos relevantes) e ações recomendadas de remediação.
6. IF o padrão do alerta corresponder a um incidente previamente resolvido no histórico, THEN THE Agent SHALL incluir na Suggestion a referência ao incidente anterior e a solução aplicada.
7. THE Agent SHALL apresentar as ações recomendadas exclusivamente como sugestões textuais, sem executar nenhuma delas automaticamente.

---

### Requisito 7: Entrega dos Resultados e Notificações

**User Story:** Como SRE, quero receber o resultado da análise em um formato estruturado e acessível, e que o time de operações seja notificado automaticamente, para que todos possam tomar decisões rapidamente.

#### Critérios de Aceitação

1. WHEN a análise de um alerta for concluída, THE Agent SHALL entregar o resultado em formato JSON estruturado contendo: identificador do alerta, timestamp da análise, lista de Suggestions com Confidence_Score, evidências e ações recomendadas.
2. THE Agent SHALL concluir a análise e entregar o resultado em até 2 minutos após a ingestão do alerta.
3. WHEN a análise de um alerta for concluída, THE Agent SHALL criar um Jira_Ticket no projeto configurado contendo: identificador do alerta, severidade, resumo da análise, Suggestion principal com Confidence_Score e lista de ações recomendadas.
4. IF a criação do Jira_Ticket falhar, THEN THE Agent SHALL registrar o erro e prosseguir sem interromper o fluxo de análise de outros alertas.
5. WHERE a integração com o Microsoft Teams estiver habilitada, THE Agent SHALL enviar uma mensagem ao Teams_Channel configurado com o resumo da análise, o Confidence_Score da Suggestion principal e o link para o Jira_Ticket criado.
6. IF o envio da mensagem ao Teams_Channel falhar, THEN THE Agent SHALL registrar o erro sem interromper o fluxo de análise de outros alertas.

---

### Requisito 8: Histórico de Análises no S3

**User Story:** Como SRE, quero que o histórico de todas as análises seja armazenado de forma persistente e acessível no Amazon S3, para que eu possa consultar análises anteriores e identificar padrões recorrentes.

#### Critérios de Aceitação

1. WHEN a análise de um alerta for concluída, THE Agent SHALL armazenar o Analysis_Record completo em um bucket Amazon S3 configurado, no caminho `analyses/{ano}/{mês}/{dia}/{alert_id}.json`.
2. THE Agent SHALL armazenar no Analysis_Record: identificador do alerta, timestamp de ingestão, timestamp de conclusão, dados de contexto coletados, resultado da correlação, lista de Suggestions e identificador do Jira_Ticket criado.
3. THE Agent SHALL recuperar Analysis_Records do S3 por identificador de alerta para suportar a referência a incidentes anteriores (Requisito 6, critério 6).
4. IF a gravação no S3 falhar, THEN THE Agent SHALL registrar o erro e realizar até 3 tentativas com intervalo de 5 segundos antes de marcar a operação como falha.
5. THE Agent SHALL aplicar políticas de ciclo de vida no S3 para mover Analysis_Records com mais de 90 dias para a classe de armazenamento S3 Glacier, reduzindo custos de armazenamento.

---

### Requisito 9: Segurança e Controle de Acesso

**User Story:** Como engenheiro de plataforma, quero que o agente acesse os recursos AWS e Kubernetes com o mínimo de privilégios necessários, para que o risco de exposição de dados seja minimizado.

#### Critérios de Aceitação

1. THE Agent SHALL autenticar-se na AWS utilizando IAM Roles com permissões restritas às ações de leitura necessárias (CloudWatch Logs, CloudWatch Metrics, EKS) e às ações de escrita no bucket S3 de histórico.
2. THE Agent SHALL autenticar-se na API do Kubernetes utilizando um ServiceAccount com permissões de leitura (get, list, watch) restritas aos namespaces configurados.
3. THE Agent SHALL armazenar credenciais de integração (Jira API Token, Microsoft Teams Webhook) exclusivamente em AWS Secrets Manager ou variáveis de ambiente protegidas.
4. IF uma credencial de acesso expirar ou for revogada, THEN THE Agent SHALL registrar o erro de autenticação e notificar o operador sem expor detalhes da credencial nos logs.
5. THE Agent SHALL registrar em log de auditoria todas as consultas realizadas a fontes de dados externas, incluindo timestamp, fonte consultada e identificador do alerta associado.

---

### Requisito 10: Observabilidade do Próprio Agente

**User Story:** Como SRE, quero monitorar a saúde e o desempenho do próprio agente, para que falhas no agente não passem despercebidas.

#### Critérios de Aceitação

1. THE Agent SHALL expor métricas operacionais próprias, incluindo: número de alertas processados, tempo médio de análise, taxa de erros e número de análises inconclusivas.
2. WHEN o tempo de análise de um alerta exceder 2 minutos, THE Agent SHALL registrar um evento de timeout e marcar a análise como incompleta.
3. WHEN a taxa de erros nas consultas a fontes externas exceder 20% em uma janela de 5 minutos, THE Agent SHALL emitir um alerta operacional para o Teams_Channel de monitoramento configurado.
4. THE Agent SHALL disponibilizar um endpoint de health check que retorne o status operacional atual em formato JSON.

---

## Propriedades de Corretude

> ### O que é Property-Based Testing?
>
> Ao invés de escrever testes com exemplos fixos ("dado este alerta específico, espero este resultado"), o **property-based testing** define **propriedades** — regras que devem ser verdadeiras para **qualquer** entrada válida. Uma biblioteca de testes gera automaticamente centenas de casos aleatórios e verifica se a propriedade se mantém em todos eles.
>
> **Exemplo simples:** Em vez de testar "o Confidence_Score do alerta X é 75", você define a propriedade: "para qualquer alerta válido, o Confidence_Score deve estar sempre entre 0 e 100". O framework testa isso com milhares de alertas gerados automaticamente, encontrando casos extremos que você nunca pensaria em testar manualmente.
>
> As propriedades abaixo descrevem invariantes do sistema que devem ser verificadas dessa forma.

### Propriedade 1: Invariante do Confidence_Score

Para qualquer alerta válido processado pelo Agent, o Confidence_Score de cada Suggestion gerada deve estar sempre no intervalo fechado [0, 100].

- **Tipo:** Invariante
- **Por que é importante:** Garante que a pontuação de confiança nunca produza valores absurdos (negativos ou acima de 100), independentemente da combinação de dados de entrada.

### Propriedade 2: Ordenação das Suggestions

Para qualquer análise concluída com múltiplas Suggestions, a lista retornada deve estar sempre ordenada do maior para o menor Confidence_Score.

- **Tipo:** Invariante
- **Por que é importante:** A ordenação é uma propriedade estrutural que deve se manter para qualquer conjunto de Suggestions, não apenas para exemplos específicos.

### Propriedade 3: Round-Trip de Serialização do Analysis_Record

Para qualquer Analysis_Record gerado pelo Agent, serializar o registro em JSON e desserializá-lo de volta deve produzir um objeto equivalente ao original.

- **Tipo:** Round-trip (serialização/desserialização)
- **Por que é importante:** Parsers e serializadores são componentes propensos a bugs sutis. Essa propriedade garante que nenhum dado seja perdido ou corrompido ao gravar e ler do S3. É especialmente crítica para campos como timestamps, Confidence_Score e listas de evidências.

### Propriedade 4: Idempotência da Correlação

Para qualquer conjunto de dados de contexto (logs, métricas, eventos), executar o Correlation_Engine duas vezes sobre os mesmos dados deve produzir o mesmo resultado que executá-lo uma vez.

- **Tipo:** Idempotência
- **Por que é importante:** Garante que reprocessamentos (por falha ou retry) não alterem o resultado da análise, tornando o sistema previsível e seguro para reexecução.

### Propriedade 5: Completude das Suggestions

Para qualquer alerta válido processado, o Agent deve sempre gerar ao menos uma Suggestion — mesmo quando nenhuma correlação for encontrada entre as fontes de dados.

- **Tipo:** Invariante de completude
- **Por que é importante:** Garante que o Agent nunca retorne uma análise vazia, assegurando que o SRE sempre receba alguma orientação, mesmo que inconclusiva.

### Propriedade 6: Condições de Erro — Alertas Inválidos

Para qualquer entrada que não contenha os campos obrigatórios (identificador, severidade, timestamp), o Agent deve sempre retornar um erro descritivo e nunca gerar uma Suggestion.

- **Tipo:** Condição de erro
- **Por que é importante:** Garante que dados malformados nunca produzam análises silenciosamente incorretas. Testar com entradas inválidas geradas aleatoriamente (campos ausentes, tipos errados, valores nulos) é muito mais eficaz do que testar apenas exemplos manuais.
