# Data Pipeline with Astronomer, DBT, Soda and Metabase using Airflow on GCP

This project demonstrates how to build and automate a data pipeline using DAGs in Airflow and load the transformed data to BigQuery.  Various tools are used, including Astro (a Docker wrapper around Airflow), DBT (for data modeling and SQL reporting), Soda (for data quality checks), Metabase (a containerized data visualization tool), and Google Cloud Platform for data storage.



# Project Goals - To try new tools and learn!
0.  The data engineering field is rapidly evolving with new tools and technologies.  The best way to stay current is through hands-on experience. This project utilizes several new tools to create an effective and robust data pipeline.  Integrating these tools presents challenges (dependencies can be complex), but this complexity allows for a deeper understanding of the architecture.



1. Data Ingestion - Create a data ingestion pipeline to extract new incoming data into the GCP BigQuery.
2. Data Quality - Use Soda to create highly customized data quality checks using YAML files.
3. Data Transformation - Use DBT to perform data modeling and transform the data into a star schema.
4. Data Loading - Use the data pipeline to load the extracted and transformed data into GCP BigQuery.
5. Data Reporting/Analytics - Use Metabase to create dashboards for reporting or analytics purposes.

# Data Architecture

The architecture (data flow) used in this project employs various tools and languages.

<p align="center">
  <img height="600" src="https://github.com/chayansraj/Orchestrate-Python-ETL-Pipeline-with-DBT-using-Airflow-on-GCP/assets/22219089/c0e1b490-510d-43aa-b34c-9e6ea3c750a5">
  <h6 align = "center" > Source: Original Author </h6>
</p>


# Dataset Used 
The Online Retail II dataset contains transactions for a UK-based online retailer between 01/12/2009 and 09/12/2011.  The company primarily sells unique giftware, with many wholesale customers.

Dataset link: [Online Retail Dataset](https://www.kaggle.com/datasets/mashlyn/online-retail-ii-uci)

# Tools and technologies used in this project

1. **BigQuery (GCP)** - A fully managed, serverless data warehouse and analytics platform offered by Google Cloud.
2. **SODA** - A data quality framework and set of tools designed to automate data validation and quality checks.
3. **DBT** - An open-source command-line tool and modeling framework for modern data analytics and engineering.
4. **Astro CLI** - The command-line interface for data orchestration within the Astronomer suite.
5. **Metabase** - An open-source business intelligence (BI) and data analytics tool.
6. **Visual Code Studio** - A free, open-source code editor developed by Microsoft.
7. **Docker** - A platform for packaging and distributing applications as lightweight containers.
8. **Git Version Control** - A distributed version control system.


# Implementation

* **Step 1** - Install Astro CLI and create a working Airflow environment with a GCP connection.  This requires Docker and enabled Hyper-V. After installing [Astro CLI](https://docs.astronomer.io/astro/cli/install-cli?tab=windowswithwinget#install-the-astro-cli), run:
  ```
  astro dev init
  ```

This command will create an Airflow environment.  You'll need a Google Cloud account and project, along with service accounts (Storage Admin and BigQuery Admin). Download the key and place it in your 'gcp' folder under the 'include' folder. **Do not share it.**

> A connection is a two-way street

Install the Airflow Google provider:
```
apache-airflow-providers-google
```
Start the Docker container:
```
astro dev start
```
Access the Airflow UI at localhost:8080 to create a GCP connection.

* **Step 2** - Create the data pipeline using DAGs in Airflow.  A DAG represents a workflow of tasks.

    ```
    from airflow.decorators import (
    dag,
    task,)
    
    # DAG and task decorators for interfacing with the TaskFlow API
    from datetime import datetime
    from airflow.providers.google.cloud.transfers.local_to_gcs import LocalFilesystemToGCSOperator
    from airflow.providers.google.cloud.operators.bigquery import BigQueryCreateEmptyDatasetOperator
    
    upload_csv_to_gcs = LocalFilesystemToGCSOperator(
    task_id='upload_csv_to_gcs',
    src='/usr/local/airflow/include/dataset/online_retail.csv',
    dst='raw/online_retail.csv',
    bucket='online_retail_database',
    gcp_conn_id='gcp',
    mime_type='text/csv')
    ```

Test a task:
    ```
    astro dev bash -> airflow tasks test <DAG name> <Task Name> <Start Date>
    ```

    * Second Task: Create a BigQuery dataset:
      ```
      create_retail_dataset = BigQueryCreateEmptyDatasetOperator(
      task_id='create_retail_dataset',
      dataset_id='retail',
      gcp_conn_id='gcp',)
      ```

    * Third Task: Load data from GCS to BigQuery:
      ```
      gcs_to_raw = aql.load_file(
  
          task_id='gcs_to_raw',
          input_file=File(
              'gs://online_retail_database/raw/online_retail.csv',
              conn_id='gcp',
              filetype=FileType.CSV,
          ),
          output_table=Table(
              name='raw_online_retail',
              conn_id='gcp',
              metadata=Metadata(schema='retail')
          ),
          use_native_support=False,      )
      ```

* **Step 3** - Use Soda to run data quality checks. Register at [soda.io](https://soda.io) to obtain connection credentials.  A simple data type check example:

    ```
    soda scan -d retail -c include/soda/configuration.yml include/soda/checks/sources/raw_online_retail.yml
    ```

To integrate Soda checks into the DAG, use a Python external task.  Modify the Dockerfile to include the Soda library:

```
RUN python -m venv soda_venv && source soda_venv/bin/activate && \
    pip install --no-cache-dir soda-core-bigquery==3.0.45 &&\
    pip install --no-cache-dir soda-core-scientific==3.0.45 && deactivate
```

* **Step 4** - Use DBT to transform the data.  This involves creating a star schema with fact and dimension tables.  Create a virtual environment for DBT:

  ```
  RUN python -m venv dbt_venv && source dbt_venv/bin/activate && \
    pip install --no-cache-dir dbt-bigquery==1.5.3 && deactivate
  ```

Integrate DBT models into the DAG using the `DbtTaskGroup`.

* **Step 5** - Repeat Step 3 to run data quality checks on the transformed data.

```
@task.external_python(python='/usr/local/airflow/soda_venv/bin/python')
def check_transform(scan_name='check_transform', checks_subpath='transform'):
    from include.soda.check_function import check

    return check(scan_name, checks_subpath)
```

Chain tasks in the DAG:

```
chain(
    upload_csv_to_gcs,
    create_retail_dataset,
    gcs_to_raw,
    check_load(),
    transform,
    check_transform(),
    report,
    check_report(),
)
```

* **Step 6** - Create a Metabase dashboard. Use a `docker-compose` override file:

  ```
  version: "3.1"
  services:
  metabase:
    image: metabase/metabase:v0.46.6.4
    volumes:
      - ./include/metabase-data:/metabase-data
    environment:
      - MB_DB_FILE=/metabase-data/metabase.db
    ports:
      - 3000:3000
    restart: always
  ```

Access Metabase at localhost:3000.


**The End**

**Maintainer:**

Sai Kiran Reddy Kondreddygari
Data Engineer
kondreddygarisaikiranreddy18@gmail.com
