
# Automated Well Abandonment Data Pipeline




## A Serverless Data Lakehouse Architecture for OLAP

**Tech Stack & Environment**
- **Languages & Frameworks:** Python, Pandas, OpenPyXL, PyArrow
- **Serverless Data Infra:** AWS AppFlow, S3, EventBridge, SQS + DLQ, AWS Lambda
- **Analytics & AI Layer:** Claude Haiku (AWS Bedrock), AWS Glue, Athena, QuickSight
## Business Impact & Core Design
- **Unlocked Dark Data:** Converted 750+ manual, isolated legacy multi-tab XLSX workbooks into an active, queryable data asset.
- **Zero Infrastructure Overhead:** Designed a 100% serverless OLAP architecture. No idle compute, no persistent clusters, costing a fraction of a traditional Redshift or Snowflake deployment.
- **Executive Insights:** Enabled three automated QuickSight dashboards tracking well scorecards, supplier costs, and regulatory compliance.
## Architecture & Medallion Data Flow

The pipeline implements a decoupled, event-driven Medallion Architecture to securely ingest, process, and enrich data:

- **Bronze (S3 raw/):** Untouched legacy .xlsm files ingested via AWS AppFlow.
- **Silver (S3 extracted/):** Clean, structured intermediate JSON generated via cell-anchored extraction.
- **Gold (S3 clean/):** Schema-enforced, LLM-enriched Parquet datasets partitioned into 5 key business metrics (main, charges, wells, etc.).
## Fault Tolerance & Zero-Loss Data Redriving
- **The Challenge:** Processing unstructured legacy files means edge-case failures are inevitable. If a single file fails mid-pipeline, the entire bulk upload shouldn't need to be restarted.

- **The Solution:** Implemented dedicated Dead-Letter Queues (DLQ 1 & DLQ 2) on failure paths. A CloudWatch DLQ depth alarm triggers an SNS email notification the moment an item drops out. Failed payloads can be easily analyzed, fixed, and redriven directly from the DLQ without re-uploading the source files.
## Brittle Layouts vs. Visual Excel Forms
- **The Challenge:** Field supervisors used Excel as a visual form (merged cells, inconsistent rows, hand-aligned text labels). Standard pd.read_excel() scripts failed instantly across highly variable supervisor layouts.

- **The Solution:** Developed a dynamic anchor-based cell detection engine using openpyxl. Instead of reading rigid row/column coordinates, the script scans for known keyword text labels and extracts data from neighboring cells using relative directional offsets, wiping out silent parsing errors.
## Context Extraction from Handwritten Field Notes

- **The Challenge:** Critical operational and regulatory data (e.g., plug depths, cement volume, pressure tests) was trapped inside free-text paragraphs. Traditional regex was useless because it lacked the domain context to distinguish a successful test from a tool activation failure.

- **The Solution:** Prompt-engineered Claude Haiku on AWS Bedrock to handle domain-specific text evaluation. The model ingests daily free-text summaries and returns strongly typed, structured JSON classifying well events, separating operational parameters, and converting units automatically.
## Downstream Schema Drift in Parquet / Glue

- **The Challenge:** Independent parsing of 750 files causes implicit data-type mismatches (e.g., an empty column inferred as a float in one file but a string in another), causing PyArrow and Glue Crawlers to crash during aggregation.

- **The Solution:** Enforced a centralized Master Schema dictionary at the write layer of Lambda 2. Prior to saving as Parquet, all DataFrames are forcibly coerced: missing columns are injected as nulls, data types are strict-cast, and booleans are mapped cleanly to pandas' nullable Boolean type.
