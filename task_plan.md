# Plano de Trabalho - ai-data-pilot-manager (Modulo 3)

## Objetivo
Concluir o Modulo 3 de forma executavel, reaproveitando a implementacao parcial existente em `ai-data-pilot-manager` e `api-geo-nlp`, readequando o plano anterior ao delta real do codigo, ao banco atual e ao relatorio `relatorio_bird_spider_diagnostico_otimizacao.md`.

## Fases
| Fase | Status | Entrega |
|---|---|---|
| 1. Levantamento de contexto (docs + codigo atual) | completed | Requisitos consolidados, estado atual da API/UI identificado |
| 2. Estruturacao de arquivos de planejamento | completed | `task_plan.md`, `findings.md`, `progress.md` |
| 3. Criacao do subprojeto de planejamento | completed | `ai-data-pilot-manager/` com README e plano detalhado |
| 4. Validacao tecnica com MCP (DB/API/UI) | completed | Tabelas existentes validadas, extensoes e gaps reais mapeados |
| 5. Reavaliacao do plano com base no relatorio BIRD/SPIDER | completed | Plano executivo otimizado e backlog repriorizado para fechar gaps reais de control plane e NL2SQL |
| 6. Implementacao API/control plane | completed | Schema write, sincronizacao canonica, feedback/listagens, endurecimento de configuracao |
| 7. Melhorias NL2SQL Fase 1 | completed | Multi-candidate search, rerank, repair limitado, decision trace/diagnosticos |
| 8. Evolucao UI (fluxo completo + UX) | completed | Setup, metadata, questions, lab e navegacao em nivel produto |
| 9. Dominio Vanna Access + auditoria | completed | Usuarios/grupos/permissoes por projeto, enforcement no runtime e trilha consultavel na UI concluídos |
| 10. Validacao final | completed_with_notes | Testes locais, smoke browser, validacao em DB real e commits concluidos; permanece apenas warning isolado do chunk de charts e delecoes historicas fora do escopo no repo da API |

## Entregaveis desta sessao
- Plano do Modulo 3 readequado ao estado real do repositorio.
- Ajustes estruturais na `api-geo-nlp` para control plane e para qualidade NL2SQL.
- Evolucao funcional da UI `ai-data-pilot-manager` com fluxo implementacao -> teste -> validacao.
- Verificacao final do banco `ia-data-pilot` e do estado de `pgvector`.
- Fechamento do dominio Vanna Access e da auditoria operacional ponta a ponta.
- Commits separados por repositorio filho para backend e frontend.

## Erros Encontrados
| Erro | Tentativa | Resolucao |
|---|---|---|
| `extension "vector" is not available` ao executar `CREATE EXTENSION vector` em 2026-03-05 | 1 | Bloqueio registrado na sessao anterior |
| Em 2026-03-06, `SELECT extname FROM pg_extension` ainda nao retornava `vector` no banco MCP `ia-data-pilot` antes do startup da API | 2 | Resolvido ao subir a API local com auto-init de schema; apos o startup, `vector` apareceu em `pg_extension`, a coluna `embedding` foi criada e o indice `idx_project_rag_chunks_embedding` ficou ativo |
| `pytest` falhando por plugin externo `logfire` / `opentelemetry.sdk._logs.ReadableLogRecord` | 1 | Executar testes Python com `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` ou ambiente isolado do projeto |
| Warning de chunk >500 kB no frontend apos code-splitting | 1 | Bundle principal foi quebrado em lazy routes e manual chunks; o warning restante ficou restrito ao chunk opcional de `charts` (ECharts), carregado sob demanda no laboratorio |
| Escopo remanescente identificado apos reavaliacao do plano | 1 | Confirmado que ainda faltam dominio Vanna Access com enforcement real e tela/endpoint de auditoria; trabalho retomado para fechar o delta antes da validacao final |
| Repositorio `api-geo-nlp` com delecoes historicas em `docs/workflow_runtime_20x_*` | 1 | Mantidas fora dos commits desta entrega por nao fazerem parte do escopo implementado nem da validacao funcional |
