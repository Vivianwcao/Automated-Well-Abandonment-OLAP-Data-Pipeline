
# Automated Well Abandonment Data Pipeline




## A Serverless Data Lakehouse Architecture for OLAP

**Tech Stack**
- **Languages & Libraries** Python, Pandas, OpenPyXL, PyArrow
- **AWS Infrastructure** AWS AppFlow, S3, EventBridge, SQS + DLQ, AWS Lambda
- **Analytics & AI Layer & BI:** Claude Haiku (AWS Bedrock), AWS Glue, Athena, QuickSight(SPICE)

## Project Scope & Business Value
- **Historical Data Ingestion:** Processed 750+ legacy .xlsm/.xlsx field reports spanning 5 years of historical oil and gas operations in Alberta.
- **Analytics Readiness:** Transformed disjointed, hand-entered spreadsheet data into 5 normalized relational tables, enabling long-term trend analysis.
- **Cost & Performance Optimization:** Built an entire OLAP pipeline using a serverless architecture to eliminate idle infrastructure costs, utilizing in-memory caching for downstream BI reporting.
- 
## Architecture & Medallion Data Flow
The pipeline implements an event-driven **Medallion Architecture** to ingest, parse, and enrich data step-by-step:

- **Bronze (S3 raw/):** Raw, multi-tabbed Excel workbooks transferred from Google Drive via AWS AppFlow.
- **Silver (S3 extracted/):** Intermediate structured JSON containing raw data pulled from cell-anchored parsing.
- **Gold (S3 clean/):** Schema-enforced, LLM-enriched Parquet datasets partitioned into 5 base data tables.

## Technical Challenges
- **The Challenge:** Transferring 750 historical files simultaneously via AppFlow creates a massive burst of objects landing in S3. If S3 triggered the processing Lambdas directly, it would instantly flood the system, spike concurrent invocations, and trigger AWS account-level throttling.

- **The Solution:** Decoupled the ingestion layer using **Amazon EventBridge and SQS Queues**. EventBridge monitors the S3 bucket prefixes and routes object creation events into SQS. SQS acts as a buffer, metering message delivery to downstream Lambdas at a controlled, predictable execution rate.
  
## Layout Drift in Hand-Entered Excel Forms (5-Year Span)
- **The Challenge:** Field supervisors filled out these sheets by hand over a 5-year period. The spreadsheets were designed to be visually appealing to human readers rather than machine-readable (utilizing merged cells, inconsistent row heights, and erratic label alignments). Standard coordinate-based parsing via pd.read_excel() broke immediately due to historical layout variations.

- **The Solution:** Built an anchor-based cell detection mechanism using openpyxl. The script scans sheets for known text strings (labels) and dynamically extracts data from adjacent or relative offsets rather than hardcoded cell coordinates. This stopped layout changes from causing silent parsing failures.

## Multi-Tab Relational Mapping
- **The Challenge:** Each Excel workbook contains multiple tabs. These tabs do not map 1:1 into neat database tables; instead, data must be extracted across several tabs and joined logically to construct a complete, accurate understanding of the operational event.

- **The Solution:** Prompt-engineered Claude Haiku on AWS Bedrock to handle domain-specific text evaluation. The model ingests daily free-text summaries and returns strongly typed, structured JSON classifying well events, separating operational parameters, and converting units automatically.
## Downstream Schema Drift in Parquet / Glue

- **The Challenge:** Independent parsing of 750 files causes implicit data-type mismatches (e.g., an empty column inferred as a float in one file but a string in another), causing PyArrow and Glue Crawlers to crash during aggregation.

- **The Solution:** Enforced a centralized Master Schema dictionary at the write layer of Lambda 2. Prior to saving as Parquet, all DataFrames are forcibly coerced: missing columns are injected as nulls, data types are strict-cast, and booleans are mapped cleanly to pandas' nullable Boolean type.
