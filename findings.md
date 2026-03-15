# Findings & Decisions

## Runtime State Atual
- Perguntas simples com candidato deterministico forte agora pulam o `generate_sql`.
- Caso medido: `qual o estado com maior producao de uva`.
- Benchmark aceito:
  - `avg backend ms`: `1345.41`
  - `avg llm ms`: `0.00`
  - `R2.generate_candidates`: `108.36`
- Warm path observado:
  - `backend_ms`: `860.18-908.43`

## Proximo Gargalo
- O gargalo remanescente esta concentrado nas perguntas que ainda precisam de LLM.
- Nessas perguntas, o custo dominante continua sendo `generate_sql::google/gemini-3-flash-preview`.
- O proximo ataque correto nao e mais cache de infraestrutura; e roteamento de modelo com fallback estrito.

## Requisitos do Proximo Ciclo
- Nunca aceitar SQL do modelo rapido sem validacao forte.
- Manter fallback obrigatorio para o modelo principal quando:
  - `QueryContract` divergir
  - a relevancia cair
  - o `SemanticCritic` apontar issues
  - o guard bloquear
  - a SQL parecer estruturalmente instavel para a pergunta

## Estratégia Recomendada
1. Construir uma suite de aceitacao com perguntas validadas do projeto.
2. Separar perguntas por classe:
   - `deterministic_skip`
   - `llm_required_simple`
   - `llm_required_complex`
3. Testar um modelo rapido apenas em `llm_required_simple`.
4. Validar automaticamente antes de aceitar:
   - `normalize_sql_candidate`
   - `sql_matches_query_contract`
   - `is_sql_relevant_to_question`
   - `SemanticCritic.assess`
   - `guard.validate`
5. Se qualquer gate falhar, chamar o modelo principal e descartar a tentativa rapida.

## Decisões
| Decision | Rationale |
|----------|-----------|
| Nao usar corte agressivo de prompt como estrategia principal | Reduziu tokens, mas nao trouxe ganho consistente e aumentou risco semantico |
| Continuar com `abstention-first` | A latencia so vale quando preserva qualidade |
| Tratar benchmark direto da uva como prova de que o skip deterministico resolveu o gargalo para perguntas simples | O `llm_ms` foi a zero no caso real |

## Recursos
- Relatorio tecnico principal: `/Users/davi/development/geo-ia-db-master/api-geo-nlp/docs/relatorio-otimizacao-memory-adaptation-2026-03-15.md`
- Runtime orchestrator: `/Users/davi/development/geo-ia-db-master/api-geo-nlp/app/modules/runtime/orchestrator.py`
- Vanna agent: `/Users/davi/development/geo-ia-db-master/api-geo-nlp/app/integrations/vanna/agent.py`
