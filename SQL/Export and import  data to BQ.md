
# import 
> Identity and Access Management (IAM)
	如要將資料載入 BigQuery，您需要具備執行載入工作，以及將資料載入 BigQuery 資料表和分區的 IAM 權限。如要從 Cloud Storage 載入資料，您也需要 IAM 權限，才能存取包含資料的值區

IAM 權限的設定 `CRUD`
- `bigquery.tables.create`
- `bigquery.tables.updateData`
- `bigquery.tables.update`
- `bigquery.jobs.create`

## SQL call cloud storage import data

```
LOAD DATA OVERWRITE mydataset.mytable
(x INT64,y STRING)
FROM FILES (
  format = 'CSV',
  uris = ['gs://bucket/path/file.csv']);
```


```

from google.cloud import bigquery

# Construct a BigQuery client object.
client = bigquery.Client()

# TODO(developer): Set table_id to the ID of the table to create.
# table_id = "your-project.your_dataset.your_table_name"

job_config = bigquery.LoadJobConfig(
    schema=[
        bigquery.SchemaField("name", "STRING"),
        bigquery.SchemaField("post_abbr", "STRING"),
        bigquery.SchemaField("date", "DATE"),
    ],
    skip_leading_rows=1,
    time_partitioning=bigquery.TimePartitioning(
        type_=bigquery.TimePartitioningType.DAY,
        field="date",  # Name of the column to use for partitioning.
        expiration_ms=7776000000,  # 90 days.
    ),
)
uri = "gs://cloud-samples-data/bigquery/us-states/us-states-by-date.csv"

load_job = client.load_table_from_uri(
    uri, table_id, job_config=job_config
)  # Make an API request.

load_job.result()  # Wait for the job to complete.

table = client.get_table(table_id)
print("Loaded {} rows to table {}".format(table.num_rows, table_id))
```

# export

bigquery 中可以直接透過query 把東西送到cloud storage 下面的bucket

```

EXPORT DATA
  OPTIONS (
    uri = 'gs://bucket/folder/*.csv',
    format = 'CSV',
    overwrite = true,
    header = true,
    field_delimiter = ';')
AS (
  SELECT field1, field2
  FROM mydataset.table1
  ORDER BY field1
);


```
