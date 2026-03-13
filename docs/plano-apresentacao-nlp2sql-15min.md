# Plano de Apresentacao - NLP2SQL em 15 Minutos

## Objetivo da apresentacao

Explicar, para publico leigo, como a estrategia do projeto transforma perguntas em linguagem natural em consultas a banco de dados com apoio do Vanna AI, destacando:

- o problema que a ferramenta resolve;
- por que existe uma etapa de preparo antes do uso;
- como a API executa uma pergunta com seguranca;
- como a resposta pode voltar em texto, tabela, grafico e mapa;
- por que a abordagem em duas etapas aumenta confianca.

## Mensagem central

A ferramenta nao promete "adivinhar dados". Ela prepara o banco para ser entendido pela IA, publica esse conhecimento no Vanna e so entao executa perguntas com validacao, controle de custo e governanca.

## Quantidade recomendada de slides

Recomendacao: **10 slides**.

Motivo:

- 15 minutos pedem uma narrativa objetiva;
- 10 slides permitem cerca de 1 minuto a 1 minuto e 30 segundos por slide;
- esse volume cobre contexto, estrategia, pipeline e beneficios sem ficar corrido.

## Estrutura sugerida

### Slide 1 - Titulo e promessa

**Objetivo:** abrir com a ideia principal.

**Mensagem principal:** permitir que qualquer pessoa consulte um banco de dados com linguagem natural, sem precisar escrever SQL.

**Sugestao de fala:**
"A proposta do projeto e reduzir a distancia entre a pergunta de negocio e a resposta tecnica. Em vez de pedir uma consulta para um especialista, a pessoa pergunta em portugues e o sistema faz o trabalho pesado."

### Slide 2 - O problema que estamos resolvendo

**Objetivo:** mostrar a dor do usuario.

**Mensagem principal:** hoje consultar dados exige saber onde o dado esta, como ele se relaciona e como escrever a consulta.

**Traducao para leigos:**
- o banco guarda a informacao;
- mas conversar com ele ainda e dificil;
- isso limita o acesso ao conhecimento.

### Slide 3 - O que a ferramenta se propoe a facilitar

**Objetivo:** apresentar a proposta de valor.

**Mensagem principal:** a ferramenta atua como uma ponte entre pergunta humana e estrutura tecnica do banco.

**Pontos simples para mostrar:**
- entender perguntas em linguagem natural;
- localizar o contexto certo no banco;
- montar a consulta SQL;
- devolver a resposta em um formato facil de usar.

### Slide 4 - A ideia-chave: antes de responder, a IA precisa aprender o banco

**Objetivo:** preparar o publico para o pipeline em duas etapas.

**Mensagem principal:** a IA nao deve consultar o banco "no escuro"; ela precisa primeiro conhecer tabelas, colunas, regras e exemplos.

**Frase forte sugerida:** "Nao e um passe de magica. E um processo de preparo."

### Slide 5 - Etapa 1: montar a base semantica

**Objetivo:** explicar a primeira metade da estrategia.

**Mensagem principal:** o sistema investiga o banco e cria uma base de conhecimento que ensina a IA a falar a lingua daquele projeto.

**Elementos que o projeto gera:**
- `DDL`: como o banco esta estruturado;
- `Dictionary`: o significado dos dados e dominios;
- `Questions`: exemplos de perguntas com SQL.

**Analogia:** "E como montar um manual e um caderno de exemplos antes de treinar um assistente."

### Slide 6 - Revisao humana e governanca

**Objetivo:** mostrar que nao e um processo cego.

**Mensagem principal:** antes de liberar o uso, pessoas revisam a base semantica e corrigem ambiguidades, erros e termos de negocio.

**Pontos essenciais:**
- a revisao separa rascunho de material confiavel;
- qualquer mudanca posterior pode deixar os embeddings desatualizados;
- isso evita que a IA aprenda algo incorreto.

### Slide 7 - Etapa 2: publicar embeddings no Vanna AI

**Objetivo:** explicar a segunda metade da estrategia.

**Mensagem principal:** depois da revisao, o conhecimento aprovado vira memoria pesquisavel no Vanna, que passa a enriquecer cada nova pergunta.

**Traducao simples:** "O Vanna vira a memoria de apoio do sistema."

**Importante para a narrativa:**
- o projeto separa base semantica de embeddings;
- o Lab so e liberado depois da publicacao dessa memoria.

### Slide 8 - Como uma pergunta vira resposta na API

**Objetivo:** explicar o runtime de forma didatica.

**Mensagem principal:** quando a pergunta chega, a API segue um fluxo controlado, e nao um chute direto do modelo.

**Fluxo simplificado para o slide:**
1. recebe a pergunta;
2. entende a intencao;
3. busca contexto no conhecimento publicado;
4. gera ou adapta SQL;
5. valida seguranca e custo;
6. executa no banco;
7. monta a resposta.

### Slide 9 - Como a resposta chega ao usuario

**Objetivo:** mostrar valor pratico.

**Mensagem principal:** a mesma pergunta pode virar texto, tabela, grafico ou mapa, dependendo do tipo de dado retornado.

**Pontos para destacar:**
- texto para resposta direta;
- tabela para detalhe;
- grafico para comparacao;
- mapa quando houver dimensao geografica.

### Slide 10 - Por que essa estrategia e confiavel

**Objetivo:** fechar com os diferenciais.

**Mensagem principal:** o sistema combina IA com controle operacional.

**Fechamento sugerido em 4 ideias:**
- prepara o banco antes de responder;
- revisa antes de publicar memoria;
- valida SQL antes de executar;
- entrega resposta util, nao apenas texto tecnico.

**Ultima frase sugerida:** "O objetivo nao e so gerar SQL. E permitir acesso mais simples, seguro e util aos dados."

## Slide opcional de reserva

Se quiser 11 slides, adicionar um slide curto entre os slides 8 e 9:

### Slide extra - O que aprendemos com o Vanna AI

**Mensagem principal:** memoria boa ajuda; memoria ruim atrapalha.

**Traducao para leigos:**
- exemplos anteriores devem servir como contexto;
- respostas erradas nao devem virar aprendizado automatico sem validacao;
- por isso a estrategia atual prioriza feedback validado e memoria controlada.

Use esse slide apenas se houver necessidade de destacar a estrategia tecnica em relacao ao Vanna.

## Roteiro de tempo

- Slide 1: 1 min
- Slide 2: 1 min 30 s
- Slide 3: 1 min 15 s
- Slide 4: 1 min 15 s
- Slide 5: 2 min
- Slide 6: 1 min 30 s
- Slide 7: 1 min 30 s
- Slide 8: 2 min 30 s
- Slide 9: 1 min 30 s
- Slide 10: 1 min

Total aproximado: 14 min 30 s

## Recursos visuais recomendados

- usar um diagrama simples de fluxo para os slides 5, 7 e 8;
- evitar muito texto por slide;
- preferir icones e setas para mostrar a passagem "pergunta -> contexto -> SQL -> resposta";
- no slide 9, usar quatro cards visuais: texto, tabela, grafico e mapa;
- no slide 10, encerrar com um resumo em 4 pilares: preparo, revisao, seguranca, resposta util.

## Termos tecnicos que valem simplificar ao falar

- "base semantica" -> "conhecimento organizado do banco"
- "embeddings" -> "memoria de apoio da IA"
- "runtime" -> "o fluxo que roda quando a pergunta chega"
- "guard" / "policy" -> "regras de seguranca"
- "multimodal" -> "mais de um jeito de mostrar a resposta"

## Base usada para montar este plano

- API principal em `api-geo-nlp/app/api/routes/runtime.py`
- Orquestrador de runtime em `api-geo-nlp/app/modules/runtime/orchestrator.py`
- Pipeline de candidatos SQL em `api-geo-nlp/app/modules/runtime/candidate_pipeline.py`
- Integracao com Vanna em `api-geo-nlp/app/integrations/vanna/agent.py`
- Pipeline em duas etapas em `api-geo-nlp/docs/plano-ui-pipeline-duas-etapas.md`
