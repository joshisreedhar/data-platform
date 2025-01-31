# Bronze Layer Metadata Enrichment Examples

## 1. Technical Metadata

### Source Information
```python
from pyspark.sql.functions import (
    input_file_name, current_timestamp, lit, 
    regexp_extract, to_timestamp, size
)

def enrich_source_metadata(df):
    return df.withColumns({
        # Source file information
        "source_file_name": input_file_name(),
        "source_file_path": regexp_extract(input_file_name(), "^(.*)/[^/]*$", 1),
        "source_system": lit("salesforce"),  # or extract from path/config
        
        # Ingestion metadata
        "ingestion_timestamp": current_timestamp(),
        "ingestion_job_id": lit("job_123"),
        "ingestion_batch_id": lit("batch_456"),
        
        # File format metadata
        "source_format": lit("csv"),  # or json, parquet, etc.
        "source_version": lit("1.0")
    })
```

### Schema Information
```python
def enrich_schema_metadata(df):
    # Capture schema information
    schema_info = {
        "column_count": len(df.columns),
        "schema_version": "1.0",
        "schema_hash": hash(str(df.schema)),
        "nullable_columns": [f.name for f in df.schema.fields if f.nullable],
        "data_types": {f.name: str(f.dataType) for f in df.schema.fields}
    }
    
    return df.withColumns({
        "schema_version": lit(schema_info["schema_version"]),
        "schema_hash": lit(schema_info["schema_hash"]),
        "_metadata": to_json(struct([
            lit(schema_info).alias("schema_info")
        ]))
    })
```

## 2. Data Quality Metadata

### Record-Level Statistics
```python
def enrich_quality_metadata(df):
    return df.withColumns({
        # Record counts
        "record_count": count("*"),
        "null_count": sum(when(col("*").isNull(), 1).otherwise(0)),
        
        # Basic validation flags
        "has_duplicates": lit(df.count() > df.dropDuplicates().count()),
        "has_nulls": lit(df.dropna().count() < df.count()),
        
        # Data ranges
        "min_timestamp": min("timestamp_column"),
        "max_timestamp": max("timestamp_column"),
        
        # Quality score (example)
        "quality_score": compute_quality_score(df)
    })
```

### Data Profiling Metadata
```python
def enrich_profiling_metadata(df):
    profile_metrics = {
        # Column statistics
        "numeric_columns": get_numeric_stats(df),
        "string_columns": get_string_stats(df),
        "timestamp_columns": get_timestamp_stats(df),
        
        # Pattern detection
        "detected_patterns": detect_patterns(df),
        
        # Anomaly scores
        "anomaly_scores": compute_anomaly_scores(df)
    }
    
    return df.withColumn("_profiling", to_json(struct([
        lit(profile_metrics).alias("profile_metrics")
    ])))
```

## 3. Business Metadata

### Classification Tags
```python
def enrich_business_metadata(df):
    return df.withColumns({
        # Data classification
        "data_classification": lit("PII"),  # or PHI, CONFIDENTIAL, etc.
        "sensitivity_level": lit("HIGH"),
        
        # Business domain
        "business_domain": lit("sales"),
        "business_unit": lit("north_america"),
        
        # Retention policy
        "retention_period": lit("7_years"),
        "deletion_date": add_months(current_date(), 84),
        
        # Compliance
        "compliance_tags": array(lit("GDPR"), lit("CCPA")),
        "requires_encryption": lit(True)
    })
```

### Data Governance Metadata
```python
def enrich_governance_metadata(df):
    governance_info = {
        # Ownership
        "data_owner": "sales_team",
        "data_steward": "john.doe@company.com",
        
        # Access control
        "access_level": "restricted",
        "authorized_groups": ["sales_analysts", "data_scientists"],
        
        # Audit trail
        "last_reviewed_by": "jane.smith@company.com",
        "last_reviewed_date": current_timestamp(),
        
        # Lineage
        "upstream_systems": ["salesforce", "marketo"],
        "downstream_systems": ["data_warehouse", "reporting"]
    }
    
    return df.withColumn("_governance", to_json(struct([
        lit(governance_info).alias("governance_info")
    ])))
```

## 4. Operational Metadata

### Processing Information
```python
def enrich_operational_metadata(df):
    return df.withColumns({
        # Processing metrics
        "processing_start_time": current_timestamp(),
        "processing_environment": lit("production"),
        "processing_cluster": lit("cluster-001"),
        
        # Performance metrics
        "batch_size": lit(df.count()),
        "processing_time_ms": lit(get_processing_time()),
        
        # Resource utilization
        "memory_used": lit(get_memory_usage()),
        "cpu_used": lit(get_cpu_usage())
    })
```

### Pipeline Metadata
```python
def enrich_pipeline_metadata(df):
    pipeline_info = {
        # Pipeline details
        "pipeline_id": "pipeline_123",
        "pipeline_version": "2.0",
        "pipeline_run_id": "run_456",
        
        # Dependencies
        "upstream_jobs": ["extract_salesforce", "extract_marketo"],
        "downstream_jobs": ["silver_transform", "quality_check"],
        
        # Schedule
        "schedule_interval": "hourly",
        "next_run_time": add_hours(current_timestamp(), 1)
    }
    
    return df.withColumn("_pipeline", to_json(struct([
        lit(pipeline_info).alias("pipeline_info")
    ])))
```

## 5. Integration Example

```python
def apply_bronze_enrichment(df, config):
    """
    Apply all metadata enrichment in one pass
    """
    enriched_df = (df
        .transform(enrich_source_metadata)
        .transform(enrich_schema_metadata)
        .transform(enrich_quality_metadata)
        .transform(enrich_profiling_metadata)
        .transform(enrich_business_metadata)
        .transform(enrich_governance_metadata)
        .transform(enrich_operational_metadata)
        .transform(enrich_pipeline_metadata)
    )
    
    # Write enriched data with metadata
    (enriched_df.write
        .format("delta")
        .mode("append")
        .option("userMetadata", get_table_metadata())
        .save(config.bronze_path))
    
    return enriched_df
```

## Benefits of Rich Metadata

1. **Data Governance**
   - Complete audit trail
   - Compliance tracking
   - Access control

2. **Data Quality**
   - Quality monitoring
   - Issue detection
   - Trend analysis

3. **Operations**
   - Performance optimization
   - Resource planning
   - Cost allocation

4. **Analytics**
   - Data discovery
   - Impact analysis
   - Usage patterns

This metadata enrichment provides a solid foundation for:
- Data catalog integration
- Automated data quality
- Compliance reporting
- Performance optimization
- Cost analysis
