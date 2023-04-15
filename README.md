# dlt-ingestion-from-streaming-src
## Basic idea
The basic idea for this project is to use the Databricks Delta Live Tables (DLT) feature to ingest data from a streaming source.
The dataset used is the NYC taxi data in Databricks that ships with every Databricks workspace, located in the 'samples' schema in the dbfs Databricks file system.

The DLT feature is built on top of Spark's structured streaming engine. It is a very useful feature for incrementally extracting data from a streaming source and loading it into a sink, and it is also possible to do some transformations in between the DLT source and sink. However, there are limitations to the sort of transformations that are possible directly in DLT pipelines. In particular, it is not possible to do non-time based aggregations directly in a DLT pipeline. So this project explores an approach on how to do this.
### The streaming data source
As mentioned previously, the dataset used in this project is the NYC taxi data. This is a static dataset. Therefore, in the [Streaming Source Simulation Notebook](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/simulate_streaming_source.ipynb), I copy the entire dataset into a Spark dataframe, and iteratively get the oldest pick-up time and write it into a delta table before deleting the record from the dataframe. In this way, the delta table acts as the streaming source.
### DLT pipelines
In the [Continuous DLT Pipeline Notebook](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/dlt_code/nyctaxi_continuous_dlt_code.ipynb), data from the streaming source is read into a dataframe, some minor transformations are performed, and then written into a delta table. Because the transformations are performed as part of the DLT pipeline, only a simple transformation is performed, in this case, the trip duration is calculated by subtracting two timestamps. The json configuration for the continuous DLT pipeline is available in this [json file](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/workflow_configs/nycdata_cont_dlt_pipeline.json).

For the purpose of implementing more complex transformations, a different strategy is employed. Rather than implementing complex transformations inside a continuous DLT pipline, a job scheduled to run every 5 minutes consisting of 2 tasks ([json configuration](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/workflow_configs/aggregate_latest_batch_nyctaxi_ingested_data_job.json)) is created:
1. The first task is a triggered DLT pipeline ([DLT code](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/dlt_code/nyctaxi_triggered_dlt_code.ipynb) and [json configuration](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/workflow_configs/nycdata_trigg_dlt_pipeline.json)) that reads the dataset from the continuous DLT's sink and writes it to a staging table.
2. The [second task](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/agg_latest_batch_by_zip_codes.ipynb) reads data from the staging table and does some more complex operations that calculate average trip duration aggregated geographically before writing to a delta table. This [geographical aggregation](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/function_definitions.ipynb) is accomplished by creating bins on the zip codes of the pick-up location.

This scheduled job aggregates data only for the latest batch, the latest batch being the subset of records written to the continuous DLT pipeline's sink since the last time this sink was last consumed by the triggered DLT pipeline. It does this by truncating the staging table after the geographical aggregation is performed, so that on the subsequent job trigger, the staging table contains only the latest batch of data.

### Historical aggregations
Another scheduled job performs the same geographical aggregations mentioned previously, but aggregates the whole dataset from the beginning of time rather than just the latest batch. In this case, the job does not include a DLT pipeline. Therefore, data from the continuous pipeline's sink is read in a [regular notebook](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/agg_historical_data_by_zip_codes.ipynb) and then persisted into a delta table. The job frequency here is set to 15 minutes ([json configuration](https://github.com/utlond/dlt-ingestion-from-streaming-src/blob/main/workflow_configs/aggregate_historical_nyctaxi_ingested_data_job.json)). In practice, a job that recomputes the entire dataset would probably run much less frequently than this.
