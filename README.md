# PesquisaElasticBRS
Componente python que aproxima o uso dos operadores do BRS em queries internas do ElasticSearch 

Componente python que aproxima o uso dos operadores do BRS em queries internas do ElasticSearch não há intenção de substituir ou competir com a ferramenta BRS, apenas aproveitar o conhecimento do usuário ao receber critérios usados no BRS (`PROX`, `ADJ`, `COM`) e converter para os critérios do elastic.

`FINALIZANDO TESTES antes de disponibilizar`

## Regras:
 - o elastic trabalha com grupos. Operadores diferentes não podem ser agrupados.
 - como operadores diferentes não podem ser agrupados, não é possível usar PROX ou ADJ antes ou depois de parênteses
 - operadores `PROX` e `ADJ` antes ou depois de parênteses serão transformados em `E`
 - o `NÃO` anter de um termo afeta apenas o termo por ele seguido
 - o `NÃO` antes de um grupo afeta todo o grupo
 - se nos critérios tiver `ADJ` e depois `PROX` ou vice-versa, os termos entre eles serão duplicados para cada grupo ex.: `termo1 prox10 termo2 adj3 termo3` ==> `(termo1 prox10 termo2) E (termo2 ADJ3 termo3)`
 - o elastic trabalha com proximidade (SLOP) sequencial (como o ADJ) ou não sequencial (como o PROX) mas não permite juntar esses operadores nem ter uma distância para cada termo, então será usada a maior distância por grupo criado
 
## Dessa forma, serão criados grupos de termos por operadores
 - `termo1 prox10 termo2 adj3 termo3` ==> `(termo1 prox10 termo2) E (termo2 ADJ3 termo3)`
 - `termo1 prox5 termo2 prox10 termo3` ==> `(termo1 prox10 termo2 prox10 termo3)`

## Curingas:
 - O elastic trabalha com wildcards ou regex mas possui uma limitação de termos retornados pelos curingas
   pois ele cria um conjunto interno de subqueries retornando erro se esse conjunto for muito grande
 - São aceitos os curingas em qualquer posição do termo:
   - `*` ou `$` para qualquer quantidade de caracteres: `dano?` pode retornar `dano`, `danos`, `danosos`, etc.
   - `?` para 0 ou um caracteres: `dano?` pode retornar `dano` ou `danos`.
   - `?` para 0 ou um caracteres: `??ativo` pode retornar `ativo`, `inativo`, `reativo`, etc.
   - `??` para 0 caracteres ou quandos `?` forem colocados: `dan??` pode retornar `dano`,`danos`,`dani`,`danas`,`dan`, etc.
 - Exemplo de erro causado com um curinga no termo `dan????`. Uma sugestão seria usar menos curingas como `dan??`.
 - ERRO: `"caused_by" : {"type" : "runtime_exception","reason" : "[texto:/dan.{0,4}o/ ] exceeds maxClauseCount [ Boolean maxClauseCount is set to 1024]"}`
 - contornando o erro:
   - deve-se controlar esse erro e sugerir ao usuário substituir <b>*<b> por <b>??<b> ou reduzir o número de <b>??<b> que possam retornar muitos termos, principalmente em termos comuns e pequenos

## Correções automáticas 
 - Alguns erros de construção das queries serão corrigidos automaticamente
   - Operadores seguidos, mantém o último: 
     - `termo1 OU E ADJ1 PROX10 termo2` --> `termo1 PROX10 termo2`
   - Operadores especiais (PROX, ADJ) antes ou depois de parênteses, converte para E:
     - `termo1 PROX10 (termo2 termo3)` --> `termo1 E (termo2 E termo3)`
   - Operadores especiais (PROX, ADJ) iguais com diferentes distâncias, usa a maior distância:
     - `termo1 PROX10 termo2 PROX3 termo3` --> `termo1 PROX10 termo2 PROX10 termo3`
   - Operadores especiais (PROX, ADJ) diferentes em sequência, quebra em dois grupos e duplica o termo entre os grupos:
     - `termo1 PROX10 termo2 ADJ5 termo3` --> `termo1 PROX10 termo2 E termo2 ADJ5 termo3`
   - Operadores ou termos soltos entre parênteses, remove os parênteses: 
     - `termo1 (ADJ1) termo2` --> `termo1 ADJ1 termo2` 
     - `(termo1) ADJ1 (termo2)` --> `termo1 ADJ1 termo2` 
 
## Query:
 - A query será construída por grupos convertidos dos critérios BRS para os mais próximos usando os operadores <b>MUST<b>, <b>MUT_NOT<b>, <b>SPAN_NEAR<b> e <b>SHOULD<b>
 - no caso do uso de curingas, serão usados <b>WILDCARD<b> ou <b>REGEXP<b>

## Exemplo de queries transformadas:
 - Escrito pelo usuário: `dano prox5 moral dano adj20 material estetico`
 - Ajustado pela classe: `(dano PROX5 moral) E (dano ADJ20 material) E estetico`
 - Query do Elastic criada: 
 ```json
 {"query": {"bool": 
    {"must": [{"span_near": 
                   {"clauses": [{"span_term": {"texto": "dano"}}, 
                                {"span_term": {"texto": "moral"}}], "slop": 4, "in_order": false}}, 
              {"span_near": 
                   {"clauses": [{"span_term": {"texto": "dano"}}, 
                                {"span_term": {"texto": "material"}}], "slop": 19, "in_order": true}}, 
              {"term": {"texto": "estetico"}}]}}}
 ```
  - Ou você pode omitir os campos e retornar os trechos do texto grifados com os termos encontrados
 ```json
{"_source": [""], "query": {"bool": 
   {"must": [{"span_near": 
                  {"clauses": [{"span_term": {"texto": "dano"}}, 
                               {"span_term": {"texto": "moral"}}], "slop": 4, "in_order": false}}, 
             {"span_near": 
                  {"clauses": [{"span_term": {"texto": "dano"}}, 
                               {"span_term": {"texto": "material"}}], "slop": 19, "in_order": true}}, 
             {"term": {"texto": "estetico"}}]}},
  "highlight": {  "fields": {   "texto": {}   }}} 
 ```
 
## Exemplos de simplificações/transformações (estão nos testes do componente)
 - `'dano Adj moRal'` ==> `'dano ADJ1 moRal'`
 - `'"dano moral'` ==> `'"dano" ADJ1 "moral"')`
 - `'"dano" prox10 "moral"'` ==> `'"dano" PROX10 "moral"')`
 - `'termo1 E termo2 termo3 OU termo4'` ==> `'termo1 E termo2 E (termo3 OU termo4)')`
 - `'termo1 E termo2 termo3 NÃO termo4'` ==> `'termo1 E termo2 E termo3 NAO termo4')`
 - `'termo1 E termo2 termo3 NÃO termo4 ou termo5'` ==> `'termo1 E termo2 E termo3 NAO (termo4 OU termo5)')`
 - `'dano moral e material'` ==> `'dano E moral E material')`
 - `'dano prox5 material e estético'` ==> `'(dano PROX5 material) E estético')`
 - `'dano prox5 material estético'` ==> `'(dano PROX5 material) E estético')`
 - `'estético dano prox5 material'` ==> `'estético E (dano PROX5 material)')`
 - `'estético e dano prox5 material'` ==> `'estético E (dano PROX5 material)')`
 - `'dano moral (dano prox5 "material e estético)'` ==> `'dano E moral E (dano E ("material" ADJ1 "e" ADJ1 "estético"))')`
 - `(dano moral) prova (agravo (dano prox5 "material e estético))' ` ==> `'(dano E moral) E prova E (agravo E (dano ("material" ADJ1 "e" ADJ1 "estético")))')`
 - `'teste1 adj2 teste2 prox3 teste3 teste4' ` ==> `'(teste1 ADJ2 teste2) E (teste2 PROX3 teste3) E teste4')`
 - `'termo1 E termo2 OU termo3 OU termo4' ` ==> `'termo1 E (termo2 OU termo3 OU termo4)')`
 - `'termo1 E termo2 OU (termo3 adj2 termo4)' ` ==> `'termo1 E (termo2 OU (termo3 ADJ2 termo4))')`
 - `'termo1 OU termo2 termo3' ` ==> `'(termo1 OU termo2) E termo3')`
 - `'termo1 OU termo2 (termo3 termo4)' ` ==> `'(termo1 OU termo2) E (termo3 E termo4)')`
 - `'termo1 OU termo2 termo3 OU termo4' ` ==> `'(termo1 OU termo2) E (termo3 OU termo4)')`
 - `'termo1 OU termo2 (termo3 OU termo4 termo5)' ` ==> `'(termo1 OU termo2) E ((termo3 OU termo4) E termo5)')`
 - `'termo1 OU termo2 OU (termo3 OU termo4 termo5)' ` ==> `'termo1 OU termo2 OU ((termo3 OU termo4) E termo5)')`
 - `('dano adj2 mora* dano prox10 moral prox5 material que?ra' ` ==> `'(dano ADJ2 mora*) E (dano PROX10 moral PROX5 material)  que?ra')`
 - `'termo1 OU termo2 nao termo3' ` ==> `'(termo1 OU termo2) NAO termo3')`
 - `'termo1 OU termo2 nao (termo3 Ou termo4)' ` ==> `'(termo1 OU termo2) NAO (termo3 OU termo4)'`
