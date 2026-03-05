Aqui está o plano completo em formato Markdown, pronto para você copiar e colar:

# Arquitetura Avançada para Sistemas Text-to-SQL Geoespaciais com Integração Agêntica Multicamadas (Plano Master Atualizado)

A convergência entre Modelos de Linguagem de Grande Escala (LLMs) e bancos de dados relacionais representa um vetor de grande transformação. Para domínios geoespaciais, a complexidade escala ao envolver o PostGIS operando sobre PostgreSQL, exigindo um controle rigoroso sobre a geração de queries, a renderização de milhares de polígonos e a interatividade da interface.

Este plano redefine a infraestrutura em **cinco módulos interdependentes**. Utilizando o **Vanna AI 2.0** e o **Pydantic AI**, a lógica do sistema opera como uma Máquina de Estados (Grafo) lastreada por execução durável. O armazenamento vetorial RAG repousa sobre a extensão nativa `pgvector`, a comunicação com a interface adota o protocolo de estado compartilhado **AG-UI**, e um microsserviço isolado atua como orquestrador do **GeoServer**. Adicionalmente, o sistema adota um padrão de múltiplos agentes (Multi-Agent), onde agentes roteadores, de banco de dados, de design (estilização) e de interface de usuário (UI) colaboram fluidamente.

---

## Módulo 1: Motor Principal Text-to-SQL (Backend API)

Este módulo atua como o cérebro da operação e o servidor de processamento para os Módulos 2 e 3. O fluxo de inferência foi modelado como um **Grafo de Estados** (`pydantic-graph`), substituindo fluxos lineares frágeis por transições interligadas entre sub-agentes especializados.

### Fase 1: Classificador de Intenção (Router Agent)

Antes de qualquer processamento de dados, a mensagem do usuário é interceptada por um Agente Roteador incrivelmente rápido e leve.

* **Análise Semântica Preliminar:** Avalia se a intenção é **Analítica/SQL** (ex: "Mostre-me os estados com maior produção de milho") ou um **Comando de Interface** (ex: "Mude o mapa de fundo para satélite", "Limpe o mapa", "Feche a janela").
* **Delegação (Hand-off):** Se for analítica, encaminha para a Fase 2. Se for visual, devolve via protocolo AG-UI diretamente para o **Agente de UI** (Módulo 2), poupando o banco de dados.

### Fase 2: Gerenciamento de Contexto, Segurança e Memória de Cache

* **Integração de Segurança RBAC e UserResolver:** O agente recebe os tokens de sessão enviados pelo Módulo 2. A classe `UserResolver` processa essas credenciais e aplica as políticas do `AgentConfig`. Ferramentas complexas e dados sob RLS (Row-Level Security) tornam-se visíveis apenas de acordo com o nível de acesso do usuário.
* **Filtros e Cache Semântico (Redis):** O LLM Middleware calcula a similaridade cossenoidal da pergunta contra o histórico. Requisições com aderência superior a 0,97 são resolvidas instantaneamente com a resposta cacheada.

### Fase 3: Motor de Treinamento RAG (Operado pelo Módulo 3)

* **Ingestão Automatizada e Vetorização:** Recebe comandos visuais do Módulo 3 para varrer o esquema do PostGIS e salvar a taxonomia (DDL e Dicionário de Dados) no `pgvector`.
* **Geração Agêntica de Perguntas:** Possui uma rota de API que analisa a DDL conectada e invoca o LLM para sugerir perguntas e respostas SQL automaticamente, devolvendo-as ao Módulo 3 para curadoria.

### Fase 4: Inferência e Execução Durável (Temporal)

* **Durable Execution:** Utiliza o Temporal acoplado ao Pydantic AI. Se a API vacilar durante a extração, o agente recupera o estado e continua de onde parou.
* **Constrição Geométrica (BBox):** O *System Prompt* obriga o uso do filtro espacial de limite `&&` antes de funções caras como `ST_Intersects`.

### Fase 5: Validação Autônoma de Sintaxe e Custos

* **Reescrita via `ModelRetry`:** Erros detetáveis levantam uma instrução `ModelRetry`. O agente internaliza a repulsa e formula uma nova sintaxe.
* **Auditoria de Custo via `EXPLAIN JSON`:** Analisa gargalos antecipadamente e sugere aumento de `work_mem` para o manipulador transacional se necessário.

### Fase 6: Agente Especializado de Estilização (Styling Agent)

* **Interpretação Estética e Algoritmo de Jenks:** Analisa pedidos de cores ou classificações distribuídas. Invoca a biblioteca `jenkspy` para calcular "quebras naturais", separando classes matematicamente.
* **Geração de Estilo:** Converte as classes em **Mapbox Style JSON** (Rotas A/C) ou traduz para **SLD XML** (Rota B).

### Fase 7: Roteamento Geográfico Avançado (Tipagem Discriminada)

A resposta sai sob **Dual Output** (resumo em texto para o LLM e objeto rico tipado para o usuário).

* **Rota A (Vetor Leve - GeoJSON):** Até 500 feições com `ST_SimplifyPreserveTopology`.
* **Rota C (MVT Nativo - Payload Médio):** Entre 500 e 50.000 polígonos via `ST_AsMVT`.
* **Rota B (Delegação a GeoServer):** Acima de 50.000 linhas, despacha SQL e SLD para o Módulo 4.

---

## Módulo 2: Interface Principal Multimodal (CopilotKit, AG-UI e Leaflet)

Este módulo é o ambiente de trabalho (Workspace) principal do usuário final. Desenvolvido em TypeScript e React, ele fornece a experiência conversacional e interativa direta com o banco de dados.

### 2.1 Autenticação e Integração de Acesso

O acesso à interface pode ocorrer de duas formas, geridas na fronteira com o Módulo 1:

* **Acesso Anônimo:** Usuários sem login recebem um token de sessão básico. O `UserResolver` do Vanna AI (Módulo 1) restringe esse perfil a um "Grupo Público", permitindo apenas perguntas limitadas e bloqueando acesso a dados restritos ou execução de comandos avançados.
* **Acesso Autenticado (Login/Senha ou SSO):** O usuário faz login na interface React. O token JWT (ou sessão gerada) é trafegado via protocolo AG-UI a cada interação. O backend valida a identidade e libera o acesso aos dados daquele respectivo cliente corporativo, aplicando as permissões corretas (ex: "Visualizador", "Analista") e injetando regras de *Row-Level Security* diretamente nas queries SQL invisíveis ao usuário.

### 2.2 Respostas Multimodais e Generativas

O usuário realiza perguntas complexas sobre o banco de dados e a interface orquestra as respostas utilizando o conceito de *Dual Output* e *Generative UI*:

* **Texto e Resumos:** O agente responde conversacionalmente no painel de chat.
* **Componentes A2UI (Tabelas e Gráficos):** Quando a resposta envolve dados escalares ou estruturados, o Módulo 1 envia objetos declarativos JSONL. A interface converte isso nativamente em modais expansivos contendo gráficos ricos (ex: Plotly) ou planilhas de dados (DataFrames) dentro da própria janela do chat ou em painéis auxiliares.
* **Mapas Geográficos:** As saídas espaciais (Rotas A, B e C geradas no Módulo 1) são automaticamente renderizadas no componente de mapa web interativo (Leaflet), com a devida estilização de cores e legendas geradas pelo *Styling Agent*.

### 2.3 Comandos de Interface em Linguagem Natural (UI Command Agent)

O ambiente não serve apenas para consultas SQL. O usuário pode instruir a interface como se estivesse usando botões e menus.

* O gancho `useAgent` do CopilotKit atua interceptando comandos do **Router Agent** (Fase 1 do Backend).
* Se o usuário digita *"mude o mapa de fundo para satélite"*, *"dê o zoom inicial"*, *"limpe o mapa"*, *"pinte o mapa atual com transparência de 30%"*, ou *"feche a janela da tabela atual"*, o Módulo 1 reconhece que nenhuma query SQL é necessária. Ele envia uma instrução (tool call) via AG-UI de volta para o *UI Command Agent* rodando no navegador do usuário.
* O front-end React invoca as ações imperativas do Leaflet ou altera os estados dos componentes (React States) em tempo real para satisfazer o comando, proporcionando uma experiência extremamente responsiva sem sobrecarregar o banco de dados.

### 2.4 Integração Contínua e Shared State

A interface mantém uma sincronia fluida com o Módulo 1:

* Toda interação (pan, zoom) que o usuário faz no mapa atualiza um **Shared State** (Estado Compartilhado) de coordenadas via conexão persistente (WebSocket/SSE). Quando o usuário pergunta "quais as vendas *aqui*", o backend já sabe as coordenadas do campo visual sem que o usuário precise digitá-las.

---

## Módulo 3: Painel de Administração, Governança e Curadoria de Dados (Control Plane)

Aplicação visual dedicada a administradores e engenheiros de dados. Atua como o "Control Plane", gerenciando isolamento, segurança e a qualidade do cérebro da inteligência artificial.

* **Gestão de Identidade, Usuários e Grupos (RBAC):** Interface para criar usuários, grupos e mapear permissões funcionais. Esses dados são sincronizados ativamente com o `AgentConfig` do Vanna AI.
* **Gestão de Projetos e Bancos de Dados Alvo:** O administrador cria um Projeto (ex: "Análise Agrícola") e cadastra o PostgreSQL/PostGIS de destino. Todo o treinamento RAG fica contido (sandboxed) neste projeto.
* **Extração Visual e Edição do Dicionário de Dados:** O administrador dispara a varredura automática da DDL e metadados. Um editor de texto rico permite refinar as documentações das colunas antes da vetorização.
* **Laboratório de Treinamento RAG (Questions & SQL):** Interface gráfica para invocar o LLM para deduzir e gerar massivamente pares de perguntas e respostas SQL, permitindo aprovação humana (CRUD) e injeção automática no `pgvector`.
* **Auditoria Analítica e Comutação de Índices:** Interroga `pg_stat_statements` para localizar consultas defasadas, recomendando a troca de índices de GiST para SP-GiST.

---

## Módulo 4: Microsserviço de Provisionamento GeoServer (SQL-View Integrator)

Isolado e altamente resiliente, focado exclusivamente na automação da **Rota B**.

* **Criação Programática de SQL Views:** Dispara requisições `POST` em XML formatadas para a REST API do GeoServer, embutindo o comando analítico complexo na tag `<virtualTable>`.
* **Injeção de SLD Agêntico:** Envia o arquivo XML de estilo criado pelo **Agente de Estilização** e atrela à nova camada base.
* **Retorno e Reciclagem:** Empacota a URL WMS, devolve ao fluxo e destrói as SQL Views temporárias no GeoServer após 24h para limpar o catálogo.

---

## Módulo 5: Integração Global da Arquitetura (Visão do Pipeline)

1. **Setup Visual (Módulo 3 -> Módulo 1):** O administrador cria um projeto, conecta o banco, dispara a extração DDL e aprova as perguntas geradas. O Módulo 1 consolida isso no vetor RAG.
2. **Entrada do Usuário Autenticado/Anônimo (Módulo 2):** O usuário (anônimo ou logado) faz uma pergunta ou dá uma ordem ("limpe a tela") na interface React. O token de sessão viaja via AG-UI e o Vanna define seu nível de acesso.
3. **Classificação (Router Agent):** O Módulo 1 recebe e classifica a intenção.
* *Caminho Visual:* O Roteador repassa o comando para o Agente de UI (Módulo 2) alterar elementos de tela localmente, como aplicar transparências, abrir modais ou focar o mapa.
* *Caminho Analítico:* Submete o pedido ao fluxo de RAG do Vanna, processando de forma segura (Temporal e EXPLAIN).


4. **Design Temático (Agente de Estilo):** Usa bibliotecas como `jenkspy` para criar quebras estatísticas naturais e elabora a formatação visual do mapa baseando-se no contexto.
5. **Decisão de Roteamento Espacial:**
* **Rotas A e C:** Despacha vetores brutos ou MVT (`ST_AsMVT`) para renderização nativa WebGL.
* **Rota B:** Transfere para o Módulo 4 fabricar uma camada WMS no GeoServer.


6. **Entrega Multimodal (Módulo 1 -> Módulo 2):** A interface recebe as saídas duplas. Os modais A2UI levantam gráficos analíticos ou tabelas de dados explicativas, enquanto o mapa Leaflet renderiza e centraliza as feições espaciais estilizadas.