# Benchmark OpenRouter para Geração de SQL

Data: 2026-03-12

## Objetivo

Comparar modelos OpenRouter candidatos para geração de SQL no pipeline Vanna do projeto `datahub2`, priorizando:

1. latência de resposta do LLM;
2. aderência ao schema real (`public.producao` + dimensões territoriais);
3. estabilidade sob `provider.sort=latency`.

## Configuração do benchmark

- Projeto: `datahub2`
- Perguntas:
  - `qual o bioma que mais produz milho`
  - `qual o estado que mais produz milho`
  - `qual o municipio que mais produz soja`
- Regras OpenRouter no benchmark:
  - `provider.sort=latency`
  - `allow_fallbacks=false`
  - `require_parameters=true`
  - `reasoning.effort=minimal` para modelos `google/*` e `anthropic/*`
- Harness:
  - [benchmark_openrouter_sql_models.py](/Users/nti-dev/development/geo-ia-db-master/api-geo-nlp/scripts/benchmark_openrouter_sql_models.py)
  - resultado bruto: [openrouter_sql_model_benchmark.json](/Users/nti-dev/development/geo-ia-db-master/api-geo-nlp/benchmarks/openrouter_sql_model_benchmark.json)

## Resultado resumido

| Modelo | Amostras | Avg LLM ms | P95 ms | Taxa de acerto funcional | Observação |
| --- | ---: | ---: | ---: | ---: | --- |
| `qwen/qwen3-coder-next` | 15 | 3269.93 | 5747.34 | 73.33% | Melhor equilíbrio entre latência e qualidade |
| `google/gemini-3-flash-preview` | 15 | 5097.13 | 14590.86 | 40.00% | SQL boa quando acerta, mas com cauda pesada |
| `google/gemini-3.1-flash-lite-preview` | 15 | 3785.02 | 5243.78 | 33.33% | Mais rápido que Gemini 3 Flash, mas hallucina colunas |
| `anthropic/claude-haiku-4.5` | 15 | 1894.63 | 6777.33 | 13.33%* | Boa sintaxe, mas benchmark foi afetado por saturação de conexões |
| `z-ai/glm-4.7-flash` | 15 | 26591.39 | 69140.10 | 46.67% | Inaceitável em latência |
| `qwen/qwen3-coder-flash` | 15 | 1878.76 | 3046.16 | 0.00% | OpenRouter retornou `404 No endpoints found...` em rota estrita |
| `qwen/qwen3.5-flash-02-23` | 15 | 1430.97 | 3322.50 | 0.00% | Mesmo problema de disponibilidade/aderência |

\* O número do `claude-haiku-4.5` ficou subestimado por um problema do harness: o primeiro benchmark abria uma conexão PostgreSQL nova por execução, saturando sockets locais no meio da rodada. Os SQLs válidos do Claude foram semanticamente corretos para `estado_milho`, e a amostra manual sugere melhor qualidade do que o percentual final indica.

## Leituras importantes

### 1. Vencedor claro para SQL

`qwen/qwen3-coder-next` foi o melhor modelo da bateria.

- venceu as três perguntas no menor tempo entre as execuções corretas;
- ficou tipicamente na faixa de `3.2s` a `5.7s`;
- gerou SQL aderente ao schema com menos alucinação de coluna.

### 2. Melhor fallback de geração SQL

`google/gemini-3-flash-preview` continua sendo um bom segundo modelo.

- quando acerta, gera SQL limpa e aderente;
- porém apresentou cauda ruim, incluindo outliers de `14s+` e um caso de `35s+`;
- em uma amostra gerou SQL com `geom`, o que elevou muito o tempo da própria consulta.

### 3. Melhor modelo leve para prompts não-SQL

`anthropic/claude-haiku-4.5` é a melhor opção para `LLM_MODEL_LIGHT`.

- mais adequado para classificação e prompts curtos do que um coder puro;
- a amostra manual mostrou SQL correta para `estado_milho`;
- o benchmark dele foi contaminado pelo problema de conexões, então o percentual final não representa bem o potencial real.

### 4. Modelos a evitar neste pipeline

- `z-ai/glm-4.7-flash`: latência muito alta (`10s` até `72s`);
- `qwen/qwen3-coder-flash`: indisponível com a política estrita de velocidade usada no teste;
- `qwen/qwen3.5-flash-02-23`: mesmo problema de disponibilidade e baixa aderência;
- `google/gemini-3.1-flash-lite-preview`: bom tempo médio, mas errou colunas do schema (`cd_geocmu`, `nm_municipio`).

## Recomendação para o `.env`

Substituir os modelos atuais por:

```env
API_GEO_NLP_LLM_MODEL_BEST=qwen/qwen3-coder-next
API_GEO_NLP_LLM_MODEL_HEAVY=google/gemini-3-flash-preview
API_GEO_NLP_LLM_MODEL_LIGHT=anthropic/claude-haiku-4.5
```

## Recomendação para a chamada OpenRouter

Para produção com foco em rapidez, sem sacrificar disponibilidade:

```env
API_GEO_NLP_OPENROUTER_PROVIDER_SORT=latency
API_GEO_NLP_OPENROUTER_ALLOW_FALLBACKS=true
API_GEO_NLP_OPENROUTER_REQUIRE_PARAMETERS=false
API_GEO_NLP_OPENROUTER_REASONING_EFFORT=minimal
API_GEO_NLP_OPENROUTER_TIMEOUT_SECONDS=45
```

Para benchmark estrito de latência:

```env
API_GEO_NLP_OPENROUTER_PROVIDER_SORT=latency
API_GEO_NLP_OPENROUTER_ALLOW_FALLBACKS=false
API_GEO_NLP_OPENROUTER_REQUIRE_PARAMETERS=true
API_GEO_NLP_OPENROUTER_REASONING_EFFORT=minimal
API_GEO_NLP_OPENROUTER_TIMEOUT_SECONDS=45
```

## Mudanças aplicadas no código

- [config.py](/Users/nti-dev/development/geo-ia-db-master/api-geo-nlp/app/core/config.py)
  - adicionados knobs de roteamento do OpenRouter.
- [agent.py](/Users/nti-dev/development/geo-ia-db-master/api-geo-nlp/app/integrations/vanna/agent.py)
  - client do Vanna agora injeta `provider.sort`, fallback policy, reasoning effort e timeout no OpenRouter.
- [.env.example](/Users/nti-dev/development/geo-ia-db-master/api-geo-nlp/.env.example)
  - documentadas as novas variáveis.

## Validação

- `python -m py_compile app/core/config.py app/integrations/vanna/agent.py scripts/benchmark_openrouter_sql_models.py`
- `API_GEO_NLP_ARTIFACTS_BACKEND=filesystem API_GEO_NLP_ARTIFACTS_BASE_PATH=/tmp/api-geo-nlp-test-artifacts PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 ./.venv/bin/pytest tests/unit/test_vanna_agent_relevance.py -q`
  - `31 passed`
