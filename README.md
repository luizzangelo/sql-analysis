# 📊 SQL Insights for E-commerce

This repository collects SQL queries used for **real marketing and sales analyses** in an e-commerce project, aiming to deliver practical and strategic business insights. Each analysis includes the business question, the SQL query used, and a brief interpretation of the results.

---

## 🧱 Table Structure

**dim_cliente** : customer data (CPF, name, email, birth date, location).  
**dim_produto** : product catalog (ID, SKU, name, category).  
**fato_vendas** : transactional sales data (order, product, value, status, shipping, customer).

---

## 📋 Business Questions and SQL Analyses

### 1. What are the top 10 neighborhoods with the highest number of registered customers?  
```sql
SELECT 
  bairro,
  COUNT(*) AS total_cliente,
  ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER (), 2)||'%' AS percentual_clientes
FROM 
  dim_cliente
GROUP BY 
  bairro
ORDER BY 
  total_cliente DESC
LIMIT 10;
````
>**Insight:** Helps identify priority regions for local campaigns, giveaways, or pickup partnerships.  
> ![Top 10 neighborhoods](https://github.com/luizzangelo/sql-analysis/blob/main/TOP_10_NEIGH.png?raw=true)

### 2. What is the total sales value per month in the last year?

```sql
SELECT 
  DATE_PART('month', data_criacao) AS mes,
  SUM(valor_total - valor_envio) AS total_vendido
FROM (
  SELECT DISTINCT pedido_numero,
    cpf_cliente, endereco_entrega_nome,
    valor_total, valor_envio, data_criacao
  FROM fato_vendas
  WHERE pedido_situacao NOT IN ('Pedido Cancelado', 'Pagamento devolvido')
) pedidos_unicos
WHERE DATE_PART('year', data_criacao) = 2025
GROUP BY DATE_PART('month', data_criacao)
ORDER BY mes;
```
>**Insight:** Reveals sales seasonality and supports planning promotions for slower months.
> ![total sales](https://github.com/luizzangelo/sql-analysis/blob/main/SALES_LAST_YEAR.png?raw=true)

### 3. Which are the top 5 best-selling products by quantity?

```sql
SELECT 
  produto_nome, 
  COUNT(*) AS qtd_vendido
FROM fato_vendas
GROUP BY produto_nome
ORDER BY qtd_vendido DESC
LIMIT 5;
```
>**Insight:** Highlights the most popular items for featured promotions and stock prioritization.
> ![top 5 best-selling](https://github.com/luizzangelo/sql-analysis/blob/main/TOP_5_PRODUCTS.png?raw=true)

### 4. How many orders were placed by each payment method?

```sql
SELECT 
  pagamento_nome, 
  COUNT(*) AS qtd
FROM (
  SELECT DISTINCT pedido_numero, cpf_cliente, pagamento_nome
  FROM fato_vendas
  WHERE pedido_situacao NOT IN ('Pedido Cancelado','Pagamento devolvido')
) pedidos_unicos
GROUP BY pagamento_nome
ORDER BY qtd DESC;
```
>**Insight:** Shows preferred payment options and informs strategic offers (e.g., Pix, boleto, credit card).
> ![payment method](https://github.com/luizzangelo/sql-analysis/blob/main/METHOD_PAYMENT.png?raw=true)

### 5. What is the average ticket per customer?

```sql
SELECT 
  cpf_cliente,
  MIN(endereco_entrega_nome) AS nome_cliente,
  COUNT(*) AS qtd_pedidos,
  SUM(valor_comprado_real) AS total_spent,
  ROUND(SUM(valor_comprado_real) / COUNT(*), 2) AS avg_ticket_per_customer
FROM (
  SELECT DISTINCT pedido_numero, cpf_cliente, endereco_entrega_nome,
         (valor_total - valor_envio) AS valor_comprado_real
  FROM fato_vendas
  WHERE pedido_situacao NOT IN ('Pedido Cancelado','Pagamento devolvido')
    AND endereco_entrega_nome != 'CLIENTE WPP'
) pedidos_unicos
GROUP BY cpf_cliente
ORDER BY total_spent DESC;
```
>**Insight:** Enables segmentation for campaigns aimed at increasing average spend and loyalty programs.
> ![average ticket](https://github.com/luizzangelo/sql-analysis/blob/main/AVERAGE_TICKET.png?raw=true)

### 6. Which product categories generate the highest revenue?

```sql
SELECT
  categoria,
  SUM(preco_produto) AS total_revenue
FROM (
  SELECT 
    fv.produto_id,
    fv.produto_nome,
    fv.preco_venda AS preco_produto,
    dp.categoria
  FROM fato_vendas fv 
  JOIN dim_produto dp ON dp.id_produto = fv.produto_id
  WHERE pedido_situacao NOT IN ('Pedido Cancelado','Pagamento devolvido')
) sales_by_category
GROUP BY categoria
ORDER BY total_revenue DESC
LIMIT 10;
```
>**Insight:** Guides inventory prioritization and targeted category promotions.
> ![best categories](https://github.com/luizzangelo/sql-analysis/blob/main/TOP_10_CATEGORIES.png?raw=true)

### 7. What percentage of customers are repeat buyers (2+ orders)?

```sql
WITH filtered_orders AS (
  SELECT DISTINCT pedido_numero, cpf_cliente
  FROM fato_vendas
  WHERE pedido_situacao NOT IN ('Pedido Cancelado','Pagamento devolvido')
    AND endereco_entrega_nome != 'CLIENTE WPP'
),
orders_per_customer AS (
  SELECT cpf_cliente, COUNT(*) AS order_count
  FROM filtered_orders
  GROUP BY cpf_cliente
),
repeat_customers AS (
  SELECT cpf_cliente
  FROM orders_per_customer
  WHERE order_count > 1
)
SELECT
  ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM orders_per_customer), 2) || '%' 
    AS repeat_customer_percentage
FROM repeat_customers;
```
>**Insight:** Measures customer loyalty and informs retention strategies.

### 8. Which categories sell the most in each neighborhood?

```sql
SELECT
  fv.endereco_entrega_bairro,
  dp.categoria,
  COUNT(*) AS qtd_vendas
FROM fato_vendas fv 
JOIN dim_produto dp ON dp.id_produto = fv.produto_id
WHERE pedido_situacao NOT IN ('Pedido Cancelado','Pagamento devolvido')
GROUP BY fv.endereco_entrega_bairro, dp.categoria
HAVING COUNT(*) > 1
ORDER BY fv.endereco_entrega_bairro, qtd_vendas DESC;
```
>**Insight:** Enables geo-targeted campaigns and optimized local inventory mixes.
> ![best categories](https://github.com/luizzangelo/sql-analysis/blob/main/MOST_CATEGORIES_SALES.png?raw=true)

### 9. What is the average ticket per product category?

```sql
SELECT
  categoria,
  SUM(preco_produto) AS total_revenue,
  COUNT(*) AS total_sales,
  ROUND(SUM(preco_produto) / COUNT(*), 2) AS avg_ticket_category
FROM (
  SELECT 
    fv.produto_id,
    fv.produto_nome,
    fv.preco_venda AS preco_produto,
    dp.categoria
  FROM fato_vendas fv 
  JOIN dim_produto dp ON dp.id_produto = fv.produto_id
  WHERE pedido_situacao NOT IN ('Pedido Cancelado','Pagamento devolvido')
) sales_by_category
GROUP BY categoria
ORDER BY avg_ticket_category DESC
LIMIT 10;
```
>**Insight:** Identifies high-value categories and opportunities for cross-sell promotions.
> ![ticket per category](https://github.com/luizzangelo/sql-analysis/blob/main/AVERAGE_TICKET_CATEGORY.png?raw=true)

### 10. How are customers distributed by age group?

```sql
SELECT
  COUNT(*) AS customer_count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) || '%' AS customer_percentage,
  CASE
    WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, data_nascimento)) BETWEEN 0 AND 17 THEN '0-17 years'
    WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, data_nascimento)) BETWEEN 18 AND 23 THEN '18-23 years'
    WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, data_nascimento)) BETWEEN 24 AND 29 THEN '24-29 years'
    WHEN EXTRACT(YEAR FROM AGE(CURRENT_DATE, data_nascimento)) BETWEEN 30 AND 35 THEN '30-35 years'
    ELSE '35+ years'
  END AS age_group
FROM dim_cliente
GROUP BY age_group
ORDER BY customer_count DESC;
```
>**Insight:** Helps tailor marketing messages to demographic segments.
> ![ticket per product](https://github.com/luizzangelo/sql-analysis/blob/main/AGE_GROUP.png?raw=true)

### 11. What is the total sold per year/month?

```sql
SELECT 
  TO_CHAR(data_criacao, 'YYYY/MM') AS year_month,
  SUM(valor_total - valor_envio) AS total_revenue
FROM (
  SELECT DISTINCT pedido_numero,
    cpf_cliente, endereco_entrega_nome,
    valor_total, valor_envio, data_criacao
  FROM fato_vendas
  WHERE pedido_situacao NOT IN ('Pedido Cancelado','Pagamento devolvido')
) pedidos_unicos
GROUP BY year_month
ORDER BY year_month;
```
>**Insight:** Tracks sales evolution over time and identifies seasonal trends.
> ![total sold](https://github.com/luizzangelo/sql-analysis/blob/main/TOTAL_SOLD_PER_YEAR.png?raw=true)
---

## 📌 How to Contribute

Suggestions for new queries and analyses are welcome! Please open an issue or submit a pull request.

---

## 🚀 About

Built to consolidate key marketing and sales analyses using SQL in a real e-commerce context.

---


