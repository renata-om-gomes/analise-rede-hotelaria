# IBM/Laboratoria: Análise de Rede de Hotelaria
## Introdução ao projeto
Para este projeto, o desafio era realizar uma análise de reservas em uma empresa hoteleira. Recebemos um banco de dados de cerca de 119.000 registros com dados históricos de reservas de uma rede hoteleira, e 15.39 MiB.

Os principais objetivos de aprendizado eram:
- Organizar e manipular dados usando os principais comandos do SQL, como SELECT, FROM, WHERE, GROUP BY, ORDER BY, AVG, COUNT). Limpar, filtrar e resumir os dados de acordo com a necessidade do negócio.
- Tomar decisões de negócios com base em dados: resumir e organizar os dados de forma a encontrar informações importantes para apoiar uma decisão de negócios. 
- Visualizar dados em uma ferramenta de Business Intelligence (BI): conhecer o ambiente de desenho do PowerBI, mostrar os principais indicadores e as estruturas em um dashboard que permite ao negócio obter conclusões acionáveis.
- Organizar e comunicar as descobertas: estruturar a análise em um relatório claro e organizado.

## Exploração de dados inicial
#### Segmentação de clientes por estação do ano
```
SELECT *,
    CASE
        WHEN arrival_date_month = 'January' THEN 'Verão'
        WHEN arrival_date_month = 'February' THEN 'Verão'
        WHEN arrival_date_month = 'March' THEN 'Verão'
        WHEN arrival_date_month = 'April' THEN 'Outono'
        WHEN arrival_date_month = 'May' THEN 'Outono'
        WHEN arrival_date_month = 'June' THEN 'Outono'
        WHEN arrival_date_month = 'July' THEN 'Inverno'
        WHEN arrival_date_month = 'August' THEN 'Inverno'
        WHEN arrival_date_month = 'September' THEN 'Inverno'
        WHEN arrival_date_month = 'October' THEN 'Primavera'
        WHEN arrival_date_month = 'November' THEN 'Primavera'
        WHEN arrival_date_month = 'December' THEN 'Primavera'
    END AS arrival_season
FROM `turismo-351822.dados_turismo.tabela2`
```
![Captura de Tela (154)](https://user-images.githubusercontent.com/107365734/174087932-d8342b4a-dbd4-4532-813a-718a532401c3.png)

## Quantificação de impacto monetário
#### Impacto de um cancelamento
- Impacto de um cancelamento devido ao pagamento de comissão de MKT
```
CREATE TABLE `turismo-351822.dados_turismo.tabela10`
AS
Select
  booking_date,
  sum(is_canceled) as canceled_bookings,
  countif (is_canceled=0) as not_canceled_bookigns,
  sum (is_canceled)*1.5 as canceled_booking_payments,
  countif(is_canceled=0)*1.5 not_canceled_booking_payments
FROM turismo-351822.dados_turismo.tabela9
GROUP BY booking_date
ORDER BY canceled_bookings DESC
```
![Captura de Tela (155)](https://user-images.githubusercontent.com/107365734/174090848-800ff9cd-292f-4291-bcd1-344664b92b80.png)

#### Impacto de um cancelamento feito com menos de 3 dias de antecedência
```
CREATE TABLE `turismo-351822.dados_turismo.tabela_51`
as
Select
  booking_date,
  sum(is_canceled) as canceled_bookings,
  countif (is_canceled=0) as not_canceled_bookigns,
  sum (is_canceled)*1.5 as canceled_booking_payments,
  countif(is_canceled=0)*1.5 not_canceled_booking_payments
FROM turismo-351822.dados_turismo.tabela9
GROUP BY booking_date

SELECT *,
    COUNT(number_cancellations_less_3_days) * 120 AS financial_impact
  FROM
    `turismo-351822.dados_turismo.tabela_52`
  GROUP BY
    reservation_date
```
![Captura de Tela (156)](https://user-images.githubusercontent.com/107365734/174091567-6373b0a6-ef5f-4d4e-9cef-7030ebbed339.png)

## Identificação das reservas com maior probabilidade de serem canceladas
#### Hipótese 1: Reservas feitas com antecedência correm alto risco de cancelamento
```
SELECT *,
	CASE
		WHEN lead_time 0 < 15 THEN 'Pouquissima antecedência'
		WHEN lead_time 16 < 30 THEN 'Pouca antecedência'
		WHEN lead_time 31 < 60 THEN 'Média antecedência'
		WHEN lead_time 61 < 90 THEN 'Justa antecedência' 
		WHEN lead_time 91 < 180 THEN 'Boa antecedência'
		WHEN lead_time 181 < 360 THEN 'Grande antecedência'
		WHEN lead_time < 360 THEN 'Gradissima antecedência'
	END AS categoria_de_cancelacao
FROM `turismo-351822.dados_turismo.tabela9`

SELECT 
	AVG (is_canceled) as taxa_de_cancelamento,
	categoria_de_cancelacao,
	FROM `turismo-351822.dados_turismo.tabela12`
	GROUP BY categoria_de_cancelacao
```

#### Hipótese 2: As reservas que incluem crianças têm menor risco de cancelamento
```
SELECT *,
	CASE
		WHEN children > 0 THEN 'com criancas'
		WHEN children = 0 THEN 'sem criancas'
		END AS children_category
	FROM `turismo-351822.dados_turismo.tabela9`
	
SELECT 
	children_category,
	AVG (is_canceled) as taxa_de_cancelamento,
	hotel,
	FROM `turismo-351822.dados_turismo.tabela14`
	GROUP BY children_category, hotel
```

#### Hipótese 3: Os usuários que fizeram uma alteração em sua reserva têm menor risco de cancelamento
```
SELECT *,
	CASE
		WHEN previous_cancellations = 0 THEN 'Sem alterações'
		WHEN previous_cancellations BETWEEN 1 e 10 THEN 'Entre 1 e 10 alterações'
		WHEN previous_cancellations > 10 THEN '10 alterações ou mais'
		END AS catergorias_frequencia_cancelamentos
	FROM `turismo-351822.dados_turismo.tabela9`
	
	SELECT 
		catergorias_frequencia_cancelamentos,
		AVG (is_canceled) as taxa_de_cancelamento,
		hotel,
		FROM `turismo-351822.dados_turismo.tabela16`
	GROUP BY catergorias_frequencia_cancelamentos, hotel
```

#### Hipótese 4: Quando o usuário fez uma solicitação especial, o risco de cancelamento é menor
```
SELECT *,
	CASE
		WHEN total_of_special_requests = 0 THEN 'Sem pedidos especiais'
		WHEN total_of_special_requests BETWEEN 1 e 3 THEN 'Entre 1 e 3 pedidos especiais'
		WHEN total_of_special_requests > 3 THEN 'Mais de 3 pedidos especiais'
		END AS categorias_pedidos_especiais
		FROM `turismo-351822.dados_turismo.tabela9`
	
	SELECT 
		categorias_pedidos_especiais,
		AVG (is_canceled) as taxa_de_cancelamento,
		hotel,
		FROM `turismo-351822.dados_turismo.tabela21`
		GROUP BY categorias_pedidos_especiais, hotel
```

#### Hipótese 5: As reservas que possuem diárias mais baratas o risco é menor
```
SELECT *,
	CASE
		WHEN adr < 100 THEN 'Diária de 100'
		WHEN adr BETWEEN 100 AND 400 THEN 'Diária entre 100 e 400'
		WHEN adr > 100 THEN 'Diária mais de 400'
		END AS categoria_valor_adr
	FROM `turismo-351822.dados_turismo.tabela9`
	
	SELECT 
		categoria_valor_adr,
		AVG (is_canceled) as taxa_de_cancelamento,
		hotel, 
		FROM `turismo-351822.dados_turismo.tabela19`
		GROUP BY categoria_valor_adr, hotel
```
