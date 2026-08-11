# Elasticsearch - Seção 2: Getting Started (Resumo de Estudo)

> Resumo contextualizado das aulas 6 a 19, unindo os temas em uma narrativa única: **Elasticsearch vs. OpenSearch → onde e como instalar → arquitetura básica (cluster, nó, documento, índice) → explorando o cluster na prática → escalabilidade (sharding e replicação) → colocando tudo em prática (múltiplos nós e roles de nó)**.

---

## 1. Elasticsearch vs. OpenSearch (Aula 06)

Antes de instalar qualquer coisa, vale entender o contexto do ecossistema, porque hoje existem **dois produtos concorrentes**: Elasticsearch e OpenSearch.

### A história por trás da disputa

- O Elasticsearch nasceu como projeto open source sob licença **Apache 2.0** (uso, distribuição e modificação livres).
- A empresa **Elastic** (por trás do Elasticsearch) começou a desenvolver recursos comerciais pagos, o **X-Pack** (segurança, monitoramento, alerting, machine learning), para gerar receita.
- Em **2015**, a AWS lançou o *Amazon Elasticsearch Service*, o que incomodou a Elastic (violação de marca registrada + falta de parceria oficial). A Elastic reagiu lançando o **Elastic Cloud**, tornando-se concorrente direta da AWS.
- Em **março de 2019**, a AWS lançou o **Open Distro for Elasticsearch**: uma distribuição sob Apache 2.0 que reimplementava, de graça, funcionalidades parecidas com as do X-Pack. Isso pressionou a Elastic, que passou a liberar gratuitamente alguns recursos básicos de segurança/monitoramento (parcialmente em resposta à AWS e à comunidade).
- A tensão resultou em um **processo judicial** (setembro de 2019) da Elastic contra a AWS por infração de marca.
- Em **janeiro de 2021**, a Elastic mudou a licença da parte antes Apache 2.0 do Elasticsearch e Kibana para a **SSPL (Server Side Public License)**, mesma estratégia usada pela MongoDB (e depois pelo Redis, em 2024), para impedir que provedores de nuvem oferecessem o software como serviço gerenciado sem contribuir de volta com o projeto. O código-fonte continua público, mas o software deixou de ser considerado "open source" pela Open Source Initiative, por isso a Elastic hoje usa os termos **"open" e "free"** em vez de "open source".
- Uma semana depois, a AWS anunciou que criaria sua própria versão aberta. Em **abril de 2021** nasceu o **OpenSearch**: um **fork** (cópia independente) da versão **7.10** do Elasticsearch, a última lançada sob Apache 2.0. O Kibana também foi "forkado", dando origem ao **OpenSearch Dashboards**.
- O processo judicial terminou em acordo: a AWS parou de usar a marca "Elasticsearch", renomeando seu serviço para **Amazon OpenSearch Service**.

> 💡 Resumindo o conflito: a Elastic queria retorno financeiro pelo desenvolvimento do software; a AWS queria mais controle e liberdade sobre a tecnologia que usava como base de seu serviço gerenciado. Ambas tinham interesses comerciais legítimos - a "verdade" está no meio-termo.

### Elasticsearch ou OpenSearch: qual escolher?

- As **APIs principais são praticamente idênticas** (mapeamento de campos, indexação, busca, agregações), então dá pra começar aprendendo um e migrar para o outro depois sem grandes problemas, **para o propósito deste curso, a escolha não importa muito**.
- As diferenças ficam mais evidentes nas **soluções verticais/ecossistema** (observabilidade, segurança, enterprise search, IA): a Elastic oferece soluções mais integradas e "prontas", porém com recursos avançados pagos; o OpenSearch depende mais de plugins e é mais "aberto", mas menos polido.
- Critérios relevantes na hora de decidir:
  - **Auto-hospedagem**: no Elasticsearch, alguns recursos exigem licença paga; no OpenSearch, não.
  - **Serviço gerenciado**: se você já usa AWS, o Amazon OpenSearch Service se integra bem com outros serviços da AWS (ex: Kinesis). O **Elastic Cloud** roda sobre AWS, Google Cloud e Azure, e já vem com os recursos comerciais "embutidos" na licença.
  - Existem provedores menores e mais baratos para OpenSearch (ex: **Bonsai**, que tem um plano gratuito permanente, ótimo para estudo/hobby).
- Nada impede usar os dois ao mesmo tempo para propósitos diferentes (ex: Elastic Cloud para observabilidade + OpenSearch para busca).

---

## 2. Onde instalar: visão geral das opções (Aula 07)

Depois de entender o contexto, é hora de escolher onde rodar Elasticsearch/Kibana (ou OpenSearch/OpenSearch Dashboards):

- **Local (na sua máquina)**: builds nativos disponíveis para os principais sistemas operacionais. Exceção: **OpenSearch não tem build nativo para macOS**, nesse caso, é necessário usar Docker ou um serviço gerenciado.
- **Docker**: funciona, mas exige mais configuração; o curso **não cobre essa abordagem** por ter sido historicamente instável entre versões.
- **Serviços gerenciados**: recomendado para quem não quer lidar com instalação, a maioria oferece testes gratuitos, suficientes para acompanhar o curso.

O curso usa **Elasticsearch + Kibana**, mas o aluno pode optar por **OpenSearch + OpenSearch Dashboards** sem prejuízo de aprendizado, dado que as APIs centrais são as mesmas.

---

## 3. Instalação prática (Aulas 08, 09, 10 e 11)

Estas aulas são majoritariamente **passo a passo prático** (cliques, comandos de terminal). Aqui vai o essencial de cada uma:

### 3.1 Hospedando OpenSearch no Bonsai (Aula 08)
Demonstração de como criar uma instância gratuita de OpenSearch + OpenSearch Dashboards usando o provedor **Bonsai**, útil para quem não quer instalar nada localmente. A ferramenta de consultas do OpenSearch Dashboards (**Console**) é equivalente à do Kibana, pois é um fork dele.

### 3.2 Hospedando Elasticsearch & Kibana no Elastic Cloud (Aula 09)
- O **Elastic Cloud** tem **teste grátis de 14 dias**, sem necessidade de cartão de crédito.
- Ao criar um "deployment", você escolhe um caso de uso (Search, Observability, Security, para o curso, qualquer um serve) e o provedor de nuvem/região.
- Informações importantes que você vai precisar depois:
  - **Elasticsearch endpoint** (URL para se comunicar com o cluster via API).
  - **Cloud ID** (usado por produtos como Logstash e SDKs, não necessário neste curso).
  - **API key**: criada na seção "API keys" (dentro de Stack Management), usada no header `Authorization` das requisições HTTP (`ApiKey <chave>`).
- O gerenciamento do próprio *deployment* (nome, hardware, exclusão) é feito na interface do **Elastic Cloud**, não dentro do Kibana.

### 3.3 Instalando localmente no macOS/Linux (Aula 10)
- Elasticsearch e Kibana são apenas **arquivos compactados para extrair**, sem instaladores. Ambos já vêm com suas dependências embutidas (Elasticsearch inclui o **OpenJDK**; Kibana inclui o **Node.js**).
- Iniciar Elasticsearch: `bin/elasticsearch`. Na primeira execução, ele gera automaticamente:
  - Um usuário superusuário (**elastic**) com senha exibida no terminal (guarde-a!).
  - Certificados **TLS** para comunicação interna e externa (HTTPS).
  - Um **token de enrollment** (válido por 30 min) usado para conectar o Kibana com segurança.
- Iniciar Kibana: `bin/kibana` (na porta **5601**). No macOS, pode ser necessário desabilitar o **Gatekeeper** para o diretório do Kibana. Ao acessar a URL exibida no terminal, cole o token de enrollment para configurar a conexão com o Elasticsearch automaticamente.
- Para encerrar: `CTRL + C` no terminal correspondente.

### 3.4 Instalando localmente no Windows (Aula 11)
Processo equivalente ao do macOS/Linux, com uma ressalva: o unzip nativo do Windows pode falhar por causa de caminhos de arquivo muito longos dentro do pacote do Kibana, recomenda-se usar uma ferramenta como o **7-Zip**.

---

## 4. Arquitetura básica do Elasticsearch (Aula 12)

Esta é uma das aulas mais importantes conceitualmente, define o vocabulário que será usado no resto do curso.

| Conceito | Definição |
|---|---|
| **Nó (node)** | Uma instância em execução do Elasticsearch (em máquina física, virtual ou container). |
| **Cluster** | Um conjunto de nós relacionados que juntos armazenam todos os dados. |
| **Documento** | Unidade de informação, um objeto **JSON**. Pode representar qualquer coisa (pessoa, produto, evento etc.). |
| **Índice (index)** | Agrupamento lógico de documentos com características semelhantes (ex: índice "people", índice "departments"). |

Pontos-chave:

- É possível ter **múltiplos clusters**, mas geralmente **um só é suficiente**. Clusters são independentes entre si por padrão (embora buscas entre clusters sejam tecnicamente possíveis, não é algo comum). Divide-se em vários clusters normalmente para separar propósitos (ex: um cluster para busca de e-commerce, outro para APM).
- Um nó **sempre pertence a um cluster**, mesmo sozinho, ele forma automaticamente seu próprio cluster ao iniciar (a menos que seja configurado para entrar em um cluster existente). Um cluster de nó único é aceitável em desenvolvimento, mas tem limitações de disponibilidade e escalabilidade (que serão vistas nas aulas de sharding/replicação).
- Ao indexar um documento, o Elasticsearch guarda o JSON original dentro de um campo chamado **`_source`**, junto com metadados internos.
- **Toda busca é executada contra um ou mais índices**, por isso, ao pesquisar, você sempre especifica em qual(is) índice(s) quer buscar.

---

## 5. Inspecionando o cluster na prática (Aula 13)

Primeira aula "mão na massa" com a API REST, usando a ferramenta **Console** do Kibana (Dev Tools → Console).

- O formato de uma requisição no Console: **verbo HTTP** (GET, POST, PUT, DELETE) + **caminho da requisição** (sem o endereço de rede, que o Kibana já adiciona automaticamente).
- **Cluster Health API**, `GET /_cluster/health`: retorna informações gerais, incluindo o **status do cluster**:
  - 🟢 **green**: tudo saudável.
  - 🟡 **yellow**: funcional, mas com risco (será visto na aula de replicação).
  - 🔴 **red**: problema sério (dados indisponíveis).
- **CAT API** (`_cat`): retorna dados em formato legível por humanos. Exemplos:
  - `GET /_cat/nodes?v` → lista os nós do cluster (o parâmetro `v` adiciona cabeçalhos descritivos).
  - `GET /_cat/indices?v` → lista os índices "normais" do cluster.
- **Índices de sistema**: índices cujo nome começa com `.` (ponto), usados internamente por Elasticsearch e Kibana (ex: Kibana guarda suas próprias configurações e dashboards dentro de um índice do Elasticsearch, já que não tem banco de dados próprio). Ficam ocultos por padrão; para vê-los, é preciso usar o parâmetro `expand_wildcards`.
- Convenção: nomes de API sempre começam com `_` (underscore), ex: `_cluster`, `_cat`.

---

## 6. Enviando queries com cURL (Aula 14)

Até aqui, todas as consultas foram feitas pelo **Console** do Kibana, a forma mais fácil, pois formata a resposta, define headers automaticamente e oferece autocompletar. Mas o Elasticsearch expõe uma **API REST comum**, acessível por qualquer cliente HTTP (cURL, Postman etc.).

Passo a passo dos obstáculos comuns ao usar cURL contra uma instância local:

1. **Usar HTTPS**, não HTTP puro, a partir da versão 8, o Elasticsearch exige TLS.
2. **Erro de certificado**: o certificado é autoassinado por padrão. Soluções:
   - Rápida (menos segura): flag `--insecure` (ignora a validação do certificado).
   - Correta: apontar para o certificado da CA com `--cacert config/certs/http_ca.crt` (ou caminho absoluto).
3. **Autenticação**: usar `-u usuario` (o cURL pede a senha interativamente) ou `-u usuario:senha` (menos seguro, pois expõe a senha no terminal/histórico).
4. **Enviar corpo JSON** (ex: para o endpoint de busca `_search`): usar `-d '{...}'` **e** o header `-H "Content-Type: application/json"` (obrigatório, senão o Elasticsearch rejeita por assumir um envio tipo formulário).
   - No Windows, aspas simples não funcionam como no macOS/Linux, é preciso usar aspas duplas e escapar as aspas internas com `\`.

> 💡 O curso continuará usando o Console do Kibana por conveniência, mas é bom saber replicar isso com cURL/Postman quando necessário (ex: integração com uma aplicação real).

---

## 7. Sharding e escalabilidade (Aula 15)

Um dos motivos do Elasticsearch escalar tão bem: a capacidade de **dividir um índice em pedaços menores** chamados **shards**.

- **Sharding acontece no nível do índice**, não do cluster ou do nó (faz sentido, já que um índice pode ter 1 bilhão de documentos e outro apenas algumas centenas).
- **Motivo principal**: permitir que um índice cresça além da capacidade de disco de um único nó (ex: um índice de 600 GB não cabe em um nó de 500 GB, mas dividido em shards menores, cabe distribuído entre nós).
- **Motivo secundário**: permitir que buscas sejam **paralelizadas e distribuídas** entre os shards (e, portanto, entre nós), aumentando a performance.
- Cada shard é, na prática, um **índice Lucene independente**, um índice Elasticsearch de 5 shards é, por baixo dos panos, 5 índices Lucene.
- Limite técnico: um shard pode conter no máximo ~**2 bilhões de documentos**.
- **Padrão desde a versão 7**: cada índice novo é criado com **1 shard** (antes da v7 eram 5 por padrão, o que causava "over-sharding" em índices pequenos, muito desperdício). Hoje, se for preciso, dá pra aumentar o número de shards com a **Split API** (ou diminuir com a **Shrink API**), mas isso ainda envolve criar um novo índice.
- **Não existe fórmula mágica** para o número ideal de shards, depende de fatores como número de nós, capacidade de hardware, volume de consultas etc. Regra prática: se você espera **milhões de documentos**, considere começar com ~**5 shards**; caso contrário, o padrão de 1 já é suficiente (e você pode aumentar depois, se necessário).

---

## 8. Entendendo a replicação (Aula 16)

Sharding resolve o problema de volume de dados, mas **não** resolve o problema de **perda de dados** em caso de falha de hardware, é aí que entra a **replicação**.

- A replicação é **habilitada por padrão, sem configuração**.
- Funciona criando **cópias de cada shard**, chamadas de **réplicas (replica shards)**.
- Um shard que possui réplicas é chamado de **shard primário (primary shard)**; o primário + suas réplicas formam um **grupo de replicação (replication group)**.
- Uma réplica é uma cópia completa e **totalmente funcional**, pode responder buscas exatamente como o shard primário.
- **Regra de ouro**: uma réplica **nunca** fica no mesmo nó que seu shard primário, assim, se um nó cai, sempre existe ao menos uma cópia em outro nó.
- **Réplicas só têm efeito com mais de um nó no cluster.** Em um cluster de nó único, mesmo configurando réplicas, elas ficam **"unassigned" (não alocadas)** até que outro nó seja adicionado.
- Regra prática de quantas réplicas usar: **1 réplica** para a maioria dos casos; **2 ou mais** para sistemas críticos (ex: um hospital) onde a perda de dois nós simultaneamente não pode acontecer. Isso implica que, em produção, você precisa de **pelo menos 2 nós** para se proteger de perda de dados.

### Replicação x Snapshot - não são a mesma coisa!

| | Replicação | Snapshot |
|---|---|---|
| O que faz | Protege os **dados atuais/ao vivo** contra falha de nó | Cria um **backup pontual** (de índices específicos ou do cluster inteiro) |
| Quando ajuda | Nó cai / hardware falha | Você precisa **reverter** para um estado anterior (ex: uma operação de restruturação de dados deu errado) |
| Automática? | Sim, por padrão | Não, precisa ser configurada/disparada |

> Réplica não substitui snapshot: se você corrompeu seus dados com uma operação errada, a réplica também estará corrompida (ela reflete o estado atual). Snapshots são o que te permite voltar no tempo.

### Replicação também aumenta o throughput (bônus)

Como cada réplica é um índice funcional independente, o Elasticsearch pode **distribuir buscas simultâneas entre o shard primário e suas réplicas**, aproveitando os múltiplos núcleos de CPU disponíveis, aumentando a capacidade de atender buscas concorrentes, mesmo sem adicionar novos nós (desde que haja recursos de hardware disponíveis e nós suficientes para distribuir as réplicas). Isso tem um custo: espaço em disco extra para armazenar cada cópia.

### Sobre o status "yellow" do cluster
Quando um índice tem uma réplica configurada mas não alocada (por falta de outro nó), o índice, e consequentemente o cluster, fica com status **yellow**: totalmente funcional, mas em risco de perda de dados se o único nó cair.

> Curiosidade: os índices internos do Kibana usam uma configuração chamada `auto_expand_replicas` com valor `0-1`, que ajusta automaticamente o número de réplicas: **0 réplicas com um único nó**, e **1 réplica assim que outro nó é adicionado**.

---

## 9. Adicionando mais nós ao cluster (Aula 17 - prática opcional)

Aula prática que demonstra, passo a passo, o comportamento do cluster ao crescer:

1. Extraia uma **cópia limpa** do pacote do Elasticsearch para cada novo nó (nunca copie o diretório de um nó existente, pois ele contém dados).
2. Opcionalmente, configure um `node.name` no arquivo `elasticsearch.yml` para facilitar a identificação.
3. Gere um **token de enrollment** com escopo `node` usando o script `elasticsearch-create-enrollment-token` (só necessário na primeira vez que o nó entra no cluster).
4. Inicie o novo nó com `bin/elasticsearch --enrollment-token <token>`.

**O que acontece ao adicionar um segundo nó:**
- O status do cluster passa de **yellow** para **green**, a réplica pendente do índice criado anteriormente agora tem onde ser alocada.
- Os índices de sistema também ganham réplicas automaticamente (graças ao `auto_expand_replicas`).
- Shard primário e réplica continuam sempre em nós diferentes.

**Ao adicionar um terceiro nó:**
- ⚠️ Atenção: com **3 nós no cluster, é preciso manter ao menos 2 rodando**, não é mais possível operar com um único nó, por causa de como funciona a **eleição de nó mestre (master)** (não é possível eleger um mestre com apenas 1 de 3 nós disponíveis).
- O Elasticsearch **redistribui os shards automaticamente** entre os 3 nós, mesmo que 2 nós já fossem suficientes, evitando deixar hardware ocioso.

**Simulando a queda de um nó:**
- Ao desligar (ou "matar") um nó, os shards que estavam nele ficam **"unassigned"**.
- Existe um **atraso de ~60 segundos** antes de realocar os shards perdidos, proposital, para evitar realocações custosas em caso de falhas de rede momentâneas.
- Após o atraso, os shards são realocados nos nós restantes e o cluster volta ao status **green**.
- Se o nó voltar depois, os shards são novamente rebalanceados entre os nós disponíveis.

---

## 10. Visão geral dos papéis (roles) de nó (Aula 18)

Cada nó pode ter **um ou mais papéis (roles)**, que definem sua responsabilidade dentro do cluster. Por padrão, todo nó tem os três papéis mais comuns: **d**ata, **i**ngest e **m**aster (abreviado como **"dim"** na coluna `node.role` do Kibana).

| Role | Função |
|---|---|
| **master** | Torna o nó **elegível** a ser eleito nó mestre (master node), responsável por ações de escopo do cluster: criar/apagar índices, rastrear nós, alocar shards. Ter essa role não significa ser o master, a eleição é feita por votação entre os nós elegíveis. Em clusters grandes, é comum ter **nós mestres dedicados** para garantir estabilidade (evitando que o master fique sobrecarregado com buscas/indexação). |
| **data** | Permite ao nó **armazenar shards** e executar buscas/modificações sobre eles. É o papel "padrão" mais comum. Em clusters grandes com nós mestres dedicados, essa role costuma ser desabilitada nos nós mestres. |
| **ingest** | Permite ao nó executar **ingest pipelines**, uma sequência de "processadores" que transformam documentos antes de serem indexados (ex: converter um IP em dados geográficos). Pode ser pensado como uma **versão simplificada do Logstash**, embutida no próprio Elasticsearch, útil para transformações simples; para algo mais complexo, o Logstash continua sendo a ferramenta certa. Ferramentas como o Filebeat já usam ingest pipelines internamente (ex: parsing de logs do Apache). |
| **machine learning** (`node.ml` + `xpack.ml.enabled`) | Habilita o nó a executar jobs de machine learning e/ou responder a requisições da API de ML, útil para isolar cargas de ML em nós dedicados. |
| **coordination** (nó de coordenação) | Nó responsável por **coordenar** uma requisição (delegar o trabalho aos nós de dados corretos), mas **não armazena dados por conta própria**. Não existe uma role específica para isso, obtém-se removendo todas as outras roles do nó. Útil como uma espécie de "load balancer" em clusters grandes. |
| **voting-only** | Participa apenas da **votação** para eleição do nó mestre, mas nunca pode ser eleito. Uso extremamente raro, relevante só em clusters muito grandes. |

**Quando mudar os papéis padrão?** Praticamente só em **clusters grandes**, quando o uso de hardware está alto e você já ajustou número de shards/nós. Regra prática: **não mexa nos papéis se não tiver um motivo claro**, os padrões atendem bem a maioria dos casos, e essa mudança sempre pode ser feita depois, quando necessário.

---

## 11. Encerramento da seção (Aula 19)

Recapitulando o que foi construído nesta seção:

- Elasticsearch e Kibana funcionando (local ou via Elastic Cloud).
- Compreensão da **arquitetura básica**: clusters, nós, índices e documentos.
- **Sharding e replicação**, os pilares que permitem ao Elasticsearch escalar e manter alta disponibilidade.
- Introdução rápida a **snapshots** como mecanismo de backup.
- Prática de **adicionar nós** a um cluster de desenvolvimento e configuração de **papéis (roles)** de nó.

Com essa base pronta, a próxima seção do curso passa a tratar de **operações básicas sobre documentos** (CRUD) e outros tópicos práticos do dia a dia com Elasticsearch.

---

## 🔑 Conexões-chave da Seção 2

- A Aula 06 contextualiza **por que existem dois produtos** (Elasticsearch/OpenSearch) antes de qualquer instalação, importante para não se confundir depois ao ver nomes parecidos (Kibana vs. OpenSearch Dashboards, por exemplo).
- As Aulas 07–11 são a parte "mecânica" (instalar e configurar), preparando o ambiente que será usado durante todo o resto do curso.
- A Aula 12 define o **vocabulário fundamental** (cluster, nó, documento, índice) que sustenta tudo que vem depois.
- A Aula 13 dá o primeiro contato com a **API REST na prática** (Console do Kibana), e a Aula 14 mostra a mesma coisa via cURL, reforçando que o Kibana é só um cliente conveniente por baixo dos panos.
- As Aulas 15 e 16 (**sharding** e **replicação**) são conceitualmente a dupla mais importante da seção: uma resolve **volume de dados**, a outra resolve **disponibilidade/perda de dados**, e juntas explicam por que o Elasticsearch escala tão bem.
- A Aula 17 traz esses dois conceitos para a prática, mostrando o comportamento real do cluster (status yellow/green, realocação de shards) ao adicionar/remover nós.
- A Aula 18 fecha o ciclo explicando **como especializar nós** (roles) para clusters maiores, um tópico mais avançado, mas importante conhecer para o futuro.

### Tabela-resumo de status do cluster

| Status | Significado |
|---|---|
| 🟢 **Green** | Todos os shards primários e réplicas estão alocados e funcionando. |
| 🟡 **Yellow** | Todos os shards primários estão OK, mas pelo menos uma réplica está **unassigned** (geralmente por falta de nós suficientes). Funcional, porém em risco. |
| 🔴 **Red** | Pelo menos um shard **primário** está indisponível, dados podem estar inacessíveis. |