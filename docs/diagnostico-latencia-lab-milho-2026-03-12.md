# Diagnóstico de Latência do Lab: `qual o bioma que mais produz milho`

## Objetivo

Medir o pipeline completo do `Perguntar` para a pergunta `qual o bioma que mais produz milho`, identificar o gargalo real, confirmar qual LLM gera a SQL e registrar a estratégia de otimização sem depender de cache semântico entre perguntas parecidas.

## Pipeline observado

1. `LabPage.onAsk()` marca o início do clique, monta `debug_options` e chama `ask()`.
2. `src/api/client.ts::ask()` gera `request_id`, envia `POST /projects/{project_id}/ask` e mede `http_roundtrip_ms`.
3. `RuntimeOrchestrator.ask()` executa:
   - `R0.resolve_context`
   - `R1.classify_intent`
   - `T0.resolve_credentials`
   - `R2.generate_candidates`
   - `R2B.rank_select`
   - `R3.guard_validate`
   - `R4.cost_estimate`
   - `R5.execute_query`
   - `R6.plan_modalities`
   - `R10.save_memory`

## LLMs e configuração observados

- Provider: `openrouter`
- Modelo principal de geração SQL (`llm_model_best`): `google/gemini-3-flash-preview`
- Modelo fallback/heavy (`llm_model_heavy`): `qwen/qwen3.5-flash-02-23`
- Modelo light (`llm_model_light`): `z-ai/glm-4.7-flash`
- Embeddings vetoriais: `text-embedding-3-small`
- Backend de memória Vanna: `postgres`

## Medições relevantes

### 1. Runtime frio antes da otimização atual

Medição direta em processo novo, sem cache em memória do processo:

- `end_to_end_ms`: `14814.23`
- `total_backend_ms`: `14763.96`
- `llm_total_ms`: `12123.46`
- `db_total_ms`: `411.58`
- `R2.vector_memory_embedding`: `507.51`
- `R2.vector_memory_search`: `71.06`
- `R2.generate_candidates`: `14133.22`
- `generate_sql :: google/gemini-3-flash-preview`: `6392.61`
- `generate_sql :: qwen/qwen3.5-flash-02-23`: `5730.85`

Conclusão:

- O banco não era o gargalo.
- A pergunta disparava duas chamadas LLM sequenciais para gerar SQL.

### 2. Runtime frio após parar no primeiro LLM aderente

Medição direta em processo novo após remover a segunda chamada desnecessária:

- `end_to_end_ms`: `9550.97`
- `total_backend_ms`: `9497.36`
- `llm_total_ms`: `6816.84`
- `db_total_ms`: `321.53`
- `R2.vector_memory_embedding`: `435.12`
- `R2.vector_memory_search`: `71.13`
- `R2.vanna_ready`: `533.81`
- `generate_sql :: google/gemini-3-flash-preview`: `6816.84`

Conclusão:

- Cortar o segundo modelo reduziu o caminho frio em cerca de `5.26s`.
- Mesmo assim, a única chamada LLM ainda dominava o tempo total.

### 3. HTTP frio com servidor reiniciado antes do fast path determinístico

Primeiro request HTTP após restart do worker:

- `http_roundtrip_ms`: `20418.17`
- `total_backend_ms`: `20339.79`
- `llm_total_ms`: `17113.95`
- `db_total_ms`: `321.50`
- `R2.vector_memory_embedding`: `538.18`
- `R2.vanna_instance_init`: `175.63`
- `R2.vanna_dataset_hydrate`: `259.13`
- `R2.vanna_ready`: `533.81`
- `generate_sql :: google/gemini-3-flash-preview`: `17113.95`

Conclusão:

- A variância principal está no provedor/modelo LLM.
- O mesmo pipeline pode oscilar de ~`6.8s` para ~`17.1s` em uma única chamada `generate_sql`.

### 4. HTTP frio após fast path determinístico

Primeiro request HTTP após restart com `structured_rule_fast_path` habilitado:

- `http_roundtrip_ms`: `645.72`
- `total_backend_ms`: `562.12`
- `llm_total_ms`: `0`
- `db_total_ms`: `324.97`
- `R2.structured_rule_fast_path`: `0.01`
- `R2.generate_candidates`: `70.85`
- `R2B.rank_select`: `116.79`
- `R5.execute_query`: `153.44`

SQL vencedora:

```sql
SELECT COALESCE(m.mu_bio, m.mi_bio) AS bioma,
       SUM(p.qtde_produzida) AS qtde_total
FROM public.producao AS p
JOIN public.municipio AS m ON p.geocodigo = m.geocodigo
WHERE UPPER(p.nome_produto) = UPPER('milho')
GROUP BY COALESCE(m.mu_bio, m.mi_bio)
ORDER BY qtde_total DESC, bioma
LIMIT 1
```

Conclusão:

- O caso do milho deixou de depender de LLM.
- O tempo agora é dominado por `resolve_credentials`, `estimate`, `rank_select` e `execute`.

### 5. HTTP warm path

Mesma pergunta no mesmo processo, reaproveitando o cache exato por pergunta normalizada:

- `http_roundtrip_ms`: `596.43`
- `total_backend_ms`: `525.09`
- `llm_total_ms`: `0`
- `R2.candidate_cache_hit`: `0.01`

## Comparativo de modelos para `vn.generate_sql`

Microbenchmark com 3 execuções por modelo, mesma instância Vanna já hidratada:

### `google/gemini-3-flash-preview`

- `avg_ms`: `6673.45`
- `p50_ms`: `6819.01`
- `min_ms`: `6196.74`
- `max_ms`: `7004.61`

Observação:

- Em 1 de 3 execuções gerou um join menos confiável (`p.geocodigo = m.cd_geocmu`).

### `qwen/qwen3.5-flash-02-23`

- `avg_ms`: `6963.18`
- `p50_ms`: `6498.03`
- `min_ms`: `6113.26`
- `max_ms`: `8278.25`

Observação:

- Velocidade semelhante ao Gemini.
- Não houve ganho consistente de latência.

### `z-ai/glm-4.7-flash`

- amostra isolada: `17846.34ms`

Conclusão:

- Trocar o modelo, isoladamente, não resolve o gargalo.
- `glm-4.7-flash` foi claramente pior.
- `qwen` não mostrou vantagem suficiente para justificar troca imediata.

## Causa raiz consolidada

1. O SQL do banco é rápido; o gargalo estava antes da execução.
2. A geração de candidatos fazia trabalho redundante:
   - embeddings + busca vetorial;
   - bootstrap/hidratação do Vanna;
   - 2 chamadas LLM sequenciais.
3. Mesmo com apenas 1 chamada LLM, a latência do provedor variava muito.
4. O cache exato por pergunta escondia o problema em requests repetidos, mas não resolvia o cold path.

## Mudanças aplicadas

1. Instrumentação adicional no runtime:
   - `R2.vanna_instance_init`
   - `R2.vanna_dataset_hydrate`
   - `R2.vanna_ready`
   - `R2.stop_after_relevant_generate_sql`
2. `generate_sql_candidates()` agora para após o primeiro `vanna.generate_sql` aderente.
3. `generate_sql_candidates()` ganhou `structured_rule_fast_path` antes de Tool Memory, PgVector e Vanna.

## Estratégia recomendada

1. Manter apenas cache exato por `project_id + active_version + normalized_question`.
2. Evitar cache semântico para “perguntas parecidas” retornarem SQL pronta.
3. Para perguntas simples e frequentes, usar compilação determinística baseada em contrato:
   - dimensão geográfica;
   - métrica;
   - filtro explícito;
   - cardinalidade (`LIMIT 1`, `LIMIT 10`, etc.).
4. Usar memória vetorial e Tool Memory apenas como contexto para prompts mais complexos, nunca como SQL final direta.
5. Manter `llm_model_heavy` apenas como fallback quando o principal falhar ou gerar SQL irrelevante.

## Próximos passos

1. Expandir o fast path determinístico para outras classes simples:
   - `top N` por município/estado/região/bioma;
   - totais por dimensão;
   - distribuições simples.
2. Reduzir o custo de `rank_select` para perguntas com um único candidato determinístico.
3. Persistir benchmark recorrente com cenários de cold e warm path.
