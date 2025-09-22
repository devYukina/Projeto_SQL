# Projeto_SQL
# Projeto Banco de Dados – Online Retail II

Este repositório contém o modelo de banco de dados e consultas SQL do projeto de análise do dataset *Online Retail II*.

## 📂 Estrutura
- schema.sql → criação das tabelas (modelo estrela)
- queries.sql → consultas SQL usadas no relatório
- inserts.sql → (opcional) dados fictícios para teste

## 🚀 Como testar online
Você pode usar:
- [DB Fiddle](https://www.db-fiddle.com/) (PostgreSQL recomendado)
- [SQLite Online](https://sqliteonline.com/)

### Passos:
1. Copie o conteúdo de schema.sql no editor do site.
2. Adicione alguns registros (ou use inserts.sql).
3. Execute as queries de queries.sql.

## 📊 Consultas principais
- Receita mensal em 2011
- Top 10 produtos por receita
- Segmentação de clientes
- Países com maior taxa de cancelamento
- Evolução de vendas por categoria

## schema.sql
-- Criação das tabelas (modelo star schema)

CREATE TABLE dim_date (
    date_id SERIAL PRIMARY KEY,
    invoice_date DATE,
    year INT,
    month INT,
    day INT,
    weekday VARCHAR(10)
);

CREATE TABLE dim_country (
    country_id SERIAL PRIMARY KEY,
    country_name VARCHAR(100)
);

CREATE TABLE dim_customer (
    customer_id SERIAL PRIMARY KEY,
    customer_code VARCHAR(50),
    country_id INT REFERENCES dim_country(country_id)
);

CREATE TABLE dim_product (
    product_id SERIAL PRIMARY KEY,
    stock_code VARCHAR(50),
    description VARCHAR(255),
    category VARCHAR(100)
);

CREATE TABLE fact_sales (
    sale_id SERIAL PRIMARY KEY,
    invoice_no VARCHAR(50),
    date_id INT REFERENCES dim_date(date_id),
    customer_id INT REFERENCES dim_customer(customer_id),
    product_id INT REFERENCES dim_product(product_id),
    country_id INT REFERENCES dim_country(country_id),
    quantity INT,
    unit_price DECIMAL(10,2),
    line_total DECIMAL(12,2),
    is_canceled BOOLEAN
);

## queries.sql
-- Receita mensal em 2011
SELECT d.year, d.month, SUM(f.line_total) AS revenue
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
WHERE d.year = 2011 AND f.is_canceled = FALSE
GROUP BY d.year, d.month
ORDER BY d.year, d.month;

-- Top 10 produtos por receita
SELECT p.stock_code, p.description, SUM(f.line_total) AS revenue
FROM fact_sales f
JOIN dim_product p ON f.product_id = p.product_id
WHERE f.is_canceled = FALSE
GROUP BY p.stock_code, p.description
ORDER BY revenue DESC
LIMIT 10;

-- Segmentação de clientes
SELECT c.customer_code, COUNT(DISTINCT f.invoice_no) AS invoices,
       SUM(f.line_total) AS lifetime_value,
       CASE
         WHEN SUM(f.line_total) >= 2000 THEN 'VIP'
         WHEN SUM(f.line_total) >= 500 THEN 'Gold'
         ELSE 'Regular'
       END AS segment
FROM fact_sales f
JOIN dim_customer c ON f.customer_id = c.customer_id
WHERE f.is_canceled = FALSE
GROUP BY c.customer_code
ORDER BY lifetime_value DESC;

-- Países com maior taxa de cancelamento
SELECT co.country_name,
       SUM(CASE WHEN f.is_canceled = TRUE THEN f.quantity ELSE 0 END) AS canceled_units,
       SUM(f.quantity) AS sold_units,
       CASE WHEN SUM(f.quantity)=0 THEN 0
            ELSE SUM(CASE WHEN f.is_canceled = TRUE THEN f.quantity ELSE 0 END)*1.0/SUM(f.quantity)
       END AS cancel_rate
FROM fact_sales f
JOIN dim_country co ON f.country_id = co.country_id
GROUP BY co.country_name
ORDER BY cancel_rate DESC
LIMIT 10;

-- Evolução de vendas por categoria (últimos 12 meses)
SELECT d.year, d.month, p.category, SUM(f.line_total) AS revenue
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_product p ON f.product_id = p.product_id
WHERE (d.year = EXTRACT(YEAR FROM CURRENT_DATE) OR d.year = EXTRACT(YEAR FROM CURRENT_DATE)-1)
  AND f.is_canceled = FALSE
GROUP BY d.year, d.month, p.category
ORDER BY p.category, d.year, d.month;

## inserts.sql

-- Populando tabela dim_date
INSERT INTO dim_date (invoice_date, year, month, day, weekday)
VALUES 
('2011-01-10', 2011, 1, 10, 'Monday'),
('2011-02-15', 2011, 2, 15, 'Tuesday'),
('2011-03-20', 2011, 3, 20, 'Sunday'),
('2012-01-05', 2012, 1, 5, 'Thursday');

-- Populando tabela dim_country
INSERT INTO dim_country (country_name)
VALUES 
('United Kingdom'),
('Germany'),
('France');

-- Populando tabela dim_customer
INSERT INTO dim_customer (customer_code, country_id)
VALUES 
('C001', 1),
('C002', 2),
('C003', 3);

-- Populando tabela dim_product
INSERT INTO dim_product (stock_code, description, category)
VALUES 
('P001', 'Chair', 'Furniture'),
('P002', 'Table', 'Furniture'),
('P003', 'Lamp', 'Electronics');

-- Populando tabela fact_sales
INSERT INTO fact_sales (invoice_no, date_id, customer_id, product_id, country_id, quantity, unit_price, line_total, is_canceled)
VALUES
('INV001', 1, 1, 1, 1, 10, 50.00, 500.00, FALSE),
('INV002', 2, 2, 2, 2, 5, 200.00, 1000.00, FALSE),
('INV003', 2, 1, 3, 1, 2, 150.00, 300.00, TRUE),
('INV004', 3, 3, 1, 3, 3, 50.00, 150.00, FALSE),
('INV005', 4, 2, 3, 2, 1, 150.00, 150.00, FALSE);
