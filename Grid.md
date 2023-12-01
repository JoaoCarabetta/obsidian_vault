O objetivo é agregar diversas informações geolocalizadas em um grid padronizado. Isso facilita dois processos:
	- Filtragem de regiões de interesse:
		- "Quais as regiões que tem uma população de >200 habitantes, há <1km de uma região com risco de desabamento que tem alguém com dificuldade de locomoção"
		- "Quais regiões com renda abaixo de 1000 reais por família que votaram majoritariamente no PT nas últimas eleições?"
	- Caracterização da região:
		- "Dada essa região qualquer, quais as características socioeconomicas da população?"

A Base dos Dados e o Datalake já tem muitas informações de diversas fontes. Porém, os usuários tem muita dificuldade de saber que informações existem, entender as nuances das bases para extrair informações e aplicar operações geoespaciais. Esse projeto deixaria pronto o cruzamento das informações e disponibilizaria formas simples de acessar essas informações.

# Casos de Uso

Os usuários de tal ferramenta seriam os já usuários da Base dos Dados. Mas, isso deixaria o valor da Base dos Dados mais evidente. Alguns casos são:

- Bancos de desenvolvimento conseguiriam priorizar investimentos em locais importantes.
- O Centro de Operações pode, rapidamente, saber as características de uma região afetada por um desastre e quais são as pessoas mais vulneráveis.
- Um partido ou político pode direcionar esforços de campanha nos territórios
- Empresas de crédito podem enriquecer dados de pessoas via cpf para modelos matemáticos


# Empresas parecidas no mercado

Enriquecimento de dados:
- Uplexis https://uplexis.com.br/solucoes/enriquecimento-de-dados/
- Target Data https://targetdata.com.br/solucoes
- GoSell https://go2sell.com.br/sobre-nos/
- DataSeek https://dataseek.com.br/api/
- BrasilAPI https://brasilapi.com.br/ (Ong / Grátis)

Nenhuma delas oferece uma API online facilmente utilizável tipo Google Maps ou Stripe. Para acessar a API das empresas é preciso preencher um formulário e entrar em contato com um representante (chato para caralho).

Só isso já seria um ótimo avanço.

Inclusive, todas essas empresas seriam clientes nossos.
# Interfaces de Acesso

## **REST API**

- Descrição: Acesso programático aos dados através da localização (ponto), região (poligono) ou cep. Ela teria interface web ou python/R.
- Pros: Dá para manter uma parte aberta, mas cobrar por requisição. Assim como faz o Stripe ou Google Maps. As pessoas podem criar suas aplicações em cima dessa API, se assim desejarem. 
- Cons: Ruim para quem quiser fazer um batch numa base muito grande

## **Bulk**
	
- Descrição: Para usuários que precisam enriquecer muitos dados de uma só vez. Esse processo usaria o BQ ou a base que estamos usando direto. Idealmente seria um interface web em que o usuário joga uma lista de localizações e ceps

## Chatbot

- Descricão: O usuário interage com uma interface de chat para fazer perguntas sobre o território. 
- Pros: As perguntas podem ser feitas trivialmente. Estruturalmente é fácil de resolver o problema pq a query se traduz somente em um filtro de colunas e linhas.


# Infraestrutura


![[Grid Infrastructure.svg]]

## Tabelas
### grid_geometry

Descrição: Contém todos os grids da área de interesse (Rio de Janeiro, Brasil). O melhor grid para usar é o [H3](https://h3geo.org/). As resoluções interessantes para o nível cidades são 7,8 e 9. 

Resolução 9 - Boa para comparar áreas de 'quadras'. Ideal para CEPs e endereços.
![[Pasted image 20231129185402.png]]

Resolução 8 - Boa para comparar regiões de bairros.
![[Pasted image 20231129185553.png]]

Resolução 7 - Boa para comparar áreas de 'bairros' em uma cidade
![[Pasted image 20231129185441.png]]

Schema:

| Column Name      | Description                                  | Type      |
| ---------------- | -------------------------------------------- | --------- |
| id_tile          | Unique identifier for each grid element      | STRING    |
| id_tile_parent   | Identifier for the parent of the grid element | STRING    |
| id_tile_children | Identifiers for the children of the grid element | STRING    |
| resolution       | The resolution or size of the grid element   | INTEGER   |
| geometry         | Geographical data defining the grid shape and position | GEOGRAPHY |

Obs: talvez seja interessante ter partições por uf ou município para aumentar a velocidade das queries. 

### grid_data

Descrição: Contém todos os dados já prontos para seleção. A tabela é 'pivotada' e tem como índice o id_tile. Ela aceita um número indeterminado de colunas com ids no nome que são referenciados na tabela de metadados. 

Schema

| Column Name | Description                                | Type    |
| ----------- | ------------------------------------------ | ------- |
| id_tile     | Unique identifier for each grid element    | STRING  |
| resolution  | The resolution or size of the grid element | INTEGER |
| feature_1   | Description of the first specific feature  | ANY     |
| ...         | Description of intermediate features       | ANY     |
| feature_n   | Description of the nth specific feature    | ANY     |

### metadata_table

Descrição: Responsável por manter um índice dos metadados das colunas de `grid_data`. Essa tabela será usada na filtragem das colunas de `grid_data`. Ela é responsável por manter informações sobre nome, categoria, fonte, data de referência, e qualquer outra informação que seja relevante para selecionar colunas.

Ela é atualizada toda vez que é gerada uma nova coluna no `grid_data`.

Schema:

| Column Name        | Description                                                                                               | Type     |
| ------------------ | --------------------------------------------------------------------------------------------------------- | -------- |
| id_column          | Unique identifier for the column                                                                          | STRING   |
| tier               | Whether column is free, paid, etc...                                                                      | STRING   |
| as_of_datetime     | The date and time when the data was recorded or last updated                                              | DATETIME |
| category           | General classification or type to which the data belongs                                                  | STRING   |
| subcategory        | More specific classification within the broader category                                                  | STRING   |
| name               | The name or title associated with the data in the column                                                  | STRING   |
| value_type         | If it is a quantity, share of space, share of population, border                                          | STRING   |
| aggregation_method | How to aggregate values (sum, avg, max, none, ...)                                                        | STRING   |
| origin_table_id    | Identifier for the table or source from which this data originated `<project_id>.<dataset_id>.<table_id>` | STRING   |
| data_source        | Name or description of the source providing the data                                                      | STRING   |
| last_updated       | The most recent date and time when the data in the column was updated                                     | DATETIME |


### grid_aggregated

Descrição: Agregações espaciais dos dados filtrados e selecionados. Ela é basicamente uma agregação dos valores da células do grid para uma região qualquer. Para a mesma região, é possível gerar agregações com células de diferentes resoluções.

Schema: 

| Column Name     | Description                                                                                  | Type    |
| --------------- | -------------------------------------------------------------------------------------------- | ------- |
| region_id       | Unique identifier for the region id. It can be a neighborhood, municipality, any shape, etc. | STRING  |
| tile_resolution | Tile resolution used in aggregation                                                          | INTEGER | 
| agg_feature_1   | Aggregated feature_1                                                                         | ANY     |
| ...             | ...                                                                                          | ANY     |
| agg_feature_n   | Aggregated feature_n                                                                         | ANY     |


## Processos

### Construção do grid_data e grid_metadata

![[Pasted image 20231130161607.png]]


#### Merge database geom with grid

Já teremos um grid definido. Portanto, é preciso mapear as features que estão nos dados para este grid. Este grid é do tipo um `POLYGON` do tipo `GEOGRAPHY`. E existem dados geoespaciais de 2 tipos que podem ser associados a estes polígonos:  

1. Pontos (`POINT`)
	Estes seriam dados de objetos no espaço definidos a partir de um par (latitude, longitude). Alguns objetos como esse seriam equipamentos públicos (escolas, delegacias, pontos de ônibus), fenômenos naturais (alagamentos, deslizamentos), etc...
	
    Features possíveis de serem construídas:
	  **Quantidade**: número desses pontos por grid. Ex: Y escolas no grid X.
	```sql
	select 
		grid.tile_id,
		count(*) quantidade
	from grid 
	join points
	on st_intersect(grid.geometry, points.geometry)
	group by grid.tile_id
    ```
	  **Distância**: distância de qualquer ponto ao grid. Ex: escola mais próxima a Z metros do grid.
	```sql
	-- Set maximum distance to make it a bit more efficient
	select
		grid.tile_id,
		min(st_distance(grid.geometry, points.geometry)) min_distance
	from grid
	join points
	on st_intersects(st_expand(grid.geometry, YOUR_MAX_DISTANCE), points.geometry)
	group by grid.tile_id   
	```
	  **Valores**: agregação de valores dos objetos pontuais. Ex: número de votos, número de alunos.
	  ```sql
	select 
		grid.tile_id,
		sum(feature_1) agg_feature_1,
		...
	from grid 
	join points
	on st_intersect(grid.geometry, points.geometry)
	group by grid.tile_id
	 ```
		
2. Polígonos (`POLYGON`)
	Estes seriam dados de regiões definidas por um conjuntos de pares (latitude, longitude). Exemplos: setores censitários, limites geopolíticos/naturais (bairros, cidades, estados, biomas, reservas)

    Nós usualmente usamos a proporção da área ocupada pelo polígono na célula ou da célula no polígono para fazer construir as features. Essa query vai ser utilizada muitas vezes, então podemos criar um `PROCEDURE` para abstrai-la. 
    
	```sql 
	-- share_grid_polygon
	select 
		grid.*,
		polygon.*.
		st_area(st_intersection(grid.geometry, polygons.geometry) / st_area(polygons.geometry) share_polygon,
		st_area(st_intersection(grid.geometry, polygons.geometry) / st_area(grid.geometry) share_grid
	from grid 
	join polygons
	on st_intersect(grid.geometry, polygons.geometry)
	```


 Features possíveis de serem construídas:


   **Pertencimento**: Basicamente definir qual é a região daquela célula. É uma informação útil para os usuários rapidamente filtrarem regiões. Ex: qual o bairro, estado, setor censitário daquela célula. 

   ```sql
   -- Seleciona polígono que ocupa maior proporção da célula
	select
		tile_id,
		max_by(polygon_id, share_grid) polygon_name
	from share_grid_polygon
	group by tile_id 
```
   
   
   **Valores**: Agregação de valores destas regiões. Ex: população de um setor censitário, renda média, presença policial.
   
  ```sql
  	select
		tile_id,
		-- feature baseado na proporção de intersecção da célula
	    -- no polígono. Ex: população do censo. 
		sum(polygon_feature * share_polygon) feature_a,
		-- feature baseado no valor máximo/mínimo/média
		max(polygon_feature) feature_b,
		-- feature baseada na intersecção máxima 
		max_by(polygon_feature, share_polygon|share_grid)
	from share_grid_polygon
	group by grid.tile_id 
  ```

#### Atualização da `metadata_table`

Cada feature adicionada terá que atualizar o `metadata_table` com suas informações. Isso é relativamente simples, basta adicionar uma query de UPDATE com os metadados.

A boa notícia é que podemos criar um `PROCEDURE` para abstrair a query e basta chamar uma 

```sql
CREATE OR REPLACE PROCEDURE update_grid_metadata(
  column_id_param STRING,
  metadata_field1_param STRING,
  metadata_field2_param STRING
)
BEGIN
  UPDATE grid_metadata
  SET metadata_field1 = metadata_field1_param,
      metadata_field2 = metadata_field2_param
  WHERE column_id = column_id_param;
END;

-- Try to insert if column_id is new 
INSERT INTO grid_metadata (tile_id, metadata_field1, metadata_field2) 
VALUES (tile_id_param, metadata_field1_param, metadata_field2_param)
WHERE column_id_param NOT IN (select column_id from grid_metadata)
; 

-- Chamando o PROCEDURE
CALL update_grid_metadata('column_id', 'new_value_for_field1', 'new_value_for_field2');
```

#### Resumo

Para fazer uma nova feature, basta entender qual o tipo de dado que estamos lidando, juntar os dados com o grid e adicionar metadados na tabela.

Um exemplo de como faríamos para adicionar dados populacionais. Primeiro, sabemos que os dados populacionais estão no nível de setor censitário. Então, assumindo que já temos a tabela de população por setor censitário,  `populacao`. Podemos calcular a quantidade de pessoas por célula no grid.

```sql
-- calcula o share_grid_polygon 
CALL share_grid_polygon('populacao', 'geometry', 'share_populacao'); -- table_id, geom_name, temp_table_name

-- Calcula população por grid e adiciona no `grid_data`
CALL update_table(
	'select
		id_tile,
		sum(v002 * share_polygon) populacao_2010
	from share_grid_polygon
	group by grid_id' -- feature query,
	'populacao_2010' -- feature name
);

-- adiciona os metadados 
CALL update_grid_medata(
	
)
```
### Consumo do `grid_data`

O objetivo é entregar as informações pedidas (colunas) na região específica (valores) e, se requisitado, agregada. 
Portanto, são possivelmente três etapas, filtro de colunas, valores e agregação.

**Filtro de colunas**

1. Recebe query do usuário
2. Filtra a tabela `grid_metadata` de acordo com os parâmetros. 
3. Seleciona as colunas do `grid_data` para serem filtradas por valor a partir da seleção do `grid_metadata`

**Filtro de valores**

1. Recebe query do usuário e as colunas selecionadas para o filtro de valores
2. Aplica filtro de valores em colunas específicas de acordo com a query

**Agregação Espacial**

1. Recebe query do usuário e `grid_data` com colunas e valores filtrados
2. Agrega valores espacialmente de acordo com método de agregação do  `grid_metadata`

Exemplos:

1. Quais as regiões que tem uma população de >200 habitantes, há <1km de uma região com risco de desabamento que tem alguém com dificuldade de locomoção.
	1. Filtro de colunas: seleciona colunas de habitantes, distancia de risco de desabamento e população com dificuldade de locomoção na tabela de metadados.  
	2. Filtro de valores: Recebe as colunas para filtragem e aplica os filtros:  valores >200 para habitantes, <1km de distancia de risco de desabamento e >0 de população com dificuldade de locomoção.
	3. Não há necessidade de agregação.
	4. Retorna todos a tabela `grid_data` completa 
