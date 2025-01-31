# Gold Layer Implementation Guide

## Overview
The Gold layer represents business-level aggregations, metrics, and data marts optimized for specific use cases. This layer serves business users, analysts, and data scientists with ready-to-use datasets.

## 1. Business Metrics Implementation

### Customer Analytics
```python
def create_customer_metrics(spark):
    """
    Calculate key customer metrics
    """
    return (spark.table("silver_transactions")
        .join(spark.table("silver_customers"), "customer_id")
        .groupBy("customer_id")
        .agg(
            # Purchase Behavior
            F.count("transaction_id").alias("total_purchases"),
            F.sum("amount").alias("total_spend"),
            F.avg("amount").alias("avg_order_value"),
            F.max("amount").alias("largest_purchase"),
            
            # Time-based Metrics
            F.datediff(
                F.max("transaction_date"),
                F.min("transaction_date")
            ).alias("customer_lifetime_days"),
            
            # Product Metrics
            F.countDistinct("product_id").alias("unique_products_bought"),
            
            # Derived Metrics
            F.expr("sum(amount) / count(transaction_id)").alias("average_transaction_value")
        ))

def create_customer_segments(spark):
    """
    Create customer segments based on RFM analysis
    """
    current_date = datetime.now()
    
    rfm = (spark.table("silver_transactions")
        .groupBy("customer_id")
        .agg(
            # Recency
            F.datediff(
                F.lit(current_date),
                F.max("transaction_date")
            ).alias("recency"),
            
            # Frequency
            F.count("transaction_id").alias("frequency"),
            
            # Monetary
            F.sum("amount").alias("monetary")
        )
        # Add RFM scores
        .withColumn("r_score", F.ntile(5).over(Window.orderBy("recency")))
        .withColumn("f_score", F.ntile(5).over(Window.orderBy("frequency")))
        .withColumn("m_score", F.ntile(5).over(Window.orderBy("monetary")))
        # Calculate final RFM score
        .withColumn("rfm_score", 
            F.concat(
                F.col("r_score"),
                F.col("f_score"),
                F.col("m_score")
            ))
        # Add segment labels
        .withColumn("customer_segment",
            F.when(F.col("rfm_score") >= 444, "Champions")
            .when(F.col("rfm_score") >= 434, "Loyal Customers")
            .when(F.col("rfm_score") >= 333, "Active Customers")
            .when(F.col("rfm_score") >= 222, "At Risk")
            .otherwise("Lost Customers")))
```

### Product Analytics
```python
def create_product_metrics(spark):
    """
    Calculate product performance metrics
    """
    return (spark.table("silver_transactions")
        .join(spark.table("silver_products"), "product_id")
        .groupBy("product_id", "product_name", "category")
        .agg(
            # Sales Metrics
            F.sum("quantity").alias("total_units_sold"),
            F.sum("amount").alias("total_revenue"),
            F.avg("amount").alias("avg_selling_price"),
            
            # Inventory Metrics
            F.first("stock_level").alias("current_stock"),
            F.first("reorder_point").alias("reorder_point"),
            
            # Performance Metrics
            F.sum(F.col("amount") - F.col("cost")).alias("total_profit"),
            F.avg(F.col("amount") - F.col("cost")).alias("avg_profit_per_unit")
        ))

def create_product_recommendations(spark):
    """
    Create product recommendations based on co-purchase patterns
    """
    # Get product pairs from transactions
    product_pairs = (spark.table("silver_transactions")
        .join(
            spark.table("silver_transactions").alias("t2"),
            (F.col("transaction_id") == F.col("t2.transaction_id")) &
            (F.col("product_id") < F.col("t2.product_id"))
        )
        .groupBy("product_id", "t2.product_id")
        .agg(F.count("*").alias("co_purchase_count"))
        .where("co_purchase_count >= 5"))  # Minimum co-purchase threshold
```

### Sales Analytics
```python
def create_sales_metrics(spark):
    """
    Calculate sales performance metrics
    """
    return (spark.table("silver_transactions")
        .withColumn("date", F.to_date("transaction_date"))
        .groupBy(
            F.col("date"),
            F.dayofweek("date").alias("day_of_week"),
            F.month("date").alias("month"),
            F.year("date").alias("year")
        )
        .agg(
            # Daily Sales Metrics
            F.sum("amount").alias("daily_revenue"),
            F.count("transaction_id").alias("transaction_count"),
            F.sum("quantity").alias("units_sold"),
            F.avg("amount").alias("avg_transaction_value"),
            
            # Calculated Metrics
            F.expr("sum(amount) / count(transaction_id)").alias("average_basket_size"),
            F.expr("count(distinct customer_id)").alias("unique_customers")
        ))

def create_sales_forecasts(spark):
    """
    Create sales forecasts using time series analysis
    """
    from pyspark.ml.feature import VectorAssembler
    from pyspark.ml.regression import LinearRegression
    
    # Prepare historical sales data
    sales_history = (spark.table("silver_transactions")
        .groupBy(F.to_date("transaction_date").alias("date"))
        .agg(F.sum("amount").alias("daily_sales"))
        # Add time-based features
        .withColumn("day_of_week", F.dayofweek("date"))
        .withColumn("month", F.month("date"))
        .withColumn("year", F.year("date"))
        # Create feature vector
        .select(
            "date",
            "daily_sales",
            "day_of_week",
            "month",
            "year"
        ))
    
    # Train forecasting model
    assembler = VectorAssembler(
        inputCols=["day_of_week", "month", "year"],
        outputCol="features"
    )
    
    training_data = assembler.transform(sales_history)
    lr = LinearRegression(featuresCol="features", labelCol="daily_sales")
    model = lr.fit(training_data)
```

## 2. Data Marts

### Marketing Data Mart
```python
def create_marketing_mart(spark):
    """
    Create marketing-focused data mart
    """
    return (spark.table("silver_transactions")
        .join(spark.table("silver_customers"), "customer_id")
        .join(spark.table("customer_segments"), "customer_id")
        .groupBy(
            "customer_id",
            "customer_segment",
            "email",
            "location",
            "age_group"
        )
        .agg(
            # Customer Value Metrics
            F.sum("amount").alias("total_spend"),
            F.avg("amount").alias("avg_order_value"),
            F.count("transaction_id").alias("purchase_frequency"),
            
            # Product Preferences
            F.collect_set("category").alias("preferred_categories"),
            F.first("last_purchase_date").alias("last_purchase"),
            
            # Campaign Metrics
            F.sum("campaign_clicks").alias("total_campaign_interactions"),
            F.avg("email_open_rate").alias("avg_email_engagement")
        ))
```

### Finance Data Mart
```python
def create_finance_mart(spark):
    """
    Create finance-focused data mart
    """
    return (spark.table("silver_transactions")
        .join(spark.table("silver_products"), "product_id")
        .groupBy(
            F.to_date("transaction_date").alias("date"),
            "product_category",
            "payment_method"
        )
        .agg(
            # Revenue Metrics
            F.sum("amount").alias("gross_revenue"),
            F.sum("tax_amount").alias("total_tax"),
            F.sum("discount_amount").alias("total_discounts"),
            F.sum(F.col("amount") - F.col("cost")).alias("gross_profit"),
            
            # Transaction Metrics
            F.count("transaction_id").alias("transaction_count"),
            F.avg("amount").alias("avg_transaction_value"),
            
            # Payment Analytics
            F.sum(F.when(F.col("payment_status") == "refunded", F.col("amount"))
                 .otherwise(0)).alias("total_refunds"),
            F.sum(F.when(F.col("payment_method") == "credit_card", F.col("amount"))
                 .otherwise(0)).alias("credit_card_revenue")
        ))
```

### Supply Chain Data Mart
```python
def create_supply_chain_mart(spark):
    """
    Create supply chain-focused data mart
    """
    return (spark.table("silver_inventory")
        .join(spark.table("silver_products"), "product_id")
        .join(spark.table("silver_suppliers"), "supplier_id")
        .groupBy(
            "product_id",
            "supplier_id",
            "warehouse_id"
        )
        .agg(
            # Inventory Metrics
            F.first("current_stock").alias("stock_level"),
            F.first("reorder_point").alias("reorder_point"),
            F.first("lead_time_days").alias("supplier_lead_time"),
            
            # Order Metrics
            F.sum("order_quantity").alias("total_ordered"),
            F.avg("order_quantity").alias("avg_order_size"),
            
            # Quality Metrics
            F.avg("defect_rate").alias("avg_defect_rate"),
            F.sum("returned_quantity").alias("total_returns"),
            
            # Cost Metrics
            F.avg("unit_cost").alias("avg_unit_cost"),
            F.sum("holding_cost").alias("total_holding_cost")
        ))
```

## 3. Performance Optimization

### Partitioning Strategy
```python
def optimize_gold_tables(spark):
    """
    Apply optimization strategies to gold tables
    """
    # Partition sales metrics by date
    spark.sql("""
    ALTER TABLE gold_sales_metrics
    SET TBLPROPERTIES (
        delta.autoOptimize.optimizeWrite = true,
        delta.autoOptimize.autoCompact = true
    )
    PARTITIONED BY (year, month)
    """)
    
    # Z-order customer metrics by high-cardinality columns
    spark.sql("""
    OPTIMIZE gold_customer_metrics
    ZORDER BY (customer_id, location)
    """)
```

### Materialized Views
```python
def create_materialized_views(spark):
    """
    Create materialized views for common query patterns
    """
    spark.sql("""
    CREATE MATERIALIZED VIEW gold_daily_sales_mv
    REFRESH EVERY 24 HOURS
    AS
    SELECT 
        date,
        SUM(amount) as daily_revenue,
        COUNT(DISTINCT customer_id) as unique_customers,
        AVG(amount) as avg_transaction_value
    FROM silver_transactions
    GROUP BY date
    """)
```

## 4. Data Quality Checks

```python
def validate_gold_metrics(spark):
    """
    Implement data quality checks for gold layer
    """
    def check_metric_consistency():
        # Verify customer metrics
        customer_metrics = spark.table("gold_customer_metrics")
        validation_results = []
        
        # Check for negative values
        negative_metrics = customer_metrics.filter(
            (F.col("total_spend") < 0) |
            (F.col("avg_order_value") < 0)
        ).count()
        
        # Check for metric relationships
        invalid_averages = customer_metrics.filter(
            F.col("total_spend") < F.col("avg_order_value")
        ).count()
        
        # Check for completeness
        null_metrics = customer_metrics.filter(
            F.col("customer_segment").isNull() |
            F.col("total_spend").isNull()
        ).count()
        
        return {
            "negative_metrics": negative_metrics,
            "invalid_averages": invalid_averages,
            "null_metrics": null_metrics
        }
```

## 5. Usage Examples

### Business Intelligence Queries
```sql
-- Customer Lifetime Value Analysis
SELECT 
    customer_segment,
    COUNT(*) as customer_count,
    AVG(total_spend) as avg_customer_value,
    MAX(total_spend) as highest_value,
    MIN(total_spend) as lowest_value
FROM gold_customer_metrics
GROUP BY customer_segment
ORDER BY avg_customer_value DESC;

-- Product Performance Dashboard
SELECT 
    category,
    SUM(total_revenue) as category_revenue,
    SUM(total_profit) as category_profit,
    AVG(profit_margin) as avg_margin,
    SUM(total_units_sold) as units_sold
FROM gold_product_metrics
GROUP BY category
ORDER BY category_revenue DESC;

-- Sales Trend Analysis
SELECT 
    date,
    daily_revenue,
    transaction_count,
    unique_customers,
    LAG(daily_revenue) OVER (ORDER BY date) as prev_day_revenue,
    (daily_revenue - LAG(daily_revenue) OVER (ORDER BY date)) / 
        LAG(daily_revenue) OVER (ORDER BY date) * 100 as revenue_growth_pct
FROM gold_sales_metrics
WHERE date >= CURRENT_DATE - INTERVAL 30 DAYS;
```

### API Integration Example
```python
from fastapi import FastAPI, HTTPException
from typing import List, Optional

app = FastAPI()

@app.get("/api/v1/customer-metrics")
async def get_customer_metrics(
    segment: Optional[str] = None,
    min_spend: Optional[float] = None
):
    """API endpoint for customer metrics"""
    query = spark.table("gold_customer_metrics")
    
    if segment:
        query = query.filter(F.col("customer_segment") == segment)
    if min_spend:
        query = query.filter(F.col("total_spend") >= min_spend)
        
    return query.toPandas().to_dict(orient="records")

@app.get("/api/v1/sales-forecast")
async def get_sales_forecast(days: int = 30):
    """API endpoint for sales forecasts"""
    forecast = spark.table("gold_sales_forecasts")
    return forecast.filter(
        F.col("forecast_date").between(
            F.current_date(),
            F.current_date() + days
        )
    ).toPandas().to_dict(orient="records")
```

This implementation provides:
1. Comprehensive business metrics
2. Optimized data marts for different departments
3. Performance optimization strategies
4. Data quality validation
5. Ready-to-use API endpoints

Would you like me to:
1. Add more specific metrics or KPIs?
2. Implement additional data marts?
3. Add more complex analytical queries?
4. Create visualization examples?
