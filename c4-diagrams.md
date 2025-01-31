# C4 Model Diagrams for Data Platform Architecture

## Level 1: System Context Diagram

```mermaid
C4Context
title System Context diagram for Data Platform

Person(business_user, "Business User", "Consumes data for analysis and reporting")
Person(data_engineer, "Data Engineer", "Manages and maintains the data platform")
Person(data_scientist, "Data Scientist", "Performs advanced analytics")

System(data_platform, "Data Platform", "Enterprise Data Platform using Medallion Architecture")

System_Ext(source_systems, "Source Systems", "External data sources (CRM, ERP, etc.)")
System_Ext(bi_tools, "BI Tools", "Business Intelligence and Visualization Tools")
System_Ext(ml_platforms, "ML Platforms", "Machine Learning Platforms")

Rel(source_systems, data_platform, "Provides raw data")
Rel(data_platform, bi_tools, "Provides processed data")
Rel(data_platform, ml_platforms, "Provides feature-ready data")

Rel(business_user, bi_tools, "Uses")
Rel(data_scientist, ml_platforms, "Uses")
Rel(data_engineer, data_platform, "Manages")

UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Level 2: Container Diagram

```mermaid
C4Container
title Container diagram for Data Platform

Person(business_user, "Business User", "Consumes data for analysis and reporting")
Person(data_engineer, "Data Engineer", "Manages and maintains the data platform")

System_Boundary(c1, "Data Platform") {
    Container(ingestion_layer, "Data Ingestion Layer", "Apache Spark, Delta Lake", "Ingests and validates raw data")
    Container(bronze_layer, "Bronze Layer", "Delta Lake", "Raw data storage with metadata")
    Container(silver_layer, "Silver Layer", "Delta Lake", "Cleaned and standardized data")
    Container(gold_layer, "Gold Layer", "Delta Lake", "Business-level aggregates")
    
    Container(metadata_store, "Metadata Store", "Delta Lake", "Stores technical and business metadata")
    Container(data_catalog, "Data Catalog", "REST API", "Data discovery and governance")
    Container(orchestrator, "Orchestration Service", "Apache Airflow", "Manages data pipelines")
    Container(monitoring, "Monitoring Service", "Prometheus, Grafana", "Platform monitoring and alerts")
}

System_Ext(source_systems, "Source Systems", "External data sources")
System_Ext(bi_tools, "BI Tools", "Business Intelligence Tools")

Rel(source_systems, ingestion_layer, "Sends data", "Various protocols")
Rel(ingestion_layer, bronze_layer, "Stores raw data", "Delta Lake")
Rel(bronze_layer, silver_layer, "Processes data", "Apache Spark")
Rel(silver_layer, gold_layer, "Creates metrics", "Apache Spark")

Rel(data_engineer, orchestrator, "Manages pipelines", "Web UI")
Rel(business_user, bi_tools, "Analyzes data", "SQL/Web UI")
Rel(bi_tools, gold_layer, "Reads data", "SQL")

Rel(ingestion_layer, metadata_store, "Records metadata", "REST API")
Rel(data_catalog, metadata_store, "Reads metadata", "REST API")
Rel(orchestrator, monitoring, "Sends metrics", "REST API")

UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Level 3: Component Diagram

```mermaid
C4Component
title Component diagram for Data Platform

Container_Boundary(bronze, "Bronze Layer") {
    Component(ingest_service, "Ingestion Service", "PySpark", "Handles data ingestion")
    Component(metadata_enrichment, "Metadata Enrichment", "PySpark", "Adds technical metadata")
    Component(schema_validation, "Schema Validation", "PySpark", "Validates data schema")
    Component(bronze_storage, "Bronze Storage", "Delta Lake", "Stores raw data")
}

Container_Boundary(silver, "Silver Layer") {
    Component(data_cleaning, "Data Cleaning Service", "PySpark", "Cleanses data")
    Component(data_quality, "Data Quality Service", "Great Expectations", "Validates data quality")
    Component(standardization, "Standardization Service", "PySpark", "Standardizes data")
    Component(silver_storage, "Silver Storage", "Delta Lake", "Stores cleaned data")
}

Container_Boundary(gold, "Gold Layer") {
    Component(metrics_engine, "Metrics Engine", "PySpark", "Calculates business metrics")
    Component(dim_modeling, "Dimensional Modeling", "PySpark", "Creates dimensional models")
    Component(data_marts, "Data Marts", "Delta Lake", "Department-specific views")
    Component(gold_storage, "Gold Storage", "Delta Lake", "Stores business metrics")
}

Container_Boundary(platform, "Platform Services") {
    Component(pipeline_scheduler, "Pipeline Scheduler", "Airflow", "Schedules jobs")
    Component(resource_manager, "Resource Manager", "Kubernetes", "Manages resources")
    Component(monitoring_service, "Monitoring Service", "Prometheus", "Monitors platform")
    Component(catalog_service, "Catalog Service", "REST API", "Data discovery")
}

Rel(ingest_service, metadata_enrichment, "Sends data")
Rel(metadata_enrichment, schema_validation, "Validates")
Rel(schema_validation, bronze_storage, "Stores")

Rel(bronze_storage, data_cleaning, "Reads")
Rel(data_cleaning, data_quality, "Validates")
Rel(data_quality, standardization, "Standardizes")
Rel(standardization, silver_storage, "Stores")

Rel(silver_storage, metrics_engine, "Reads")
Rel(metrics_engine, dim_modeling, "Models")
Rel(dim_modeling, data_marts, "Creates")
Rel(data_marts, gold_storage, "Stores")

Rel(pipeline_scheduler, resource_manager, "Allocates")
Rel(monitoring_service, catalog_service, "Updates")

UpdateLayoutConfig($c4ShapeInRow="4", $c4BoundaryInRow="2")
```

## Architecture Highlights

### Context Level
- Shows the high-level interaction between users, external systems, and the data platform
- Emphasizes the platform's role in serving different user personas
- Illustrates data flow from source systems to consumption points

### Container Level
- Details the main components of the medallion architecture
- Shows the relationship between different layers
- Includes supporting services like metadata, orchestration, and monitoring
- Demonstrates data flow and transformation stages

### Component Level
- Breaks down each layer into specific functional components
- Shows detailed data flow within each layer
- Illustrates how platform services support the data processing
- Demonstrates integration points between components

### Key Features Highlighted
1. **Data Processing Layers**
   - Bronze: Raw data ingestion and validation
   - Silver: Data cleaning and standardization
   - Gold: Business metrics and dimensional models

2. **Platform Services**
   - Metadata management
   - Data catalog
   - Orchestration
   - Monitoring and alerting

3. **Integration Points**
   - Source system connections
   - BI tool integration
   - ML platform support
   - Monitoring interfaces

4. **Data Flow**
   - Clear progression through layers
   - Transformation steps
   - Quality checks
   - Metric generation

Would you like me to:
1. Add more detail to any specific layer?
2. Include additional components?
3. Show specific data flows?
4. Add deployment views?
