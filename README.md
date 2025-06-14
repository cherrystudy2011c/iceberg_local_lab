You're in luck! Learning Apache Iceberg concepts locally without an AWS account is totally feasible and a great way to get hands-on. The key is to simulate an S3-compatible object storage service on your local machine. The most popular and well-supported tool for this is **MinIO**.

Here's a step-by-step guide to set up a local environment for Apache Iceberg using MinIO as your S3-compatible storage and Apache Spark as your processing engine.

## Core Concepts Explained:

* **Apache Iceberg:** A table format for large analytics datasets. It provides SQL-like table capabilities (ACID transactions, schema evolution, time travel, hidden partitioning) over data files stored in an object store (like S3, or in our case, MinIO).
* **MinIO:** An open-source, S3-compatible object storage server. It allows you to run a local instance that behaves just like AWS S3, providing buckets and object storage.
* **Apache Spark:** A powerful, open-source distributed processing engine. We'll use Spark (specifically PySpark, its Python API) to interact with Iceberg tables stored in MinIO.
* **Catalog:** Iceberg needs a "catalog" to store the metadata about your tables (schema, partition information, snapshots). For a local setup, a simple in-memory catalog or a REST catalog (often used with Nessie for branching/merging) or even a local Hive Metastore are common choices. We'll start with a simple in-memory Spark catalog for simplicity, and then you can explore more advanced catalogs.

## Step-by-Step Process with Example:

We'll use Docker Compose to simplify the setup, as it allows us to run MinIO and Spark in isolated containers.

**Prerequisites:**

1.  **Docker Desktop:** Install Docker Desktop (includes Docker Engine and Docker Compose) on your system. This is available for Windows, macOS, and Linux.
2.  **Basic understanding of Docker:** Familiarity with Docker commands (`docker run`, `docker-compose up`) will be helpful but not strictly necessary for this guide.

---

### Step 1: Set Up Your Project Directory

Create a new directory for your project and navigate into it:

```bash
mkdir iceberg_local_lab
cd iceberg_local_lab
```

---

### Step 2: Create `docker-compose.yml`

This file will define our services: MinIO (for S3 storage) and a Spark environment pre-configured with Iceberg.

Create a file named `docker-compose.yml` in your `iceberg_local_lab` directory with the following content:

```yaml
version: "3.8"

services:
  minio:
    image: minio/minio
    container_name: minio
    ports:
      - "9000:9000" # MinIO API port
      - "9001:9001" # MinIO Console port
    environment:
      MINIO_ROOT_USER: minioadmin # Your desired access key
      MINIO_ROOT_PASSWORD: minioadmin # Your desired secret key (at least 8 chars)
      MINIO_REGION: us-east-1 # Important for Iceberg to recognize it as S3
    volumes:
      - ./minio_data:/data # Mount a local directory for MinIO data persistence
    command: ["server", "/data", "--console-address", ":9001"] # Start MinIO server with console
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  spark:
    image: tabulario/spark-iceberg:latest # A pre-built Spark image with Iceberg support
    container_name: spark-iceberg
    depends_on:
      - minio # Ensure MinIO starts before Spark
    ports:
      - "8888:8888" # Jupyter Notebook port
      - "8080:8080" # Spark UI port
    environment:
      AWS_ACCESS_KEY_ID: minioadmin # MinIO access key
      AWS_SECRET_ACCESS_KEY: minioadmin # MinIO secret key
      AWS_REGION: us-east-1 # Must match MINIO_REGION
      # Iceberg Catalog Configuration: Using a Spark-managed catalog
      SPARK_SUBMIT_ARGS: |
        --packages org.apache.iceberg:iceberg-spark-runtime-3.4_2.12:1.5.2,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262
        --conf spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog
        --conf spark.sql.catalog.spark_catalog.type=hadoop
        --conf spark.sql.catalog.spark_catalog.warehouse=s3a://warehouse/
        --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
        --conf spark.hadoop.fs.s3a.endpoint=http://minio:9000
        --conf spark.hadoop.fs.s3a.access.key=minioadmin
        --conf spark.hadoop.fs.s3a.secret.key=minioadmin
        --conf spark.hadoop.fs.s3a.path.style.access=true
        --conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem
        --conf spark.hadoop.fs.s3a.connection.ssl.enabled=false
    volumes:
      - ./notebooks:/home/iceberg/notebooks # Mount a local directory for Jupyter notebooks
      - ./data:/home/iceberg/data # Mount a local directory for sample data
    # Command to start Jupyter Lab with PySpark kernel
    command: bash -c "start-jupyter.sh --allow-root --port 8888 --ip 0.0.0.0"
```

**Explanation of the `docker-compose.yml`:**

* **`minio` service:**
    * `image: minio/minio`: Uses the official MinIO Docker image.
    * `ports`: Maps container ports to your host machine. `9000` is for the S3 API, `9001` is for the MinIO web console.
    * `environment`: Sets the root user credentials and the region.
    * `volumes`: Creates a persistent volume `minio_data` on your host, so your data in MinIO isn't lost when the container stops.
    * `command`: Starts the MinIO server.
* **`spark` service:**
    * `image: tabulario/spark-iceberg:latest`: This is a convenient Docker image that bundles Apache Spark with the necessary Iceberg JARs and even a Jupyter Notebook server.
    * `depends_on: - minio`: Ensures MinIO is running before Spark starts.
    * `ports`: Maps Jupyter (`8888`) and Spark UI (`8080`) ports.
    * `environment (SPARK_SUBMIT_ARGS)`: This is crucial for configuring Spark and Iceberg:
        * `--packages ...`: Includes the necessary Iceberg and Hadoop AWS client JARs. Make sure the Iceberg Spark runtime version (`3.4_2.12:1.5.2` here) is compatible with your Spark version (usually derived from the image). The `hadoop-aws` and `aws-java-sdk-bundle` are needed for S3A (MinIO) communication.
        * `spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog`: Registers a Spark-managed Iceberg catalog.
        * `spark.sql.catalog.spark_catalog.type=hadoop`: Specifies that the catalog uses Hadoop FileSystem for metadata, which works with S3-compatible storage.
        * `spark.sql.catalog.spark_catalog.warehouse=s3a://warehouse/`: This is where Iceberg will store its table metadata and data files within your MinIO bucket. We'll create a `warehouse` bucket in MinIO.
        * `spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions`: Enables Iceberg-specific SQL syntax.
        * `spark.hadoop.fs.s3a.endpoint=http://minio:9000`: Tells Spark to connect to your local MinIO instance. `minio` is the service name from `docker-compose.yml`.
        * `spark.hadoop.fs.s3a.access.key` and `secret.key`: MinIO credentials.
        * `spark.hadoop.fs.s3a.path.style.access=true`: Essential for MinIO compatibility.
        * `spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem`: Specifies the S3A file system implementation.
        * `spark.hadoop.fs.s3a.connection.ssl.enabled=false`: Disable SSL for local MinIO connection.
    * `volumes`: Mounts `notebooks` and `data` directories from your host.

---

### Step 3: Create Necessary Directories

Before starting Docker Compose, create the local directories mapped in `volumes`:

```bash
mkdir minio_data
mkdir notebooks
mkdir data
```

---

### Step 4: Start the Services

From your `iceberg_local_lab` directory, run:

```bash
docker-compose up -d
```

* `-d` runs the services in detached mode (in the background).
* This will pull the Docker images (if not already downloaded) and start the MinIO and Spark containers. This might take a few minutes the first time.

---

### Step 5: Verify Services and Create a MinIO Bucket

1.  **Check Docker Status:**
    ```bash
    docker-compose ps
    ```
    You should see `minio` and `spark-iceberg` in a "running" state.

2.  **Access MinIO Console:**
    Open your web browser and go to `http://localhost:9001`.
    Log in with the `MINIO_ROOT_USER` (`minioadmin`) and `MINIO_ROOT_PASSWORD` (`minioadmin`) you set in `docker-compose.yml`.

3.  **Create a Bucket in MinIO:**
    Once logged in, click the "Create Bucket" button (or the + icon) and create a bucket named `warehouse`. This is the bucket where Iceberg will store its data and metadata files, as configured in `spark.sql.catalog.spark_catalog.warehouse=s3a://warehouse/`.

---

### Step 6: Access Jupyter Notebook and Interact with Iceberg

1.  **Access Jupyter Lab:**
    Open your web browser and go to `http://localhost:8888`. You should see the Jupyter Lab interface.

2.  **Create a New Notebook:**
    In Jupyter Lab, click on "File" -> "New" -> "Notebook" and select the "PySpark" kernel.

3.  **Run PySpark Code for Iceberg:**

    Now, in your new Jupyter notebook cell, you can start writing PySpark code to interact with Iceberg.

    **Example: Creating an Iceberg Table, Inserting Data, and Querying**

    ```python
    # Ensure SparkSession is available (it should be automatically in the tabulario image)
    # from pyspark.sql import SparkSession
    # spark = SparkSession.builder.appName("IcebergLocal") \
    #     .getOrCreate()

    # Verify Spark and Iceberg extensions are loaded
    print("Spark Version:", spark.version)
    spark.sql("SELECT 1").show() # Just to ensure Spark is working

    # Define the catalog. We configured a 'spark_catalog' in docker-compose.yml
    catalog_name = "spark_catalog"

    # --- 1. Create an Iceberg Table ---
    print("\n--- Creating Iceberg Table ---")
    table_name = "sample_db.my_iceberg_table"

    spark.sql(f"""
    CREATE TABLE IF NOT EXISTS {catalog_name}.{table_name} (
        id INT,
        name STRING,
        event_time TIMESTAMP,
        city STRING
    ) USING iceberg
    PARTITIONED BY (days(event_time), city)
    TBLPROPERTIES ('format-version' = '2');
    """).show()

    print(f"Table '{table_name}' created successfully.")
    spark.sql(f"DESCRIBE {catalog_name}.{table_name}").show()

    # --- 2. Insert Data ---
    print("\n--- Inserting Data into Iceberg Table ---")
    data = [
        (1, "Alice", "2023-01-01 10:00:00", "New York"),
        (2, "Bob", "2023-01-01 11:00:00", "London"),
        (3, "Charlie", "2023-01-02 12:00:00", "New York"),
        (4, "David", "2023-01-02 13:00:00", "Paris"),
        (5, "Eve", "2023-01-03 14:00:00", "London")
    ]
    columns = ["id", "name", "event_time", "city"]
    df = spark.createDataFrame(data, columns)

    df.writeTo(f"{catalog_name}.{table_name}").append()
    print("Data inserted successfully.")

    # --- 3. Query Data ---
    print("\n--- Querying Data from Iceberg Table ---")
    spark.sql(f"SELECT * FROM {catalog_name}.{table_name} ORDER BY id").show()

    # --- 4. Perform an Update (ACID transaction) ---
    print("\n--- Updating Data ---")
    spark.sql(f"UPDATE {catalog_name}.{table_name} SET city = 'Singapore' WHERE name = 'Alice'").show()
    spark.sql(f"SELECT * FROM {catalog_name}.{table_name} ORDER BY id").show()

    # --- 5. Time Travel (Querying a previous snapshot) ---
    print("\n--- Time Travel: Querying Previous Snapshot ---")
    # Get all snapshots (versions) of the table
    spark.sql(f"SELECT * FROM {catalog_name}.{table_name}.snapshots").show()

    # You'll see multiple snapshots. The latest one is the current state.
    # To query a previous state, find the snapshot_id from the `.snapshots` output
    # For example, if the snapshot before the update has ID '1234567890123456789'
    # Replace the snapshot_id with one from your output before the update
    # Note: Snapshot IDs are long integers
    # Let's try to infer the previous snapshot ID. In a real scenario, you'd get it from the .snapshots table.
    # For simplicity, we'll assume the snapshot_id is available.
    # The `tabulario` image might have some helpers for this.
    # Let's just rollback for now for demonstration.

    # Instead of direct snapshot ID, let's look at history or rollback
    print("\n--- Table History ---")
    spark.sql(f"SELECT * FROM {catalog_name}.{table_name}.history").show()
    # You can see different versions and their corresponding snapshot IDs.

    # Rollback example: This is a DDL operation, not just a query.
    # You'd use the snapshot_id from the history output that represents the state before the update.
    # snapshot_id_before_update = <YOUR_SNAPSHOT_ID_HERE>
    # spark.sql(f"CALL {catalog_name}.system.rollback_to_snapshot('{table_name}', {snapshot_id_before_update})").show()
    # print(f"Rolled back to snapshot {snapshot_id_before_update}. Current data:")
    # spark.sql(f"SELECT * FROM {catalog_name}.{table_name} ORDER BY id").show()


    # --- 6. Schema Evolution (Add a column) ---
    print("\n--- Schema Evolution: Adding a new column ---")
    spark.sql(f"ALTER TABLE {catalog_name}.{table_name} ADD COLUMN age INT").show()
    print("Column 'age' added.")
    spark.sql(f"DESCRIBE {catalog_name}.{table_name}").show()

    # Insert data with the new column (older data will have NULL for new column)
    new_data = [
        (6, "Frank", "2023-01-04 09:00:00", "Berlin", 30),
        (7, "Grace", "2023-01-04 10:00:00", "Tokyo", 25)
    ]
    new_columns = ["id", "name", "event_time", "city", "age"]
    new_df = spark.createDataFrame(new_data, new_columns)
    new_df.writeTo(f"{catalog_name}.{table_name}").append()
    print("New data inserted with 'age' column.")

    spark.sql(f"SELECT * FROM {catalog_name}.{table_name} ORDER BY id").show()

    # --- 7. Deleting data ---
    print("\n--- Deleting Data ---")
    spark.sql(f"DELETE FROM {catalog_name}.{table_name} WHERE id = 5").show()
    print("Row with ID 5 deleted.")
    spark.sql(f"SELECT * FROM {catalog_name}.{my_iceberg_table} ORDER BY id").show()
    ```

---

### Step 7: Explore MinIO for Data Files

After running the Spark code, go back to your MinIO console (`http://localhost:9001`) and navigate into the `warehouse` bucket.

You will see:
* `_SUCCESS` files (from Spark writes)
* Directories like `data/` and `metadata/`
* Inside `data/`, you'll find Parquet files (your actual data).
* Inside `metadata/`, you'll see JSON files (`.metadata.json`, `.version.json`, `.avro` files) that Iceberg uses to track table schema, partitions, and snapshots. This is the core of how Iceberg provides its capabilities.

---

### Step 8: Clean Up (Optional)

When you're done, you can stop and remove the Docker containers and their associated data:

```bash
docker-compose down
```

* This will stop and remove the `minio` and `spark-iceberg` containers.
* To also remove the `minio_data` and `notebooks` volumes (and thus all your MinIO data and notebooks), you would manually delete those directories from your host file system.

## Important Considerations and Next Steps:

* **Catalog Choice:**
    * **Hadoop Catalog (used here):** Simple, uses your file system (MinIO in this case) to store metadata directly. Good for local testing, but not ideal for concurrent access from multiple Spark applications or other engines.
    * **REST Catalog (Nessie/Tabular):** Provides a central metadata service, enabling multi-engine access, branching, and merging capabilities. Often used with Project Nessie or Tabular. The `tabulario/spark-iceberg` image can be configured with a REST catalog as well if you uncomment the `rest` service in the `docker-compose.yml` and adjust Spark configs.
    * **Hive Metastore Catalog:** A common choice in traditional Hadoop environments.
    * **AWS Glue Catalog:** For actual AWS deployments, Glue Data Catalog is typically used.
* **Performance:** For a local setup, performance won't be like a distributed cluster, but it's sufficient for understanding concepts.
* **Error Handling:** Pay attention to the logs in your terminal when running `docker-compose up`. If containers fail to start, the logs will often indicate the reason.
* **Iceberg Version:** Ensure the Iceberg runtime JAR version specified in `SPARK_SUBMIT_ARGS` is compatible with your Spark version. The `tabulario/spark-iceberg` image aims to keep them compatible.
* **Data Types:** Iceberg supports a rich set of data types, including nested types.
* **Partitioning:** Experiment with different partitioning strategies (`days`, `hours`, `truncate`, `identity`) and observe how the data files are organized in MinIO.

This local setup provides a fantastic sandbox to experiment with Iceberg's features like schema evolution, time travel, and hidden partitioning, all without needing an AWS account!
