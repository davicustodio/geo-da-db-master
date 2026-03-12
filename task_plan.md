# Task Plan

## Goal
Implementar o redesenho em duas etapas do pipeline:
1. preparar e curar a base semântica do projeto (DDL, Dictionary, Questions);
2. gerar/publicar embeddings no Vanna para liberar o RAG/Lab.

O entregável inclui backend, frontend e testes atualizados.

## Goal (2026-03-11 - Apresentação)
Analisar a arquitetura atual do projeto, com foco na API `api-geo-nlp` e no pipeline baseado em Vanna AI, para montar um plano de apresentação de 15 minutos em linguagem acessível para público leigo.

## Phases
- [x] Mapear o fluxo atual de setup, treinamento, curadoria e lab.
- [x] Definir o modelo-alvo em `api-geo-nlp/docs/plano-ui-pipeline-duas-etapas.md`.
- [x] Separar backend em estado de base semântica e estado de embeddings.
- [x] Ajustar a UI para refletir as duas etapas e a dependência entre elas.
- [x] Atualizar testes e executar verificações principais.
- [x] Mapear o fluxo real do runtime (`/ask`) e do pipeline de preparo semântico/embeddings.
- [x] Consolidar a mensagem principal para público leigo.
- [x] Definir sequência de slides, tempo por slide e mensagem-chave para apresentação de 15 minutos.

## Decisions
- Usar o skill `planning-with-files` porque a tarefa é de organização e definição estratégica.
- Implementar a separação explícita entre `base semântica` e `embeddings do Vanna`.
- Fazer a geração inicial parar em `discovery + questions`; embeddings passam a ser ação separada.
- Inutilizar embeddings quando houver edição posterior em metadata, policies, DDL, Dictionary ou Questions.
- Usar a versão mais recente como `working version` para curadoria, mantendo a versão ativa apenas para runtime/Lab.

## Errors Encountered
- `eslint` não está disponível no ambiente atual (`npm run lint` falha com `eslint: command not found`).

## Follow-up: diagnóstico da falha em embeddings
- [x] Reproduzir a falha do botão `Gerar embeddings` com usuário real na UI local.
- [x] Correlacionar o novo job com `training/state` e `training/jobs/{job_id}/events`.
- [x] Executar o mesmo validador de runtime do backend sobre a versão semântica revisada.
- [x] Isolar a SQL responsável pela reprovação do quality gate.
- [x] Mapear lacunas de persistência/observabilidade para a falha.
- [x] Corrigir operacionalmente a `question 388` e reconcluir a revisão semântica.
- [x] Persistir detalhes do quality gate mesmo em falha.
- [x] Bloquear build/publish com diagnóstico explícito para questions inválidas.
- [x] Exibir diagnóstico na curadoria e no monitor/log do job.
- [x] Reexecutar o fluxo real até `embeddings=published`.

## Decisions
- Tratar a falha atual como problema de conteúdo gerado em `questions` mais falta de observabilidade do pipeline, não como defeito do clique da UI.
- Considerar duas frentes de solução: mitigação imediata nos dados (`question` inválida) e correção estrutural no backend para persistir detalhes de falha e, idealmente, autocurar SQL inválida antes do publish.
- Implementar o bloqueio explícito para `sql_is_valid = false` antes do publish, sem esperar o erro agregado de runtime.
- Persistir `quality_report`/`quality_failure` no job e no manifesto para a UI poder mostrar causa acionável.

## Follow-up: diagnóstico do desvio no Lab (`LIMIT 5` + filtro indevido por estado)
- [x] Reproduzir a pergunta do Lab diretamente no runtime do projeto `datahub2`.
- [x] Identificar quais chunks/candidatos estão sendo recuperados para a pergunta `quais as 10 cidades que mais produzem soja`.
- [x] Medir a latência por etapa do runtime para separar custo de LLM e custo de SQL.
- [x] Revisar o gate de qualidade do pipeline para verificar se ele cobre aderência semântica pergunta→SQL.
- [x] Consolidar causa raiz e plano de execução.

## Follow-up: alinhamento com a documentação do Vanna AI 2.0
- [x] Levantar as recomendações oficiais do Vanna 2.0 para Tool Memory, recuperação de contexto e montagem de prompt.
- [x] Comparar a arquitetura atual do projeto com o fluxo `Agent + ToolRegistry + Tool Memory` documentado pelo Vanna 2.0.
- [x] Verificar se os exemplos recuperados entram de fato no prompt do `vn.generate_sql`.
- [x] Consolidar as divergências entre o comportamento recomendado pelo Vanna e a implementação atual.
- [x] Reescrever o plano de execução priorizando aderência ao Vanna 2.0 nessa etapa.
- [x] Filtrar exemplos incompatíveis antes da montagem do prompt.
- [x] Substituir o prompt legado por um prompt contextual com instrução explícita de adaptação à pergunta atual.
- [x] Limpar feedbacks `runtime_auto` persistidos do `datahub2`.
- [x] Revalidar o fluxo via API autenticada após reiniciar o backend local.

## Follow-up: performance do Lab para perguntas com resultado geografico
- [x] Mapear onde o `Lab` renderiza tabela, grafico e payload geoespacial.
- [x] Mapear no runtime onde `geom` e `GeoJSON` entram no fluxo síncrono e em background.
- [x] Confirmar se a UI depende do payload `geo` para algo além do botão/modal atual.
- [x] Confirmar se o contrato atual já expõe `modalities` para indicar tipo de resultado.
- [x] Consolidar o novo requisito de contrato: a API precisa aceitar modo com e sem retorno de geometria.
- [x] Definir o contrato novo do runtime (`AskRequest`) para controlar retorno de colunas geoespaciais.
- [x] Definir o corte de escopo do `Lab`: usar sempre o modo sem retorno de geometria.
- [x] Definir sanitização obrigatória de colunas geoespaciais na tabela retornada quando o modo sem geometria estiver ativo.
- [x] Definir ajustes de UX para indicar modalidades previstas/retornadas sem modal geográfico.
- [x] Definir propagação do novo parâmetro para `/projects/{project_id}/ask` e transportes Vanna nativos (SSE, WebSocket, polling).
- [x] Definir pacote mínimo de testes e métricas para validar ganho de latência.

## Follow-up: observabilidade detalhada do `Perguntar`
- [x] Mapear o fluxo ponta a ponta do clique em `Perguntar` ate `/projects/{project_id}/ask`.
- [x] Confirmar os pontos atuais de custo provavel no runtime (`_classify_intent`, `generate_sql_candidates`, `rank_sql_candidates`, `_suggest_chart_with_vanna`, SQL, geo e serializacao).
- [x] Definir o plano tecnico de instrumentacao ponta a ponta para separar UI, rede, backend, LLM, banco e payload.
- [x] Implementar correlacao por `request_id` entre UI, API e auditoria.
- [x] Adicionar spans/metricas por etapa no runtime do backend.
- [x] Instrumentar chamadas LLM e SQL em wrappers centrais.
- [x] Expor metricas no contrato de debug e na auditoria.
- [x] Implementar harness/script para executar bateria controlada e consolidar `avg/p50/p95/% total`.
- [x] Executar bateria controlada de medicao e consolidar `p50/p95/% total`.
- [x] Rodar testes A/B para identificar chamadas LLM desnecessarias.

## Follow-up: diagnostico da pergunta `qual o bioma que mais produz milho`
- [x] Reproduzir a pergunta no runtime com `timing` detalhado.
- [x] Separar cold path, warm path e efeito do cache exato por pergunta.
- [x] Confirmar quais modelos LLM estao sendo usados para gerar SQL.
- [x] Medir bootstrap/hidratacao do Vanna e busca vetorial antes da geracao.
- [x] Testar modelos alternativos configurados para `vn.generate_sql`.
- [x] Implementar instrumentacao adicional por etapa no runtime.
- [x] Remover fan-out redundante de modelos quando o primeiro candidato ja for aderente.
- [x] Implementar estrategia otimizada sem cache semantico para esse tipo de pergunta.
- [x] Consolidar relatorio persistido em `docs/diagnostico-latencia-lab-milho-2026-03-12.md`.

## Follow-up: diagnostico de pergunta simples desviada como `ui_command`
- [x] Reproduzir a falha no Lab com o usuario `davi.custodio@embrapa.br`.
- [x] Correlacionar a resposta do Lab com a classificacao de intencao e a rota backend usada.
- [x] Validar se a pergunta deveria ser atendida pelo corpus/dados do projeto `datahub2`.
- [x] Identificar a causa raiz no classificador/gates anteriores a geracao de SQL.
- [x] Consolidar plano de correcao obrigatorio para evitar ausencia de resposta em perguntas semelhantes.
- [x] Implementar endurecimento de `_classify_intent()` para perguntas analiticas imperativas.
- [x] Cobrir o classificador com testes unitarios.
- [x] Normalizar filtros textuais geograficos para comparacao case-insensitive no reparo de SQL.
- [x] Validar a frase original via HTTP real e confirmar que o comando de UI explicito continua preservado.
- [x] Reduzir latencia de lookup simples sem hardcode de schema/projeto.
- [x] Introduzir fast path generico baseado apenas em contexto semantico do projeto ativo.
- [x] Validar ganho de latencia via HTTP real na pergunta original.
- [x] Priorizar Tool Memory vetorial antes da hidratacao completa do Vanna.
- [x] Adicionar fast path de adaptacao para perguntas semelhantes a partir da memoria vetorial publicada.
- [x] Adicionar cache por versao semantica ativa + pergunta normalizada.
- [x] Medir cold path vs warm path com repeticao real da mesma pergunta.

## Decisions
- Tratar o problema do Lab como falha combinada de corpus, recuperação, ranqueamento e aprendizado automático, não como erro isolado da tela.
- Priorizar remoção do aprendizado automático não validado (`runtime_auto`) antes de qualquer novo retraining, porque ele contamina a memória vetorial e mascara a qualidade do corpus curado.
- Endurecer a validação de aderência pergunta→SQL no runtime e no quality gate com foco em cardinalidade solicitada (`top 10`), filtros explícitos vs. implícitos e entidades geográficas pedidas/não pedidas.
- Atacar performance reduzindo chamadas LLM evitáveis no runtime síncrono: classificação de intenção para perguntas óbvias, sugestão de gráfico e geração por múltiplos modelos quando já houver candidatos confiáveis.
- Considerar o estado atual como integração híbrida/legada com Vanna 2.0, não como implementação fiel do pipeline oficial do Agent Framework.
- Tratar como desvio crítico o fato de a memória similar ser usada como SQL candidata direta antes da geração do prompt, porque isso contorna a adaptação contextual que a documentação do Vanna 2.0 descreve.
- Corrigir a compatibilidade de `get_similar_question_sql()` com o contrato esperado pelo prompt do Vanna legado (`dict` com `question` e `sql`), ou abandonar essa via e migrar explicitamente para o fluxo oficial do Agent/Tool Memory do Vanna 2.0.
- Para perguntas geo-analiticas simples e recorrentes, priorizar compilacao deterministica por contrato antes de Tool Memory, PgVector e `vanna.generate_sql`.
