# Dimensional Modeling Guide

## Overview
Dimensional modeling is a data modeling technique optimized for data warehousing and business intelligence. It structures data in a way that's intuitive for business users and provides fast query performance.

## Core Concepts

### 1. Fact Tables
- **Definition**: Tables containing measurable, quantitative data about business events
- **Characteristics**:
  - Usually numeric and additive
  - High volume
  - Frequently updated
  - Contains foreign keys to dimension tables

### 2. Dimension Tables
- **Definition**: Tables containing descriptive attributes used to analyze the facts
- **Characteristics**:
  - Descriptive text attributes
  - Lower volume
  - Less frequent updates
  - Natural keys and business descriptions

## Types of Dimensional Models

### 1. Star Schema
- **Structure**: One fact table surrounded by dimension tables
- **Advantages**:
  - Simple to understand
  - Optimized query performance
  - Easy to modify
- **Example**:
```sql
-- Fact Table
CREATE TABLE fact_sales (
    sale_id BIGINT,
    date_key INT,
    product_key INT,
    customer_key INT,
    store_key INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    total_amount DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    FOREIGN KEY (product_key) REFERENCES dim_product(product_key),
    FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key),
    FOREIGN KEY (store_key) REFERENCES dim_store(store_key)
);

-- Dimension Tables
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    full_date DATE,
    day_of_week VARCHAR(9),
    day_of_month INT,
    month_name VARCHAR(9),
    quarter INT,
    year INT,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN
);

CREATE TABLE dim_product (
    product_key INT PRIMARY KEY,
    product_id VARCHAR(50),
    product_name VARCHAR(100),
    category VARCHAR(50),
    subcategory VARCHAR(50),
    brand VARCHAR(50),
    unit_cost DECIMAL(10,2),
    current_price DECIMAL(10,2)
);

CREATE TABLE dim_customer (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    phone VARCHAR(20),
    city VARCHAR(50),
    state VARCHAR(2),
    country VARCHAR(50),
    customer_segment VARCHAR(20)
);
```

### 2. Snowflake Schema
- **Structure**: Dimension tables are normalized into multiple related tables
- **Advantages**:
  - Reduced storage for large dimensions
  - Better data integrity
- **Example**:
```sql
-- Additional normalized tables for product dimension
CREATE TABLE dim_category (
    category_key INT PRIMARY KEY,
    category_name VARCHAR(50),
    department VARCHAR(50)
);

CREATE TABLE dim_subcategory (
    subcategory_key INT PRIMARY KEY,
    category_key INT,
    subcategory_name VARCHAR(50),
    FOREIGN KEY (category_key) REFERENCES dim_category(category_key)
);

CREATE TABLE dim_product_snowflake (
    product_key INT PRIMARY KEY,
    product_id VARCHAR(50),
    product_name VARCHAR(100),
    subcategory_key INT,
    brand VARCHAR(50),
    unit_cost DECIMAL(10,2),
    current_price DECIMAL(10,2),
    FOREIGN KEY (subcategory_key) REFERENCES dim_subcategory(subcategory_key)
);
```

## Common Dimension Types

### 1. Conformed Dimensions
- Used across multiple fact tables
- Ensures consistent analysis across different business processes
```sql
-- Conformed date dimension used across multiple fact tables
CREATE TABLE dim_date_conformed (
    date_key INT PRIMARY KEY,
    full_date DATE,
    fiscal_year INT,
    fiscal_quarter VARCHAR(2),
    calendar_year INT,
    calendar_quarter VARCHAR(2),
    month_number INT,
    month_name VARCHAR(9),
    day_of_week VARCHAR(9),
    is_weekend BOOLEAN,
    is_holiday BOOLEAN
);
```

### 2. Role-Playing Dimensions
- Same dimension used multiple times in a fact table with different roles
```sql
-- Example: Date dimension playing multiple roles
SELECT 
    od.order_date,
    sd.ship_date,
    dd.delivery_date,
    f.amount
FROM fact_orders f
JOIN dim_date od ON f.order_date_key = od.date_key
JOIN dim_date sd ON f.ship_date_key = sd.date_key
JOIN dim_date dd ON f.delivery_date_key = dd.date_key;
```

### 3. Slowly Changing Dimensions (SCD)
- **Type 1**: Overwrite old values
- **Type 2**: Keep history by adding new rows
- **Type 3**: Add new columns for current and previous values

```sql
-- Type 2 SCD Example
CREATE TABLE dim_product_scd2 (
    product_key INT PRIMARY KEY,
    product_id VARCHAR(50),
    product_name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10,2),
    valid_from DATE,
    valid_to DATE,
    is_current BOOLEAN,
    version INT
);

-- Insert history
INSERT INTO dim_product_scd2
SELECT 
    generate_surrogate_key(),  -- New surrogate key
    product_id,
    product_name,
    category,
    new_price,
    CURRENT_DATE,  -- valid_from
    NULL,         -- valid_to
    TRUE,         -- is_current
    version + 1   -- Increment version
FROM dim_product_scd2
WHERE product_id = '12345'
AND is_current = TRUE;

-- Update previous version
UPDATE dim_product_scd2
SET 
    valid_to = CURRENT_DATE - 1,
    is_current = FALSE
WHERE product_id = '12345'
AND is_current = TRUE;
```

## Implementation Best Practices

### 1. Fact Table Design
- Use surrogate keys
- Include all relevant foreign keys
- Keep grain consistent
- Include effective dates
- Consider partitioning for large tables

### 2. Dimension Table Design
- Include natural and surrogate keys
- Add descriptive attributes
- Consider hierarchies
- Plan for changes (SCD strategy)
- Include effective dates

### 3. Performance Optimization
- Create appropriate indexes
- Consider materialized views
- Implement partitioning
- Use columnar storage
- Optimize join conditions

## Example Queries

### 1. Basic Analysis
```sql
-- Sales by Product Category and Month
SELECT 
    dp.category,
    dd.month_name,
    dd.year,
    SUM(fs.quantity) as total_quantity,
    SUM(fs.total_amount) as total_sales
FROM fact_sales fs
JOIN dim_product dp ON fs.product_key = dp.product_key
JOIN dim_date dd ON fs.date_key = dd.date_key
GROUP BY dp.category, dd.month_name, dd.year
ORDER BY dd.year, dd.month_name;
```

### 2. Complex Analysis
```sql
-- Customer Segmentation Analysis
WITH customer_metrics AS (
    SELECT 
        dc.customer_segment,
        dp.category,
        COUNT(DISTINCT fs.customer_key) as unique_customers,
        SUM(fs.total_amount) as total_sales,
        SUM(fs.total_amount) / COUNT(DISTINCT fs.customer_key) as avg_customer_spend
    FROM fact_sales fs
    JOIN dim_customer dc ON fs.customer_key = dc.customer_key
    JOIN dim_product dp ON fs.product_key = dp.product_key
    WHERE fs.date_key >= (SELECT date_key FROM dim_date WHERE full_date >= CURRENT_DATE - INTERVAL '12 months')
    GROUP BY dc.customer_segment, dp.category
)
SELECT 
    customer_segment,
    category,
    unique_customers,
    total_sales,
    avg_customer_spend,
    RANK() OVER (PARTITION BY customer_segment ORDER BY total_sales DESC) as category_rank
FROM customer_metrics
ORDER BY customer_segment, total_sales DESC;
```

## Benefits of Dimensional Modeling

1. **Business Understanding**
   - Intuitive structure
   - Matches business processes
   - Easy to navigate

2. **Query Performance**
   - Optimized for aggregations
   - Efficient joins
   - Predictable query patterns

3. **Flexibility**
   - Easy to extend
   - Accommodates changes
   - Supports various analysis types

4. **Data Quality**
   - Consistent definitions
   - Clear relationships
   - Traceable lineage
