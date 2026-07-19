# HDB Resale Data ETL

I used Python and PySpark because these are the tools I normally use for data processing. The current dataset is not very large, but the same logic can also run for bigger files.

## Files

- `HDB_ETL.ipynb` - complete ETL notebook
- `architecture.png` - AWS architecture
- `requirements.txt` - Python packages
- `.gitignore` - files not to upload
- `README.md` - approach and assumptions

## Solution flow

1. Download the source CSV files from data.gov.sg.
2. Keep the downloaded files unchanged in the raw folder.
3. Read and combine the files using PySpark.
4. Profile the main columns.
5. Validate date, town, flat type, flat model and storey range.
6. Recalculate remaining lease as of the run date.
7. Remove duplicate composite keys and keep the higher resale price.
8. Detect unusual resale prices using the IQR method within town and flat type.
9. Create the Resale Identifier.
10. Hash the identifier using SHA-256.
11. Write raw, cleaned, transformed, failed and hashed outputs.

## Output folders

The notebook creates `data/raw` and the following folders under `data/output`:

- `cleaned`
- `transformed`
- `failed`
- `hashed`

## Main assumptions

- Only records from January 2012 to December 2016 are included.
- The source month is in `yyyy-MM` format.
- The lease is assumed to be 99 years from `lease_commence_date`.
- Remaining lease is calculated using the notebook run date.
- The duplicate key is all source columns except `resale_price`.
- For a duplicate key, the highest resale price is kept.
- Invalid and duplicate records are written to failed output with a reason.
- Price anomalies are identified using IQR by town and flat type.
- Anomalies are marked but not removed because a high or low price may still be valid.
- SHA-256 is used because it is one-way and supported directly by Spark.

## Resale Identifier

The identifier format is:

`S + 3 block digits + 2 average price digits + month + first town character`

Example: `S0192301A`

The average resale price is grouped by year-month, town and flat type.

## Proposed AWS architecture

The design uses tools that I have worked with where possible:

- Apache Airflow for scheduling, retries, alerts and SLA monitoring
- AWS Glue PySpark job for processing
- Amazon S3 for raw and processed data
- AWS Glue Data Catalog for table metadata used by Athena
- Amazon Athena for SQL queries
- Tableau using the Athena driver
- IAM for role-based access
- Private VPC and VPC endpoints for private AWS access where possible

The ingestion components run inside a private VPC. Data from data.gov.sg is accessed through a NAT Gateway using HTTPS.

Airflow is used to schedule the pipeline and handle retries, task failures and alerts.

AWS Glue runs the PySpark ETL processing and writes the output datasets to Amazon S3.

S3 access is private through an S3 VPC endpoint where possible.

Glue Data Catalog stores the table metadata required by Athena.

Tableau connects to Athena using the Athena driver. An Athena VPC endpoint can be used where supported so that traffic remains private within AWS.

## Security

- IAM roles control access to S3, Glue and Athena.
- Processing services are placed in private subnets.
- S3 and Athena endpoints are used where possible.
- Access to data.gov.sg is outbound-only through the NAT Gateway.

## Monitoring

Airflow is used for:

- task retries
- failure handling
- email alerts
- SLA monitoring

## Run instructions

1. Install Java 11 or Java 17.
2. Install packages:

```bash
pip install -r requirements.txt
```

3. Start Jupyter:

```bash
jupyter notebook HDB_ETL.ipynb
```

4. Run the notebook cells from top to bottom.

The notebook tries to download CSV resources from the data.gov.sg collection page. If the page structure changes, place the required CSV files directly in `data/raw` and continue from the Spark read section.

The original technical test PDF is not included in this repository.
