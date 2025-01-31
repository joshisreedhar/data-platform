# Detailed Layer Actions in Data Platform Architecture

## 1. Data Ingestion Layer

### Batch Ingestion Examples
```sql
-- Example 1: Loading daily sales data from CSV
COPY sales_bronze
FROM 's3://raw-data/sales/2024-01-30/*.csv'
FORMAT CSV
CREDENTIALS 'aws_iam_role=arn:aws:iam::123456789012:role/RedshiftLoadRole';

-- Example 2: Database snapshot via SQL
INSERT INTO customer_bronze
SELECT *
FROM source_system.customers
WHERE last_updated_date = CURRENT_DATE;
```

### Stream Ingestion Examples
```python
# Example 1: Kafka consumer for real-time orders
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'orders_topic',
    bootstrap_servers=['kafka:9092'],
    group_id='orders_consumer_group'
)

for message in consumer:
    save_to_bronze_layer(message.value)

# Example 2: CDC capture using Debezium
{
    "name": "inventory-connector",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.hostname": "mysql",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "dbz",
        "database.server.id": "184054",
        "topic.prefix": "dbserver1",
        "table.include.list": "inventory.customers"
    }
}
```

## 2. Bronze Layer (Raw)

### Data Landing Examples
```python
# Example 1: Raw JSON storage
def store_raw_event(event_data):
    bronze_path = f"s3://bronze/events/{date}/{uuid}.json"
    write_json(event_data, bronze_path)

# Example 2: Preserve source format
def store_xml_data(xml_content):
    bronze_path = f"s3://bronze/source_xml/{date}/{filename}"
    save_with_metadata(xml_content, bronze_path, {
        'source_system': 'SAP',
        'ingestion_time': datetime.now(),
        'format': 'xml'
    })
```

### Metadata Enrichment
```python
# Example: Adding technical metadata
def enrich_bronze_metadata(file_path):
    metadata = {
        'source_system': extract_source(file_path),
        'ingestion_timestamp': datetime.now(),
        'file_size': get_file_size(file_path),
        'record_count': count_records(file_path),
        'schema_version': '1.0',
        'data_format': detect_format(file_path)
    }
    attach_metadata(file_path, metadata)
```

## 3. Silver Layer (Standardized)

### Data Quality Checks
```python
# Example 1: Data validation rules
def validate_customer_data(df):
    rules = [
        df['email'].str.contains('@').all(),
        df['phone'].str.match(r'^\+?1?\d{9,15}$').all(),
        df['age'].between(0, 120).all()
    ]
    return all(rules)

# Example 2: Duplicate detection
def remove_duplicates(df):
    return df.drop_duplicates(
        subset=['email', 'phone'],
        keep='last'
    )
```

### Data Standardization
```sql
-- Example 1: Address standardization
CREATE OR REPLACE VIEW silver.standardized_addresses AS
SELECT
    customer_id,
    UPPER(TRIM(address_line1)) as address_line1,
    UPPER(TRIM(city)) as city,
    UPPER(TRIM(state)) as state,
    REGEXP_REPLACE(zip, '[^0-9]', '') as zip_code
FROM bronze.customer_addresses;

-- Example 2: Currency standardization
CREATE OR REPLACE VIEW silver.standardized_transactions AS
SELECT
    transaction_id,
    amount * exchange_rate as amount_usd,
    'USD' as currency
FROM bronze.transactions t
JOIN exchange_rates e ON t.currency = e.source_currency
AND DATE(t.transaction_date) = e.rate_date;
```

## 4. Gold Layer (Business)

### Aggregations
```sql
-- Example 1: Daily sales metrics
CREATE OR REPLACE VIEW gold.daily_sales_metrics AS
SELECT
    date_trunc('day', transaction_date) as sale_date,
    product_category,
    COUNT(*) as total_transactions,
    SUM(amount_usd) as total_revenue,
    AVG(amount_usd) as avg_transaction_value,
    COUNT(DISTINCT customer_id) as unique_customers
FROM silver.standardized_transactions
GROUP BY 1, 2;

-- Example 2: Customer segmentation
CREATE OR REPLACE VIEW gold.customer_segments AS
SELECT
    customer_id,
    CASE 
        WHEN total_spend > 10000 THEN 'Premium'
        WHEN total_spend > 5000 THEN 'Gold'
        WHEN total_spend > 1000 THEN 'Silver'
        ELSE 'Bronze'
    END as customer_segment,
    total_spend,
    avg_order_value,
    frequency
FROM (
    SELECT
        customer_id,
        SUM(amount_usd) as total_spend,
        AVG(amount_usd) as avg_order_value,
        COUNT(*) as frequency
    FROM silver.standardized_transactions
    GROUP BY 1
);
```

### Business Metrics
```python
# Example 1: Customer Lifetime Value calculation
def calculate_customer_ltv():
    sql = """
    CREATE OR REPLACE VIEW gold.customer_ltv AS
    WITH customer_metrics AS (
        SELECT
            customer_id,
            SUM(amount_usd) as total_revenue,
            COUNT(DISTINCT order_id) as total_orders,
            MIN(transaction_date) as first_purchase,
            MAX(transaction_date) as last_purchase,
            COUNT(DISTINCT DATE_TRUNC('month', transaction_date)) as active_months
        FROM silver.standardized_transactions
        GROUP BY 1
    )
    SELECT
        customer_id,
        total_revenue,
        total_revenue / NULLIF(active_months, 0) as monthly_revenue,
        total_revenue * (
            1 + GREATEST(0, 1 - EXTRACT(MONTH FROM AGE(CURRENT_DATE, last_purchase)) / 12.0)
        ) as predicted_ltv
    FROM customer_metrics
    """
    execute_sql(sql)

# Example 2: Product performance metrics
def calculate_product_metrics():
    sql = """
    CREATE OR REPLACE VIEW gold.product_performance AS
    SELECT
        p.product_id,
        p.product_name,
        p.category,
        COUNT(DISTINCT t.order_id) as total_orders,
        SUM(t.quantity) as units_sold,
        SUM(t.amount_usd) as total_revenue,
        SUM(t.amount_usd) / NULLIF(SUM(t.quantity), 0) as avg_unit_price,
        COUNT(DISTINCT t.customer_id) as unique_customers,
        SUM(t.amount_usd) / NULLIF(COUNT(DISTINCT t.customer_id), 0) as revenue_per_customer
    FROM silver.standardized_transactions t
    JOIN silver.products p ON t.product_id = p.product_id
    GROUP BY 1, 2, 3
    """
    execute_sql(sql)
```

## 5. Serving Layer

### API Examples
```python
# Example 1: REST API for customer data
from fastapi import FastAPI
app = FastAPI()

@app.get("/api/v1/customer/{customer_id}")
async def get_customer_profile(customer_id: int):
    return {
        "profile": await get_from_gold("customer_profile", customer_id),
        "segments": await get_from_gold("customer_segments", customer_id),
        "ltv": await get_from_gold("customer_ltv", customer_id)
    }

# Example 2: GraphQL API for product analytics
schema = """
type Product {
    id: ID!
    name: String!
    category: String!
    metrics: ProductMetrics!
}

type ProductMetrics {
    unitsSold: Int!
    totalRevenue: Float!
    avgUnitPrice: Float!
    uniqueCustomers: Int!
}

type Query {
    product(id: ID!): Product
    productsByCategory(category: String!): [Product]!
}
"""
```

### Real-time Analytics
```python
# Example 1: Real-time dashboard updates
def stream_metrics_to_dashboard():
    consumer = KafkaConsumer('processed_transactions')
    for event in consumer:
        metrics = calculate_real_time_metrics(event)
        websocket_broadcast('dashboard', metrics)

# Example 2: Anomaly detection
def detect_anomalies():
    def process_transaction(transaction):
        prediction = model.predict(transaction.features)
        if prediction.is_anomaly:
            alert_security_team(transaction)
        return prediction

    with StreamingContext('transactions') as stream:
        stream.map(process_transaction)
```

## 6. Monitoring Layer

### Performance Monitoring
```python
# Example 1: Pipeline monitoring
def monitor_pipeline_performance():
    metrics = {
        'ingestion_lag': measure_lag('raw_events'),
        'processing_time': measure_processing('silver_layer'),
        'data_freshness': check_data_freshness('gold_layer'),
        'error_rate': calculate_error_rate('all_pipelines')
    }
    push_to_prometheus(metrics)

# Example 2: Query performance
def track_query_performance():
    sql = """
    SELECT
        query_id,
        query_text,
        execution_time,
        rows_processed,
        cpu_time,
        memory_used
    FROM system.query_metrics
    WHERE start_time > DATEADD(hour, -1, CURRENT_TIMESTAMP)
    """
    metrics = execute_sql(sql)
    alert_if_slow(metrics)
```

### Data Quality Monitoring
```python
# Example 1: Data quality metrics
def monitor_data_quality():
    checks = {
        'completeness': check_null_rates(),
        'accuracy': validate_business_rules(),
        'consistency': check_cross_system_consistency(),
        'timeliness': measure_data_lag()
    }
    report_quality_metrics(checks)

# Example 2: Schema evolution
def track_schema_changes():
    current_schema = extract_current_schema('silver.customers')
    if detect_schema_changes(current_schema):
        notify_data_team('Schema change detected')
        update_data_catalog(current_schema)
```

These examples demonstrate concrete implementations of various actions performed at each layer of the data platform architecture. Each example includes:
- Actual code snippets
- Real-world use cases
- Common patterns and practices
- Integration points

Would you like me to:
1. Elaborate on any specific layer or example?
2. Add more examples for a particular type of processing?
3. Show how these components interact in a specific scenario?
4. Provide more details about error handling and edge cases?
