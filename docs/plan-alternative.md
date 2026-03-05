Aqui está a versão completa do plano atualizado em formato Markdown, pronta para você copiar e colar onde precisar:

# Arquitetura Avançada para Sistemas Text-to-SQL Geoespaciais com Integração Agêntica Multicamadas

A convergência entre Modelos de Linguagem de Grande Escala (LLMs) e bancos de dados relacionais representa um dos vetores de maior transformação na engenharia de dados. Para domínios geoespaciais, a complexidade escala ao envolver o PostGIS operando sobre PostgreSQL.

Para atender a esses requisitos utilizando o Vanna AI 2.0 como motor principal, projeta-se uma infraestrutura distribuída em três módulos interdependentes. O Vanna 2.0 fornece a orquestração de inferência, o controle de memória de ferramentas (Tool Memory), e a customização de prompts agênticos suportados pelo Pydantic AI. A interface final reside em um ambiente isolado baseado em React, Leaflet e CopilotKit, comunicando-se com o backend exclusivamente via APIs.

---

## Módulo 1: Motor Principal Text-to-SQL (Backend API)

Este módulo é inteiramente desenvolvido em Python, exposto via FastAPI, e atua como o cérebro da operação. Em vez de criar múltiplos scripts desconexos, as fases do pipeline foram redesenhadas para se integrarem como **Lifecycle Hooks**, **Middlewares** e **Ferramentas Customizadas (Custom Tools)** dentro do ecossistema do Vanna AI 2.0 e Pydantic AI.

### Fase 1: Gerenciamento de Contexto, Filtros de Conversação e Histórico

O Vanna 2.0 possui suporte a *Conversation Storage*. Nesta fase inicial, o middleware do sistema intercepta a chamada de API contendo o `conversation_id`.

- **Ação:** O framework recupera as mensagens anteriores da sessão do usuário. Se o usuário diz "mostre os produtores de arroz" e depois "mostre os de feijão", o gerenciador de contexto acopla o histórico dinâmico ao LLM.
- **Conversation Filters:** Neste ponto, a arquitetura do Vanna 2.0 introduz a interface `ConversationFilter` para higienizar os dados. O pipeline aplica um `SensitiveDataFilter` para interceptar e mascarar automaticamente informações sensíveis (PII, como e-mails e CPFs) antes de enviá-las ao LLM, e um `ContextWindowFilter` que calcula e corta dinamicamente as mensagens mais antigas caso o histórico ameace ultrapassar a janela máxima de tokens do modelo.
- **Custom Prompts:** O agente recebe dinamicamente "System Prompts" customizados (definidos no Módulo 3) injetados através do `UserResolver`, garantindo que regras específicas de nível de usuário já balizem a interpretação do contexto.

### Fase 2: Configuração e Extração Agêntica de Metadados (Setup)

A extração de conhecimento do banco de dados (DDL, Dicionário, SQLs válidos) é automatizada utilizando Python.

- **Domínios e Metadados:** Scripts enviam queries PostgreSQL como `select key, array_agg(distinct value) from tabela, jsonb_each_text(to_jsonb(tabela)) group by key;` para extrair exaustivamente todos os domínios válidos de colunas categóricas.
- **Tratamento Espacial:** O sistema analisa a tabela `geometry_columns` para coletar o SRID e orienta o Dicionário de Dados sobre o uso de geometrias projetadas (metros, UTM) ou geográficas (graus) usando funções de ST do PostGIS.

### Fase 3: Treinamento do Banco de Embeddings (API Programática)

Com o dicionário, DDL e as perguntas mapeadas geradas na Fase 2, o agente utiliza a API programática nativa do Vanna para alimentar o vetor de contexto.

- **Ação:** Emprega-se funções como `vn.train(ddl="...")`, `vn.train(documentation="...")` e `vn.train(sql="...", question="...")` de maneira estruturada no Python. Isso constitui o treinamento inicial obrigatório antes de qualquer inferência.

### Fase 4: Inferência Agêntica e Enriquecimento de Contexto (LlmContextEnhancer)

Quando uma pergunta contextualizada validada chega, o agente do Pydantic AI entra em ação. O Vanna AI 2.0 permite registrar ferramentas específicas no agente (`agent.register_tool()`).

- **RAG e Geração via LlmContextEnhancer:** O Vanna AI 2.0 padroniza a fase de RAG utilizando a interface `LlmContextEnhancer`. Este componente intercepta o prompt do sistema imediatamente antes da primeira requisição ao LLM, executa a busca vetorial (Vector Search) baseada na pergunta atual e injeta automaticamente a DDL recuperada e exemplos de histórico de sucesso no prompt final.
- **A Ferramenta RunSqlTool:** O Vanna então decide invocar o `RunSqlTool` acoplado a um `PostgresSqlRunner` para gerar a sintaxe com base no contexto enriquecido.

### Fase 5 e 6: Validação de Segurança, Análise de Custos e Observabilidade

No Vanna 2.0, fases de auditoria de sintaxe e validação de custo de processamento não são executadas em instâncias separadas, mas anexadas como **Lifecycle Hooks / Middlewares** disparados instantes antes da execução física no banco de dados.

- **Segurança:** Um *hook* analisa a query SQL interceptada com a biblioteca `sqlglot` para garantir a ausência de comandos DML restritos (DELETE, DROP, etc).
- **Custo (EXPLAIN):** O *hook* executa `EXPLAIN` no Postgres e utiliza heurísticas para prever gargalos de agregações geográficas (ex: cruzamento de polígonos massivos sem índices). Se reprovado, o hook retorna erro ("Query excede limite de tempo"), evitando travar a infraestrutura.
- **Auditoria e Telemetria:** O sistema utiliza os hooks do Vanna para instanciar um `Custom Audit Logger`, conectando-se a plataformas de observabilidade (como o Pydantic Logfire). Isso permite o rastreio distribuído das consultas executadas e o monitoramento em tempo real do uso de tokens e custos por usuário.

### Fase 7: Tomada de Decisão de Visualização e Roteamento Geográfico Avançado

Após o `RunSqlTool` processar com sucesso, o pipeline retorna os dados brutos. É nesta etapa que o roteamento algorítmico determina a forma da resposta, com um fluxo específico e robusto para dados espaciais.

- **Sinalização Analítica/Gráfica:** Se os dados não forem geoespaciais, o agente aciona a ferramenta nativa `VisualizeDataTool`. O Vanna examina os metadados e define a criação de um `ChartComponent` (Plotly ou Vega-Lite), ou apenas texto se for um valor escalar.
- **Orquestração Espacial (GeoJSON vs. GeoServer):** O agente detecta as colunas WKB/EWKB do PostGIS e toma uma decisão de roteamento baseada no volume de dados e nos parâmetros estilísticos recebidos da UI (Módulo 2):
- **Rota A (Vetor Leve - GeoJSON):** Para consultas simples e de baixo volume, o agente realiza a transformação geométrica diretamente no Python. Utilizando bibliotecas como o GeoPandas com o parâmetro `to_wgs84=True`, os dados são convertidos para o padrão GeoJSON (RFC 7946 EPSG:4326) e respondidos à API para renderização vetorial no cliente Leaflet.
- **Rota B (Renderização Massiva e SLD - Sql-View-Geoserver):** Se a query gerar geometrias densas, ou se o usuário tiver submetido preferências de estilo na mesma frase (ex: "mostre todos os municípios do Pará em diferentes cores aleatórias"), o agente adota uma rota de delegação. Primeiro, um prompt específico orienta o LLM a traduzir o requisito visual em código OGC Styled Layer Descriptor (SLD XML) parametrizado de forma válida. Em seguida, o agente envia o SQL puro e o SLD aprovado para o módulo de integração externo (Sql-View-Geoserver). Este módulo interage com a REST API do GeoServer para gerar dinamicamente a SQLView. O agente recebe de volta a URL de acesso WMS gerada e a empacota na resposta ao frontend, isentando a UI de carregar milhares de polígonos na memória.

### Fase 8: Ciclo de Autoaprendizado (Tool Memory)

Cada resposta e query gerada com sucesso engatilha o mecanismo embutido do Vanna de aprendizado contínuo.

- **Ação:** A funcionalidade de `ToolMemory` é invocada. A combinação da pergunta original com o código SQL aprovado é injetada vetorialmente de volta ao banco de dados do Vanna (`AgentMemory`). Na próxima vez que uma intenção semelhante surgir, a arquitetura RAG prioriza a ramificação conhecida e testada (Similar Question Path), elevando exponencialmente a acurácia.


| Workflow do Vanna (Módulo 1) | Funcionalidade Equivalente                              | Tecnologia / Componente Vanna 2.0              | Necessita de LLM |
| ---------------------------- | ------------------------------------------------------- | ---------------------------------------------- | ---------------- |
| **Histórico e Filtros**      | Lembrar chat, mascarar PII e evitar excesso de tokens.  | `Conversation Storage`, `ConversationFilter`   | Sim              |
| **Setup de Dados**           | Extração de Dicionários/DDL do PostgreSQL e PostGIS.    | Conectores SQL, `jsonb_each_text`              | Não              |
| **Treinamento**              | Preenchimento vetorial programático do banco.           | Vanna API (`vn.train`)                         | Não              |
| **Inferência e Contexto**    | Injeção vetorial RAG no Prompt e geração SQL.           | `LlmContextEnhancer`, `RunSqlTool`             | Sim              |
| **Filtros e Auditoria**      | Bloquear queries pesadas e rastrear uso de ferramentas. | Lifecycle Hooks, Logfire, `AuditLogger`        | Sim              |
| **Visualização**             | Decidir exibição de gráficos (Plotly/Vega).             | `VisualizeDataTool`, `ChartComponent`          | Sim              |
| **Roteamento Geográfico**    | Transmutar para GeoJSON ou provisionar SQLView SLD.     | GeoPandas `to_wgs84=True` / Módulo Externo WMS | Sim (SLD)        |
| **Autoaprendizado**          | Gravar sucessos como nova regra e atalho.               | `ToolMemory`                                   | Não              |


---

## Módulo 2: Interface Frontend Independente (CopilotKit + Leaflet)

O Módulo 2 é um repositório isolado, construído em **TypeScript e React**, gerenciado pelo Vite. Toda a comunicação com o Módulo 1 ocorre via protocolo **AG-UI** (Agent-User Interaction Protocol), permitindo uso contínuo de Generative UI em tempo real para a arquitetura do Pydantic AI.

### Arquitetura de UI e `useCopilotAction`

A interação principal ocorre num layout de tela dividida, dominado por um mapa interativo implementado pela biblioteca **Leaflet**.

- **Perguntas Analíticas e de Estilo:** O usuário digita as perguntas e, opcionalmente, as customizações visuais ("pinte de fundo azul"). Essa intenção trafega completa para o Módulo 1, que processará a resposta analítica e estética em SLD.
- **Manipulação Exclusiva de UI (Independência):** Comandos que não exigem o banco de dados como "feche o painel" ou "retorne ao zoom original" acionam os hooks mapeados do CopilotKit, especificamente o `useCopilotAction`. Quando reconhecidos pelo frontend, eles executam funções nativas (ex: `map.setView([lat, lon], zoom)`) localmente, sem perturbar o backend.
- **Renderização Dinâmica de Resposta:** Se o Módulo 1 retornar `ChartComponent` (Plotly) ou tabelas, modais ricos são injetados. Se o Módulo 1 retornar um GeoJSON de Rota A, o React embutirá esses shapes instantaneamente no Leaflet. Se retornar a Rota B (WMS URL), a UI a injeta organicamente como um Overlay Base via camadas do Leaflet sem sobrecarregar o cliente.

### Modo Homologação e Human In The Loop (HITL)

Ao ativar o modo de testes através da URL (ex: `?test=true`), o CopilotKit viabiliza componentes e abas exclusivas para desenvolvedores.

- Para toda resposta recebida, o bloco de chat expõe o script de consulta SQL executado e o Prompt de Sistema original.
- Caso o SQL exiba deficiência lógica ou o estilo OGC gerado pelo LLM para o GeoServer esteja mal formatado, o especialista na UI de chat pressiona "Sugerir Correção". O novo script corrigido é despachado via API para engatilhar manualmente o módulo de `ToolMemory` (Fase 8 do Módulo 1), inserindo o aprendizado forçado nas redes semânticas.

---

## Módulo 3: Painel de Administração e Governança

Sendo um sistema de acesso isolado com infraestrutura crítica, o Módulo 3 atua como orquestrador das configurações operacionais, permissões e re-treinamento da arquitetura RAG agêntica.

### Personalização Profunda Agêntica

- **Custom Prompts:** Uma tela dedicada na aplicação onde administradores podem moldar "System Prompts" mestres ou baseados em papéis de usuários. Esses dados são persistidos de maneira que o Módulo 1 (usando `UserResolver` do Vanna) os anexe dinamicamente nas intenções dependendo de quem solicita.

### Gestão Programática do Treinamento

- A premissa da extração inicial automática (Módulo 1) será governada aqui. Utilizando as chamadas de API nativas do Vanna como `vn.get_training_data()`, administradores podem exibir tabelas listando todo o conteúdo embarcado nas memórias vetoriais.
- **CRUD Completo:** O Módulo permite aos gerentes invocar exclusões de vetores obsoletos ou edições manuais no Dicionário de Dados, chamando métodos de `vn.remove_training_data(id)` ou adicionando pares novos de Pergunta/SQL diretamente à base vetorial sem onerar arquivos físicos.

### Administração de Projetos e Segurança

- Gerenciamento completo do *Lifecycle Hooks* referente ao limite de conexões (rate limiting) para contas distintas.
- Parametrização do DSN de banco de dados por grupo/projeto, garantindo as definições cruciais de *Row-Level Security* para as execuções.

---

## Módulo 4: Integração Global da Arquitetura

A harmonia entre as partes garante a consistência e escalabilidade sob alta demanda. Abaixo, detalha-se o fluxo de integração macro entre os três módulos em uma operação natural:

1. **Orquestração de Configuração (Fluxo Módulo 3 -> Módulo 1):** O administrador provisiona a conexão do PostgreSQL no Módulo 3, redige os System Prompts estritos e aperta "Geração Base". Isso engatilha os agentes de Setup do Módulo 1 a executarem `jsonb_each_text` no PostGIS para extrair metadados geográficos, que em seguida invocam recursivamente a API `vn.train()` do Vanna 2.0. O banco de embeddings é solidificado.
2. **Entrada da Conversação (Módulo 2 -> Módulo 1):** O usuário faz login no CopilotKit do React e entra no mapa. Uma intenção conversacional ("mostre os municípios que mais produzem soja, pinte de verde") e o Token JWT do usuário trafegam sob o protocolo AG-UI em conexão WebSocket para o Módulo 1.
3. **Filtragem de Permissões e Histórico (Módulo 1 Interno):** A requisição passa pelo middleware/hook do Módulo 1. A identidade `UserResolver` coleta permissões RLS do sujeito. A segurança atua via `ConversationFilter`, mascarando PII e limitando o número de mensagens passadas ao prompt para não exceder limites de tokens.
4. **Inferência e Orquestração Agêntica (Módulo 1 Interno):** O `Agent` embasado por Pydantic AI aciona o `LlmContextEnhancer` para enriquecer o prompt com regras e exemplos do RAG. A ferramenta `RunSqlTool` formula a query. Antes de submeter o script ao banco, o código dispara hooks para bloquear strings DML e medir as custas por meio do comando `EXPLAIN`. Em paralelo, o `Custom Audit Logger` envia telemetria ao Pydantic Logfire.
5. **Formatação Automática e Roteamento Espacial (Módulo 1 Interno):** Os resultados brutos são ingeridos. Se a intenção espacial for leve, é convertida para GeoJSON. Dada a instrução de cor, a orquestração adota a delegação ao GeoServer: um LLM monta o manifest SLD `<CssParameter name="fill">#00FF00</CssParameter>` associado ao SQL da query validada. O Módulo 1 repassa esses parâmetros à API do módulo independente Sql-View-Geoserver. A view WMS é materializada e sua URL é engatilhada de volta ao fluxo do Vanna.
6. **Resposta e Renderização de Ferramentas (Módulo 1 -> Módulo 2):** Os blocos são devolvidos à interface CopilotKit de forma fluida. O React lê os componentes. Se receber uma camada WMS configurada, o Leaflet exibe o mapa colorido; os gráficos acompanhantes entram por modais expansivos.
7. **Sustentação Contínua (Módulo 2/3 -> Módulo 1):** O usuário da interface confirma que a resposta de mapa/gráfico foi exata, invocando o `ToolMemory` (Autoaprendizado). Alternativamente, se rejeitada no modo de homologação (`?test=true`), o corretor redige a reposta ideal enviando pela UI à API do Módulo 1, que atualiza seu banco RAG em tempo real.

