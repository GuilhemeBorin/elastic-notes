# Elasticsearch - Seção 3: Managing Documents (Resumo de Estudo)

> Resumo contextualizado das aulas 20 a 37, unindo os temas em uma narrativa única: **criar/apagar índices → operações CRUD em documentos → como o Elasticsearch funciona por dentro (routing, leitura, escrita, versionamento) → controle de concorrência → operações em massa (by query e Bulk API) → importação real de dados**.

---

## 1. Criando e apagando índices (Aula 20)

Ponto de partida da seção: gerenciar índices via API REST.

- **Apagar um índice**: `DELETE /nome_do_indice`
- **Criar um índice** com configurações customizadas: `PUT /nome_do_indice` + corpo JSON com um objeto `settings`:

```json
PUT /products
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 2
  }
}
```

- `number_of_shards`: número de shards primários.
- `number_of_replicas`: número de réplicas por shard.
- ⚠️ **Isso é só para fins didáticos**, em produção, o recomendado é manter os valores padrão, a menos que haja um motivo específico (lembrando da Seção 2: o padrão é 1 shard, e o número de shards não pode ser alterado depois sem reindexar).

Na resposta da criação:
- `acknowledged`: indica se o índice foi criado com sucesso.
- `shards_acknowledged`: indica se o número necessário de shards (por padrão, os primários) foi iniciado antes do timeout.

---

## 2. Indexando documentos (Aula 21)

**Indexar** é o termo técnico correto para "adicionar um documento" (embora o instrutor use os dois termos de forma intercambiável).

### Sem especificar ID (gerado automaticamente)
```
POST /products/_doc
{ "name": "...", "price": ..., "in_stock": ... }
```
- Usa-se **POST** porque o Elasticsearch vai gerar o `_id` automaticamente.

### Especificando um ID
```
PUT /products/_doc/100
{ ... }
```
- Usa-se **PUT** (convenção REST) quando você define o ID manualmente.
- O ID é sempre tratado como **string**, mesmo que pareça um número.

### Detalhes da resposta
- Campo **`_shards`**: mostra em quantos shards a escrita teve sucesso/falha. Ex: se o índice tem 2 shards + 2 réplicas cada, um documento pode ser gravado em até 3 shards (1 primário + suas réplicas, o grupo de replicação daquele shard específico).
- Campo **`_id`**: identificador do documento (gerado automaticamente ou definido por você).

### Criação automática de índice
- Configuração de cluster `action.auto_create_index` (padrão: `true`): se você indexar um documento em um índice que ainda não existe, o Elasticsearch **cria o índice automaticamente**.
- Boa prática: **criar índices explicitamente** em produção, o auto-create é só conveniente em desenvolvimento.

---

## 3. Recuperando documentos por ID (Aula 22)

```
GET /products/_doc/100
```
- Mesmo endpoint da indexação, o **verbo HTTP** é que muda o comportamento (padrão REST).
- O JSON original enviado fica dentro do campo **`_source`** na resposta.
- Se o documento não existir: `"found": false` e sem campo `_source`.
- Buscas por **critério** (não por ID) serão vistas mais adiante, quando o curso chegar em queries de busca propriamente ditas.

---

## 4. Atualizando documentos (Aula 23)

```
POST /products/_update/100
{
  "doc": { "in_stock": 3 }
}
```
- O objeto `doc` contém os campos a alterar (pode também **adicionar campos novos**, como um array `tags`, funciona igual).
- Campo `result` na resposta:
  - `"updated"`: o valor mudou.
  - `"noop"`: você enviou o mesmo valor que já existia (nenhuma mudança real).

### 💡 Como funciona por baixo dos panos
**Documentos no Elasticsearch são imutáveis**, não existe "editar in-place". O que a Update API faz é: **recuperar o documento → aplicar as mudanças → reindexar (substituir) o documento com o mesmo ID**. Ou seja, "atualizar" é, na prática, um "buscar + substituir" feito em uma única requisição pelo Elasticsearch (economizando um round-trip de rede que você teria se fizesse isso manualmente).

---

## 5. Atualizações com script (Aula 24)

Para atualizar um campo **com base no valor atual** (ex: decrementar em 1) sem precisar buscar o valor primeiro, usa-se **scripted updates**.

```
POST /products/_update/100
{
  "script": {
    "source": "ctx._source.in_stock--"
  }
}
```

- `ctx` ("context"): variável que dá acesso ao documento via `ctx._source`.
- Suporta lógica mais complexa: condicionais (`if`), atribuições, etc., é uma linguagem de script embutida.
- Script de uma linha: aspas simples de string normal. Script multi-linha: use **três aspas duplas** (`"""..."""`).
- **Parâmetros dinâmicos** com o objeto `params` (útil quando a query vem de uma aplicação, não digitada manualmente):

```json
{
  "script": {
    "source": "ctx._source.in_stock -= params.quantity",
    "params": { "quantity": 4 }
  }
}
```

### Sobre o campo `result` em scripts
- Diferente de updates "normais", scripts **sempre retornam `"updated"`**, mesmo que nada tenha mudado de fato, **a menos que** você defina explicitamente `ctx.op` dentro do script:
  - `ctx.op = "noop"` → força o resultado a `"noop"` sob uma condição (ex: não decrementar se já estiver em 0).
  - `ctx.op = "delete"` → **apaga o documento** via script (raramente necessário, já que existe o Delete By Query, visto adiante).

---

## 6. Upserts (Aula 25)

**Upsert** = *update* + *insert*: atualiza o documento se ele existir, ou cria um novo se não existir, tudo em uma única requisição.

```json
POST /products/_update/101
{
  "script": { "source": "ctx._source.in_stock++" },
  "upsert": {
    "name": "...", "price": ..., "in_stock": ...
  }
}
```
- Se o documento **não existir**: o conteúdo de `upsert` é indexado como novo documento → `result: "created"`.
- Se o documento **já existir**: o `script` é executado sobre ele → `result: "updated"`.

---

## 7. Substituindo documentos (Aula 26)

Reindexar um documento com o **mesmo ID** substitui totalmente seu conteúdo, é a mesma sintaxe usada para indexar com ID específico (Aula 21):

```
PUT /products/_doc/100
{ "name": "...", "price": ... }
```

⚠️ **Atenção**: campos que existiam antes e não forem incluídos no novo corpo (ex: o campo `tags` adicionado na Aula 23) **desaparecem**, o documento inteiro é substituído, não mesclado.

---

## 8. Apagando documentos (Aula 27)

```
DELETE /products/_doc/100
```
- Mesmo endpoint de recuperação, só muda o verbo, reforçando o padrão REST do Elasticsearch.
- Depois de apagado, uma tentativa de `GET` retorna `"found": false`.
- Apagar **vários documentos por critério** (não só por ID) será visto na Aula 34 (Delete By Query).

---

## 9. Entendendo o routing (Aula 28)

Até aqui, tudo pareceu "mágico", como o Elasticsearch sabe **em qual shard** guardar/encontrar cada documento? A resposta é **routing**.

- **Routing** = processo de resolver **em qual shard** um documento está (ou deve estar), tanto para escrever quanto para ler/atualizar/apagar por ID.
- Fórmula padrão (simplificada): `shard = hash(_routing) % número_de_shards_primários`, onde **`_routing` por padrão é o próprio `_id`** do documento.
- Isso é **transparente para o usuário**, o Elasticsearch cuida disso automaticamente. É possível customizar o routing (tópico mais avançado, fora do escopo desta aula).
- **Benefício extra do routing padrão**: distribui os documentos **de forma equilibrada** entre os shards do índice.

### Metadados internos vistos até aqui
| Campo | O que armazena |
|---|---|
| `_source` | O JSON original enviado |
| `_id` | O identificador do documento |
| `_routing` | O valor usado na fórmula de routing (só aparece se você customizar; por padrão é igual ao `_id`, então fica "invisível") |

### 💡 Por que o número de shards não pode mudar depois de criado o índice
A fórmula de routing **usa o número de shards** como variável. Se esse número mudasse, documentos já indexados poderiam "sumir" (a fórmula apontaria para um shard diferente do que eles realmente estão), e a distribuição ficaria desequilibrada. Por isso, para aumentar/diminuir shards, é necessário criar um **novo índice** e reindexar os dados (as APIs **Split** e **Shrink**, mencionadas na Seção 2, facilitam esse processo).

---

## 10. Como o Elasticsearch lê dados (Aula 29)

Foco em **leitura de um único documento por ID** (busca por critério será vista mais adiante).

1. Um nó recebe a requisição de leitura, esse nó é chamado de **nó coordenador (coordinating node)** para essa requisição.
2. O nó coordenador usa **routing** para descobrir em qual **grupo de replicação** (primary shard + réplicas) o documento está.
3. Em vez de sempre ir direto ao shard primário (o que sobrecarregaria um único shard e não escalaria), o Elasticsearch usa uma técnica chamada **Adaptive Replica Selection (ARS)** para escolher a **melhor cópia disponível** (primária ou réplica) com base em fatores de performance.
4. O nó coordenador envia a requisição ao shard escolhido, recebe a resposta e a repassa ao cliente (aplicação, SDK, Kibana, cURL etc.).

---

## 11. Como o Elasticsearch escreve dados (Aula 30)

Escrita segue um caminho diferente da leitura, mais estrito, por questões de consistência.

1. A requisição passa pelo mesmo processo de **routing**, resolvendo o **grupo de replicação**.
2. Diferente da leitura, **escritas são sempre roteadas para o shard primário** (nunca para uma réplica).
3. O shard primário **valida** a operação (estrutura da requisição, tipos de campo etc.).
4. O shard primário executa a escrita **localmente**, e então a **encaminha em paralelo** para suas réplicas.
5. A operação é considerada bem-sucedida mesmo que a réplica não receba a atualização imediatamente (ela será sincronizada depois).

### Lidando com falhas: Primary Terms e Sequence Numbers

Cenário de risco: o shard primário falha **depois** de enviar a operação para apenas uma de duas réplicas. Uma réplica fica "desatualizada" achando que está em dia, gerando inconsistência (o documento aparece só às vezes).

Para resolver isso, o Elasticsearch usa dois mecanismos:

- **Primary term**: um contador que aumenta toda vez que o **shard primário de um grupo de replicação muda** (ex: quando uma réplica é promovida a primária após uma falha). Serve para distinguir "primárias antigas" de "primárias novas".
- **Sequence number**: um contador incrementado a cada operação de escrita processada pelo shard primário, define a **ordem** em que as operações aconteceram.

Com essas duas informações, o Elasticsearch consegue saber exatamente **quais operações já foram aplicadas** em cada réplica, sem precisar comparar todo o histórico de dados.

- **Global checkpoint**: por grupo de replicação, o sequence number até o qual **todos os shards ativos** já estão sincronizados.
- **Local checkpoint**: por shard de réplica, até onde aquela réplica específica está sincronizada.

Isso permite que, na recuperação, o Elasticsearch compare apenas as operações **posteriores ao checkpoint**, em vez de todo o histórico, muito mais eficiente. Ambos os campos (`_primary_term` e `_seq_no`) aparecem nas respostas de escrita e ao buscar um documento por ID, e serão usados na prática já na próxima aula.

---

## 12. Entendendo o versionamento de documentos (Aula 31)

O Elasticsearch guarda um metadado **`_version`** junto com cada documento.

- Começa em **1**, e é **incrementado a cada update ou delete**.
- Ao **apagar** um documento, o número da versão é mantido por **60 segundos** (por padrão), se um novo documento com o mesmo ID for indexado dentro desse período, a versão continua incrementando; após esse prazo, reinicia em 1.
- ⚠️ **Não é um histórico de revisões**, o Elasticsearch guarda **só a versão mais recente**, não é possível "voltar no tempo" para ver como o documento era antes.
- Esse é o chamado versionamento **"interno"** (o padrão).

### Versionamento "externo"
Para casos onde a versão é controlada **fora** do Elasticsearch (ex: um banco relacional é a fonte primária de dados, e o Elasticsearch só indexa para permitir busca), você mesmo especifica o número da versão e o tipo (`external`) ao indexar.

> ⚠️ Esse mecanismo de versionamento (`_version`) **era**, historicamente, a forma de fazer controle de concorrência otimista, mas hoje **não é mais a prática recomendada**, tendo sido substituído por primary terms + sequence numbers (próxima aula). O campo ainda existe e pode ser útil para outros fins, mas seu uso é limitado hoje em dia.

---

## 13. Controle de concorrência otimista (Aula 32)

Problema clássico de sistemas concorrentes: duas operações "pisando uma na outra" porque ambas leram o mesmo valor antes de a outra escrever sua atualização.

### Exemplo do curso (e-commerce)
1. Dois clientes finalizam a compra do **mesmo produto** quase ao mesmo tempo.
2. Duas threads do servidor web **leem** o `in_stock` (ex: valor = 6) **antes** de qualquer uma delas escrever.
3. Thread 1 calcula `6 - 1 = 5` e atualiza.
4. Thread 2, sem saber que o valor já mudou, também calcula `6 - 1 = 5` (baseado no valor antigo que ela leu) e sobrescreve.
5. Resultado: `in_stock` fica em **5**, quando deveria ser **4**, uma venda "sumiu" sem gerar nenhum erro.

### A solução: `if_seq_no` + `if_primary_term`
Ao recuperar um documento, você já tem seu `_primary_term` e `_seq_no` atuais. Ao enviar o update, você inclui esses dois valores como parâmetros:

```
POST /products/_update/100?if_seq_no=X&if_primary_term=Y
{ "doc": { "in_stock": 5 } }
```

- O Elasticsearch só aplica a atualização se o `_seq_no`/`_primary_term` **atuais do documento** ainda baterem com os valores fornecidos.
- Se o documento tiver sido modificado por outra operação nesse meio tempo → **erro de conflito de versão** (a operação falha).
- Cabe à **aplicação tratar esse erro**: buscar o documento novamente, refazer os cálculos com o valor atualizado, e tentar de novo.

> 💡 Nem toda operação de update precisa disso, só é essencial quando **múltiplos processos/threads podem atualizar o mesmo documento simultaneamente**.

---

## 14. Update By Query (Aula 33)

Equivalente a um `UPDATE ... WHERE` do SQL: atualiza **vários documentos de uma vez**, com base em um critério de busca.

```json
POST /products/_update_by_query
{
  "script": { "source": "ctx._source.in_stock--" },
  "query": { "match_all": {} }
}
```
- `query`: define **quais documentos** serão afetados (aqui, `match_all`, todos; queries reais de busca serão vistas mais adiante no curso).
- `script`: mesma sintaxe usada em updates individuais.

### Como funciona internamente
1. Um **snapshot** do índice é criado no início (para garantir que as atualizações partam de um estado consistente).
2. Uma query de busca é enviada a cada shard do índice, usando internamente a **Scroll API** (para lidar com grandes volumes de resultados em lotes/**batches**).
3. Para cada lote de documentos encontrados, um **bulk request** é disparado para aplicar as atualizações.
4. Buscas e atualizações acontecem **sequencialmente** (um lote de cada vez), para facilitar o tratamento de erros, o Elasticsearch tenta novamente **até 10 vezes** em caso de falha; se ainda assim falhar, a query inteira é **abortada**.

### ⚠️ Importante: não é transacional
Se a query for abortada no meio do processo, os documentos **já atualizados permanecem atualizados**, não há rollback. Isso é um padrão comum em várias APIs do Elasticsearch: quando uma operação pode falhar parcialmente, a resposta traz informações detalhadas (`failures`, `version_conflicts`) para você lidar com isso na aplicação.

- **Conflitos de versão**: o Elasticsearch usa `_primary_term`/`_seq_no` do snapshot para checar se o documento mudou desde então, se mudou, ocorre conflito e (por padrão) a query inteira **é abortada**.
- Para **ignorar conflitos e continuar** em vez de abortar: adicionar `"conflicts": "proceed"` no corpo (ou como query parameter). Os conflitos passam a ser apenas **contados**, sem interromper a execução.

---

## 15. Delete By Query (Aula 34)

Mesmíssima lógica do Update By Query, mas para **apagar** os documentos encontrados:

```json
POST /products/_delete_by_query
{
  "query": { "match_all": {} }
}
```

- Funciona exatamente como o Update By Query por baixo dos panos: batches, primary terms/sequence numbers, tratamento de erro, suporte a `"conflicts": "proceed"`, etc.

---

## 16. Processamento em lote com a Bulk API (Aula 35)

Até aqui, tudo foi feito **um documento por vez**. A **Bulk API** permite indexar, atualizar ou apagar **muitos documentos em uma única requisição**, potencialmente milhares.

### Formato NDJSON
A Bulk API usa um formato especial chamado **NDJSON** (*newline-delimited JSON*): cada linha é um objeto JSON separado, terminado por `\n` ou `\r\n`, **inclusive a última linha**, que deve ficar em branco (senão a requisição falha).

### As 4 ações disponíveis
| Ação | Comportamento |
|---|---|
| `index` | Adiciona o documento; se já existir com o mesmo ID, **substitui**. |
| `create` | Adiciona o documento; **falha se já existir**. |
| `update` | Atualiza (mesma sintaxe do endpoint `_update`, incluindo suporte a script). |
| `delete` | Apaga o documento (a única ação que **não** precisa de uma segunda linha com dados). |

### Exemplo de estrutura
```
POST /_bulk
{ "index": { "_index": "products", "_id": "200" } }
{ "name": "...", "price": ..., "in_stock": ... }
{ "create": { "_index": "products", "_id": "201" } }
{ "name": "...", "price": ..., "in_stock": ... }
```

- Se **todas as ações forem para o mesmo índice**, você pode simplificar usando `POST /products/_bulk` no request path e omitir `_index` dentro de cada ação, útil quando as ações vêm de diversos índices, você mantém `_index` explícito em cada uma.
- Resposta traz um array `items`, na **mesma ordem** das ações enviadas, com o resultado individual de cada uma, permitindo identificar exatamente o que deu certo/errado (a Bulk API também **pode falhar parcialmente**, seguindo o mesmo padrão de design das APIs "by query").

### Detalhes importantes de uso real
- Fora do Console do Kibana (ex: via cURL/scripts), é preciso definir manualmente o header `Content-Type` com o valor específico do NDJSON (embora o Elasticsearch aceite `application/json` "por gentileza", o correto é o tipo NDJSON).
- **Roteamento** funciona igual ao já explicado (Aula 28), aplicado a cada ação individualmente.
- Suporta **controle de concorrência otimista** também: cada ação pode incluir `if_seq_no` e `if_primary_term` na metadata, funcionando exatamente como já visto.

### Quando usar a Bulk API
Ideal quando há **muitas escritas para fazer de uma vez**, muito mais eficiente que centenas/milhares de requisições individuais, pois evita um grande número de round-trips de rede. Na prática, quase sempre gerada por **scripts**, não digitada manualmente, como você verá na importação de dados a seguir.

---

## 17. Importando dados com cURL (Aula 36)

Aplicação prática da Bulk API: importar **1000 documentos de teste** (que serão usados pelo resto do curso) usando **cURL** a partir da linha de comando.

### O arquivo de dados
- Contém 1000 ações do tipo `index`, cada uma em uma linha, com IDs sequenciais.
- Como o **nome do índice não é especificado dentro de cada ação**, ele precisa ser passado no **request path** da requisição.
- O arquivo **deve terminar com uma linha em branco** (requisito do formato NDJSON, já mencionado na aula anterior), erro comum de quem edita o arquivo manualmente.

### O comando cURL

```bash
curl -H "Content-Type: application/x-ndjson" \
     -X POST "https://localhost:9200/products/_bulk?pretty" \
     --cacert config/certs/http_ca.crt \
     -u elastic \
     --data-binary "@arquivo.json"
```

- `-H`: define o `Content-Type` correto para NDJSON.
- `-X POST`: verbo HTTP (obrigatório aqui, não é GET, o padrão).
- `-u elastic` (ou com Elastic Cloud, conforme visto na Seção 2): autenticação.
- `?pretty`: parâmetro opcional só para formatar a resposta de forma legível (pouco útil em scripts reais, já que a resposta normalmente é processada programaticamente).
- **`--data-binary`** (com dois hífens) em vez de `-d`: importante porque preserva as **quebras de linha** do arquivo, outras opções do cURL processam/removem os `\n`, quebrando o formato NDJSON.
- O `@` antes do nome do arquivo indica ao cURL que o valor é um **nome de arquivo**, não uma string literal.

### Por que funcionou mesmo com documentos já existentes?
Como as ações no arquivo usam `index` (não `create`), documentos com IDs já existentes são simplesmente **substituídos**, não é necessário apagar nada antes de reimportar.

### Verificação final
Inspecionando os shards depois da importação, os 1000 documentos ficam **distribuídos de forma razoavelmente equilibrada** entre os shards do índice, resultado direto da estratégia padrão de routing (Aula 28). Pode levar alguns minutos para a contagem de documentos refletir o total esperado.

---

## 18. Encerramento da seção (Aula 37)

Recapitulando o que foi coberto:

- Todas as operações básicas (CRUD) sobre documentos: **indexar, recuperar, atualizar (com/sem script, upsert), substituir, apagar**.
- Como o **routing** funciona e é usado tanto na leitura quanto na escrita.
- **Versionamento de documentos** e **controle de concorrência otimista** (via `_seq_no`/`_primary_term`), para evitar que atualizações concorrentes se sobrescrevam incorretamente.
- Operações **em massa**: Update By Query, Delete By Query e a **Bulk API**, usada para **importar os dados de teste** que serão usados pelo resto do curso.

A próxima seção passa a explorar **buscas (search queries)** propriamente ditas, o "coração" do Elasticsearch, mencionado desde a primeira aula do curso.

---

## 🔑 Conexões-chave da Seção 3

- As Aulas 20–27 formam o **CRUD básico** de documentos, a base prática que sustenta tudo o mais na seção.
- As Aulas 28–30 abrem a "caixa-preta": explicam **routing, leitura e escrita** internamente, dando a base conceitual necessária para entender por que o versionamento e o controle de concorrência (Aulas 31–32) funcionam do jeito que funcionam.
- As Aulas 31 e 32 se conectam diretamente: o **`_version`** (mecanismo antigo) é apresentado primeiro para depois ser **substituído** pela abordagem moderna com **primary term + sequence number**, que, por sua vez, dependem do que foi explicado na Aula 30 (escrita).
- As Aulas 33–36 aplicam tudo isso em escala: Update/Delete By Query usam os mesmos conceitos de conflito de versão e batches; a Bulk API generaliza para qualquer ação (index/create/update/delete); e a Aula 36 é a aplicação prática final, gerando os dados que **serão usados no restante do curso**.

### Tabela-resumo de APIs vistas na seção

| API | Endpoint | Ação |
|---|---|---|
| Criar índice | `PUT /indice` | Cria índice (com settings opcionais) |
| Apagar índice | `DELETE /indice` | Remove índice |
| Indexar (auto ID) | `POST /indice/_doc` | Adiciona documento com ID automático |
| Indexar (ID definido) | `PUT /indice/_doc/{id}` | Adiciona/substitui documento com ID específico |
| Buscar por ID | `GET /indice/_doc/{id}` | Recupera um documento |
| Atualizar | `POST /indice/_update/{id}` | Atualiza campos (via `doc` ou `script`) |
| Apagar | `DELETE /indice/_doc/{id}` | Remove um documento |
| Update By Query | `POST /indice/_update_by_query` | Atualiza vários documentos que casam com uma query |
| Delete By Query | `POST /indice/_delete_by_query` | Apaga vários documentos que casam com uma query |
| Bulk | `POST /_bulk` ou `POST /indice/_bulk` | Executa index/create/update/delete em lote (formato NDJSON) |

### Tabela-resumo: metadados de documento

| Campo | Significado |
|---|---|
| `_source` | JSON original do documento |
| `_id` | Identificador único |
| `_routing` | Valor usado para calcular o shard (padrão = `_id`) |
| `_version` | Contador incremental (versionamento "interno", hoje pouco usado para concorrência) |
| `_primary_term` | Contador de trocas do shard primário de um grupo de replicação |
| `_seq_no` | Número sequencial de cada operação de escrita, usado para ordenação e controle de concorrência otimista |