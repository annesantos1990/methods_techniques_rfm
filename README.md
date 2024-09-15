# Métodos e Técnicas - RFM Analysis

## Segmentação RFM

O intuito desse projeto é segmentar os clientes com base na técnica de Segmentação RFM, que é uma técnica que leva em consideração os valores de

- Recência: Quanto tempo faz que o cliente fez sua primeira compra?
- Frequência: Quantas vezes cada cliente comprou em um certo período de tempo?
- Valor Monetário: Quanto ele comprou.

Com base nessas variáveis, os clientes são classificados de acordo com o Score relativo a cada uma das variáveis acima. 

Vamos ver como é feito o cálculo de cada uma delas e como obter o código através do SQL.

### Cálculo da Recência

Para fazer o cálculo da recência é necessário fazer a diferença entre a data mais recente e a última data da compra:

$$
recencia = \text{data de mais recente} - \text{data de última compra}
$$

A data de última transação foi obtida através do seguinte código:

```sql
MAX(data_transacao) AS data_da_ultima_compra
```

E o cálculo da recência foi obtida pela fórmula DATE_DIFF:

```sql
DATE_DIFF('2022-12-31', transicoes.data_da_ultima_compra,DAY) AS recencia
```

### Cálculo da Frequência

A frequência foi calculada utilizando a fórmula = cont.se(id_cliente - Aba transação). Isso porque na aba “Transacao” contém em cada linha a informação de cada transação e fazendo a contabilização da linhas do id_cliente, eu sei quantas vezes ele apareceu.

O cálculo da frequência que um cliente comprou foi obtido contabilizando os id_transacao pelo agrupamento dos  id_clientes:

```sql
 trasicoes_limpo AS (
  SELECT
  id_cliente,
  COUNT(id_transacao) AS frequencia,
  MAX(data_transacao) AS data_da_ultima_compra
  FROM `projeto_1.transacoes`
  GROUP BY id_cliente)
```

Acima, o GROUP BY  agrupa os id_cliente e através do COUNT(id_transacao), a consulta retorna quantas vezes cada idclient fez uma transação.

### Cálculo do Valor Monetário

O valor monetário foi calculado fazendo as somas de todas as compras para cada cliente da tabela *resumo_compras*:

```sql
SELECT *,
  total_vinho + total_frutas + total_carnes + total_peixes + total_doces + total_outros AS total_compra_cliente
  FROM `projeto_1.resumo_compras`
```

### RFM_SCORE:

O quintil foi escolhido para poder categorizar os clientes segundo a segmentação RFM. Das diferentes formas de segmentar, optei em segmentar pela classificação “Campiões”, “Em Risco”, etc., no qual a classificação se dá pelo Score da Recência, Frequência e Valor Monetário.

Os score são obtidos com base nos valores da recência, frequência e valor monetário. Esses scores tem como base em qual quintil cada valor se encontra. 

Primeiramente, vamos dividir cada um dos valores de recência, frequência e valor monetário em quintis. O score de cada uma das variáveis estará de acordo com o que é apresentado na tabela abaixo:

| **Quintil** | **Recência** | **r_score** | **Frequência** | **f_score** | **Monetário** | **m_score** |
| --- | --- | --- | --- | --- | --- | --- |
| Q1 - 1º quintil | Q1 | 5 | Q1 | 1 | Q1 | 1 |
| Q2  - 2º quintil | Q2 | 4 | Q2 | 2 | Q2 | 2 |
| Q3  - 3º quintil | Q3 | 3 | Q3 | 3 | Q3 | 3 |
| Q4  - 4º quintil | Q4 | 2 | Q4 | 4 | Q4 | 4 |
| Q5  - 5º quintil | Q5 | 1 | Q5 | 5 | Q5 | 5 |

O primeiro quintil (Q1) até o último (Q5), está em ordem crescente. Por isso, a recência começa com a pontuação 5 e termina (Q5) com pontuação 1. Pois a quanto menor a recência do cliente melhor, por isso o Q1 recebe uma maior pontuação.

Já a frequência e o Valor Monetário tem o Q1 com pontuação 1 e Q5 com pontuação 5, pois para essas duas variáveis quanto maior o valor melhor

### Quintil e Score

CTE do código do BigQuery para a divisão das variáveis FRM em quintis:

```
WITH quintile_table AS (
  SELECT
    *,
    NTILE(5) OVER(ORDER BY frequencia) AS f_score,
    NTILE(5) OVER(ORDER BY recencia) AS r_score,
    NTILE(5) OVER(ORDER BY valor_monetario) AS m_score
  FROM `projeto_1.tabela_unificada`
),
```

### Score FM - Média

Existe diferentes formas de classificar os clientes de acordo com as diferentes classificações “Campiões”, “Em Risco”, etc., nesse projeto para classificar esses clientes foi utilizado o  valor do score da recência (r_score) e do valor médio do F_score e M_score ($\frac{f_{score}+m_{score}}{2}$). Assim, logo após obter os scores para cada variável, foi feito o  cálculo da média dos scores de F e M. 

Uma das 

```sql
table_score AS (
  SELECT
    *,
    ROUND((f_score + m_score) / 2.0, 2) AS fm_score,
  FROM quintile_table
)

```

### Segmentação RFM

Com os scores da Recência e a média dos scores da Frequência e Valor Monetário (fm_score), foi feita a concatenação dos valores r_score e de fm_score, obtendo um valor final (**rfm_score**) que o primeiro número representa o r_score e o segundo número (fm_score).

Ex.:

Valor de rfm_score igual 21, o 2 representa um $r_{score} = 2$ e o valor 1 representa um $fm_{score} = 1$, ou seja tanto um quanto o outro score são baixos

A classificação ficou assim:

| **Segmento de Clientes** | **Atividade** | **rfm_score** |
| --- | --- | --- |
| **Campeões** | Comprou recentemente. Compra com frequência. E gasta muito! | 54, 55 |
| **Clientes fieis** | Gasta um bom dinheiro. Com frequentemente. | 34, 35, 44, 45 |
| **Lealdade potencial** | Clientes recentes. Gastaram uma boa quantia. Compraram mais de uma vez. | 42, 43, 52, 53 |
| **Clientes Recentes** | Comprou recentemente. Mas não com frequência. | 51 |
| **Promissor** | Compradores recentes. Mas não gastaram muito. | 41 |
| **Precisam de atenção** | Recência, frequência e valores monetários acima da média. (Pode não ter comprado muito recentemente). | 33 |
| **Prestes a “hibernar”** | Abaixo da média da Recência, Frequência e valores monetários. (Os perderá se não for reativado). | 31, 32 |
| **Em risco** | Gastou muito dinheiro e comprou com freqüência. Mas há muito tempo. (Precisa trazê-los de volta)! | 13, 14, 23, 24 |
| **Não posso perdê-los** | Fez grandes compras e com frequência. Mas há algum tempo. | 15, 25 |
| **Hibernando** | A última compra foi feita a algum tempo. Pouco gasto e baixo número de pedidos. | 11, 22,  12, 21 |
| **Perdido** | Recência, frequência e pontuação monetária mais baixas. | 11, 12, 21, 22 |

Essa classificação foi feita utilizando a fórmula IFS. Para cada uma das classificações existe uma estratégia acionável, apresentada na tabela abaixo:

| **Segmento de clientes** | **Dica acionável** |
| --- | --- |
| **Campeões** | Recompense-os. Eles podem ser os primeiros a adotar novos produtos. Eles são os embaixadores de sua marca. |
| **Clientes Fiéis** | Faça upselling. Ofereça a eles produtos de maior valor. Peça-lhes que deixem comentários. Crie fidelidade. |
| **Lealdade potencial** | Ofereça a eles um programa de associação ou fidelidade, recomende outros produtos. |
| **Clientes Recentes** | Apoie-os em sua integração, faça com que sintam que estão recebendo o que desejam e comece a construir uma relação de confiança com eles. |
| **Promissor** | Crie reconhecimento da marca, ofereça descontos, brindes ou avaliações gratuitas. |
| **Clientes que precisam de atenção** | Lance ofertas e recomendações por tempo limitado com base em suas compras anteriores. |
| **Prestes a “hibernar”** | Compartilhe recursos valiosos com eles, recomende produtos populares, novos descontos, etc. Reconecte-se com eles. |
| **Em risco** | Envie e-mails personalizados para que eles se reconectem, ofereça promoções e coloque recursos valiosos ao alcance deles. |
| **Não posso perdê-los** | Traga-os de volta com lançamentos de novos produtos, não deixe que a concorrência os leve embora, converse com eles. |
| **Hibernando** | Ofereça a eles produtos relevantes e descontos especiais. Recriar o valor da marca. |
| **Perdido** | Tente reconquistá-los com uma campanha personalizada. Se não funcionar, ignore-os. |
|  |  |

De acordo cada perfil de cliente, a loja pode adotar diferentes estratégias para esses grupos e ter uma marketing mais direcionado.

Código da classificação RFM:

```sql
WITH quintile_table AS (
  SELECT
    *,
    NTILE(5) OVER(ORDER BY frequencia) AS f_score,
    NTILE(5) OVER(ORDER BY recencia DESC) AS r_score,
    NTILE(5) OVER(ORDER BY valor_monetario) AS m_score
  FROM `projeto_1.tabela_unificada_4`
),

 rfm_data AS (
  SELECT 
    *,    
    -- Criação do rfm_score, combinando o quartil de Recência com a média arredondada dos quartis de Frequência e Monetário
    CONCAT(
      CAST(r_score AS STRING), 
      CAST(ROUND((f_score + m_score) / 2) AS STRING)
    ) AS rfm_score
   FROM quintile_table
),

-- Classificação dos clientes de acordo com o rfm_score
classified_rfm AS (
  SELECT *,
    CASE
      -- Hibernando
      WHEN rfm_score IN ('11', '12', '21', '22') THEN 'Hibernando'

      -- Prestes a dormir
      WHEN rfm_score IN ('31', '32') THEN 'Prestes a Hibernar'

      -- Promissores
      WHEN rfm_score = '41' THEN 'Promissores'

      -- Novos clientes
      WHEN rfm_score = '51' THEN 'Clientes Recentes'

      -- Em risco
      WHEN rfm_score IN ('13', '14', '23', '24') THEN 'Em risco'

      -- Não posso perdê-los
      WHEN rfm_score IN ('15', '25') THEN 'Não posso perdê-los'

      -- Clientes fiéis
      WHEN rfm_score IN ('34', '35', '44', '45') THEN 'Clientes Fiéis'

      -- Precisam de atenção
      WHEN rfm_score = '33' THEN 'Clientes que precisam de atenção'

      -- Possíveis clientes fiéis
      WHEN rfm_score IN ('42', '43', '52', '53') THEN 'Lealdade Potencial'

      -- Campeões
      WHEN rfm_score IN ('54', '55') THEN 'Campeões'

      -- Perdido
      WHEN rfm_score IN ('11', '12', '21','22') THEN 'Perdido'

      -- Outros, se necessário
      ELSE 'outros'
    END AS rfm_classification
  FROM rfm_data
)

-- Selecionar os dados classificados
SELECT * FROM classified_rfm;

```

Essa forma de classificar foi retirada dessa fonte:

https://github.com/rafaelporfiriobarros/bank-customer-segmentation-india/blob/main/notebooks/eda.ipynb

Tabela final:

https://docs.google.com/spreadsheets/d/1oIfWTNXaLZMVpHAcjwwn8ZEFLap5LQq6Vq1EEp6pBiY/edit?usp=sharing
