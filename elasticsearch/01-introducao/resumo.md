# Elasticsearch e o Elastic Stack — Resumo de Estudo (Aulas 1 a 3)

> Resumo contextualizado a partir das transcrições das aulas 01, 02 e 03, unindo os três temas em uma narrativa única: **o que é o Elasticsearch → quais ferramentas o cercam (Elastic Stack) → como tudo isso se encaixa em uma arquitetura real**.

---

## 1. O que é o Elasticsearch? (Aula 01)

O **Elasticsearch** é um motor de busca e análise de dados open source, construído em **Java** sobre o **Apache Lucene**. Seu principal caso de uso é dar poder de busca a aplicações (um blog, uma loja virtual, etc.), permitindo funcionalidades avançadas parecidas com as do Google:

- Autocomplete
- Correção de erros de digitação (typos)
- Destaque de trechos encontrados (highlighting)
- Sinônimos
- Ajuste de relevância dos resultados

**Exemplo guia usado no curso:** uma loja virtual que precisa buscar produtos por nome, mas também filtrar por preço, marca, tamanho, cor, e ordenar por relevância ou preço — considerando também fatores como avaliação (rating) dos produtos para aumentar a relevância.

### Além da busca: análise de dados

O Elasticsearch não serve só para busca full-text. Ele também é usado como **plataforma de analytics**, permitindo:

- Consultar dados estruturados (números) e gerar **agregações** (parecido com `GROUP BY` em bancos relacionais).
- Armazenar logs de aplicações e métricas de servidores (CPU, memória) para monitoramento — isso é chamado de **APM (Application Performance Management)**.
- Armazenar **eventos** genéricos (ex: vendas de lojas físicas) para análises posteriores.
- Usar **machine learning** para:
  - **Previsão (forecasting)**: ex. prever vendas futuras ou volume de acessos ao site, útil para planejamento de capacidade (quantos servidores/atendentes serão necessários).
  - **Detecção de anomalias**: o ML aprende o "normal" dos dados automaticamente (sem regras/thresholds manuais) e avisa quando há um desvio significativo — podendo disparar alertas por e-mail ou Slack.

> 💡 O curso foca principalmente na parte de **busca**, pois é o "coração" do Elasticsearch — o restante das funcionalidades (ML, APM etc.) depende de produtos adicionais do Elastic Stack (visto na Aula 02).

### Conceitos fundamentais de dados

| Elasticsearch | Equivalente em banco relacional |
|---|---|
| **Documento** | Linha (row) |
| **Campo** | Coluna |
| Documento = objeto **JSON** | — |

- A comunicação com o Elasticsearch é feita via **API REST**, e tanto as consultas quanto os documentos são escritos em **JSON**.
- O Elasticsearch é **distribuído por natureza**, o que garante ótima escalabilidade: buscas continuam rápidas mesmo com milhões de documentos.
- É fácil começar a usar, mas é uma tecnologia complexa para explorar todo o seu potencial.

---

## 2. O Elastic Stack: as ferramentas ao redor do Elasticsearch (Aula 02)

O Elasticsearch é o **coração** de um conjunto maior de tecnologias chamado **Elastic Stack**. Essas ferramentas normalmente interagem com o Elasticsearch (embora isso seja opcional para algumas delas) e têm forte sinergia entre si.

### 2.1 Kibana — a interface visual

**Kibana** é a plataforma de visualização e análise que funciona como um "dashboard" para os dados armazenados no Elasticsearch.

- Permite criar visualizações: gráficos de pizza, linha, mapas em tempo real (ex: visitantes do site em um mapa), agregações por navegador, etc.
- É onde se configuram os recursos de **detecção de anomalias e forecasting** (mencionados na Aula 01).
- Também oferece uma interface para gerenciar autenticação/autorização do Elasticsearch.
- **Importante:** Kibana não faz nada que você não pudesse construir sozinho — ele apenas usa a mesma **API REST** do Elasticsearch por trás dos panos, economizando um enorme trabalho de desenvolvimento.
- É possível criar dashboards diferentes para públicos diferentes: sysadmins (CPU/memória), devs (erros de aplicação, tempo de resposta de API) e gestão (KPIs de negócio, vendas, receita).

> Um ponto interessante: você pode usar Elasticsearch + Kibana **apenas como plataforma de analytics**, sem nenhuma funcionalidade de busca para usuários finais — é um caso de uso totalmente válido e comum.

### 2.2 Logstash — pipeline de processamento de dados

Tradicionalmente usado para logs (daí o nome), hoje o **Logstash** é uma ferramenta genérica de **pipeline de processamento de dados**, tratando qualquer entrada como um "evento" (log, pedido de e-commerce, cliente, mensagem de chat etc.).

Um pipeline do Logstash tem **3 estágios**, cada um usando plugins:

1. **Input** (entrada): de onde os dados vêm — arquivo, HTTP, banco relacional, fila Kafka, etc.
2. **Filter** (processamento): como os dados são transformados — parsing de CSV/XML/JSON, enriquecimento de dados (ex: resolver geolocalização a partir de IP), consultas a bancos relacionais, etc.
3. **Output** (saída, chamada de **"stash"**): para onde os dados processados vão — Elasticsearch, Kafka, e-mail, endpoint HTTP, etc.

**Exemplo prático (log de acesso de servidor web):**
- Input: ler o arquivo de log linha por linha.
- Filter: usar um **Grok pattern** (parecido com regex) para transformar a linha de texto não estruturada em campos estruturados (status code, path, IP, etc.).
- Output: enviar o documento já estruturado para o Elasticsearch.

Características adicionais:
- Pipeline definido em uma linguagem própria (parecida com JSON), que suporta condicionais (pipelines dinâmicos).
- Pode rodar múltiplos pipelines na mesma instância.
- É **horizontalmente escalável**.

### 2.3 X-Pack — funcionalidades extras para Elasticsearch e Kibana

O **X-Pack** adiciona um conjunto de recursos extras. Principais áreas:

- **Segurança**: autenticação (integração com LDAP/Active Directory) e autorização (usuários e roles) tanto no Kibana quanto no Elasticsearch — útil para dar acesso somente-leitura a times como marketing/gestão.
- **Monitoramento**: acompanhar performance de Elasticsearch, Logstash e Kibana (CPU, memória, disco) e configurar **alertas** para qualquer situação (não só monitoramento da própria stack) — ex: picos de erro, login suspeito de um usuário em países diferentes numa mesma hora.
- **Relatórios (Reporting)**: exportar visualizações/dashboards do Kibana como **PDF** (sob demanda ou agendados, inclusive com logotipo próprio) ou dados como **CSV**.
- **Machine Learning**: é o X-Pack que fornece essa funcionalidade por trás da interface do Kibana (detecção de anomalias e forecasting, já citados na Aula 01).
- **Graph**: analisa **relacionamentos relevantes** (não apenas populares) nos dados. Exemplo: recomendar produtos relacionados ou a próxima música em um app de streaming. A ideia central é buscar o **"incomumente comum"** — ou seja, distinguir o que é relevante do que é apenas popular (ex: todo mundo usa Google, mas isso não indica relação; já usar StackOverflow indica algo em comum: interesse em programação). O Graph usa a relevância do Elasticsearch para isso e tem interface visual interativa no Kibana.
- **SQL**: permite enviar **queries SQL** para o Elasticsearch (via HTTP ou driver JDBC), que são traduzidas internamente para a linguagem nativa do Elasticsearch, a **Query DSL** (um objeto JSON). Existe até uma **Translate API** que mostra a Query DSL equivalente a uma consulta SQL — ótimo ponto de partida para quem está aprendendo a Query DSL.

### 2.4 Beats — coletores leves de dados

**Beats** é uma família de agentes leves ("data shippers"), cada um com um propósito único, instalados em servidores para enviar dados ao Logstash ou diretamente ao Elasticsearch. Os dois mais usados:

- **Filebeat**: coleta arquivos de log (possui módulos prontos para nginx, Apache, MySQL etc.).
- **Metricbeat**: coleta métricas de sistema (CPU, memória) e de serviços específicos (também com módulos prontos, como nginx/MySQL).

### 2.5 Juntando as peças: Elastic Stack x ELK Stack

- **Elasticsearch** é o centro — onde os dados ficam armazenados.
- Dados entram nele via **Beats**, **Logstash** ou diretamente pela **API**.
- **Kibana** é a interface que visualiza os dados via API.
- **X-Pack** adiciona funcionalidades extras a Elasticsearch e Kibana (segurança, ML, relatórios, etc.).
- O termo **ELK Stack** (Elasticsearch + Logstash + Kibana) é mais antigo, de antes da existência do Beats e do X-Pack. Hoje o termo correto e mais abrangente é **Elastic Stack**, sendo o ELK um subconjunto dele.

---

## 3. Arquiteturas comuns na prática (Aula 03)

Esta aula mostra, de forma progressiva, como uma aplicação real evolui ao adotar o Elastic Stack — conectando diretamente os conceitos das aulas 01 e 02.

### Etapa 1 — Adicionando busca a um e-commerce existente

Cenário inicial: uma aplicação web de e-commerce usa um **banco de dados relacional** para tudo, inclusive busca — mas bancos relacionais não são bons nisso. Solução: adicionar o Elasticsearch.

- A aplicação passa a se comunicar com o Elasticsearch (via HTTP puro ou, mais comumente, uma **client library** para a linguagem usada).
- Ao receber uma busca do usuário, a aplicação consulta o Elasticsearch e retorna os resultados.
- **Sincronização de dados**: sempre que um produto é criado/atualizado no banco relacional, o mesmo precisa ser feito no Elasticsearch — ou seja, os dados ficam **duplicados** propositalmente (isso é considerado uma boa prática nesse contexto, não um problema).
- Para dados **já existentes** (aplicação legada): é preciso migrar os dados com um script (para poucos dados) ou paginação/scroll (para grandes volumes). Existem projetos open source para ajudar, mas escrever o próprio script também é uma tarefa simples e comum.

### Etapa 2 — Dashboard de negócio com Kibana

O time de gestão quer visualizar métricas (pedidos por semana, receita etc.). Em vez de construir uma interface própria, usa-se o **Kibana**: basta rodar uma instância (em uma máquina dedicada, por exemplo) e configurá-la para se conectar ao Elasticsearch. Nenhum dado extra precisa ser enviado — o Kibana só consulta o que já está lá.

### Etapa 3 — Monitorando a infraestrutura com Metricbeat

Com o crescimento do tráfego, é preciso monitorar a saúde do servidor. Usa-se o **Metricbeat** para:

- Coletar métricas de CPU/memória do sistema e do processo da aplicação web.
- Enviar os dados para um **ingest node** do Elasticsearch (com transformações simples, se necessário) — sem precisar do Logstash nessa etapa.
- Um recurso interessante: o Metricbeat pode **configurar automaticamente um dashboard padrão no Kibana**. Isso funciona porque o Kibana guarda suas próprias configurações **dentro do Elasticsearch**, então qualquer instância nova do Kibana já nasce com essas configurações aplicadas.
- A partir daí, é possível configurar **alertas** (ex: CPU/memória atingindo um limite) para saber quando escalar a infraestrutura (adicionar mais servidores).

### Etapa 4 — Monitorando logs com Filebeat

Com o crescimento do time e do código, cresce também o risco de bugs. Além do Google Analytics, é útil monitorar os **logs de acesso e erro** do servidor web:

- Logs de acesso ajudam a monitorar tempo de resposta por endpoint (identificar código problemático em produção).
- Logs de erro ajudam a identificar bugs.
- Ferramenta usada: **Filebeat**, que já vem com módulos prontos para fazer o parsing de logs comuns (inclusive eventos multi-linha, como stack traces), sem exigir configuração extra em casos simples.

### Etapa 5 — Centralizando processamento de eventos com Logstash

Seis meses depois, a arquitetura já tem **múltiplos servidores web** e passa a armazenar também **eventos de negócio** (ex: produto adicionado ao carrinho) para análise no Kibana.

Até aqui, o processamento era feito com **ingest nodes** do Elasticsearch (simples). Mas agora é necessário um processamento mais avançado (enriquecimento de dados, etc.). Fazer isso dentro da própria aplicação web traria dois problemas:

1. O código de negócio ficaria poluído com lógica de processamento de eventos (e poderia aumentar o tempo de resposta, a menos que fosse assíncrono).
2. O processamento acabaria duplicado em vários lugares (múltiplos servidores web, backoffice, possível migração para microsserviços) — dificultando a manutenção.

**Solução:** centralizar tudo com o **Logstash**.

- Os servidores web enviam eventos ao Logstash via HTTP.
- O Logstash processa os eventos e os envia ao Elasticsearch (ou outro destino).
- Isso mantém a aplicação enxuta: ela só precisa **enviar** os dados brutos, e todo o processamento fica centralizado no Logstash.

Quanto aos dados do Metricbeat e Filebeat: geralmente não precisam de processamento customizado (podem ir direto ao Elasticsearch), mas também podem ser roteados pelo Logstash caso se queira centralizar tudo ou aplicar transformações específicas (ex: formatos de log customizados).

### Reflexão final: qual deve ser o papel da aplicação web?

Para operações de CRUD (criar/editar/excluir produto), a abordagem mais simples é a aplicação atualizar o Elasticsearch diretamente — mas isso é mais suscetível a erros (um bug no código pode quebrar o processamento). Uma boa prática de médio/longo prazo é migrar isso também para eventos processados via **Logstash**, deixando a aplicação web em um cenário ideal onde ela **apenas consulta (lê)** o Elasticsearch, nunca escreve diretamente nele. Isso nem sempre é 100% possível na prática, mas é um bom objetivo a perseguir.

---

## 🔑 Conexões-chave entre as três aulas

- A Aula 01 apresenta o **Elasticsearch puro** (busca + analytics + ML) — mas explica que ML e outras funcionalidades avançadas dependem de "produtos adicionais", que são justamente o assunto da Aula 02.
- A Aula 02 detalha essas peças complementares (**Kibana, Logstash, X-Pack, Beats**) que juntas formam o **Elastic Stack**.
- A Aula 03 mostra, na prática, **como e quando** cada uma dessas peças entra em cena conforme uma aplicação real cresce — funcionando como um estudo de caso que amarra tudo: Elasticsearch (busca) → Kibana (dashboard) → Metricbeat (monitoramento de infra) → Filebeat (logs) → Logstash (processamento centralizado de eventos).

### Tabela-resumo das ferramentas

| Ferramenta | Papel | Envia dados para |
|---|---|---|
| **Elasticsearch** | Armazenamento, busca e análise (o núcleo) | — |
| **Kibana** | Visualização e dashboards (consome via API) | — (lê do Elasticsearch) |
| **Logstash** | Pipeline de processamento de eventos (input → filter → output) | Elasticsearch, Kafka, e-mail, HTTP, etc. |
| **Filebeat** | Coleta de arquivos de log | Logstash ou Elasticsearch |
| **Metricbeat** | Coleta de métricas de sistema/serviços | Logstash ou Elasticsearch |
| **X-Pack** | Segurança, monitoramento, relatórios, ML, Graph, SQL | (adiciona recursos a Elasticsearch/Kibana) |
