To set up and run a Unity Catalog project using Docker, follow this walkthrough:

## Phase 1: Set-up Unity Catalog Project Directory

### Step 1: Clone the Unity Catalog Repository
Run the following command to clone the Unity Catalog repository:
```bash
git clone https://github.com/unitycatalog/unitycatalog.git
```

### Step 2: Setup Dockerfile for macOS and Windows
Navigate to the cloned repository and create a Dockerfile named `unitycatalog.dockerfile` with the following content:

#### macOS Dockerfile
```dockerfile
# Use Ubuntu as base image
FROM ubuntu:20.04

# Set working directory in container
WORKDIR /app

# Install OpenJDK 17, curl, and other necessary tools
RUN apt-get update && \
    apt-get install -y openjdk-17-jdk curl gnupg

# Install sbt
RUN echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | tee /etc/apt/sources.list.d/sbt.list && \
    curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x99E82A75642AC823" | apt-key add && \
    apt-get update && \
    apt-get install -y sbt

# Copy necessary files
COPY . /app

# Build project
RUN sbt package

# Make sure scripts are executable
RUN chmod +x /app/bin/start-uc-server /app/bin/uc

# Add /app/bin to PATH
ENV PATH="/app/bin:${PATH}"

# Expose port app runs on
EXPOSE 8080

# Run Unity Catalog server
CMD ["/bin/bash", "/app/bin/start-uc-server"]
```

#### Windows Dockerfile
```dockerfile
# Use Ubuntu as base image
FROM ubuntu:20.04

# Set working directory in container
WORKDIR /app

# Install OpenJDK 17, curl, and other necessary tools
RUN apt-get update && \
    apt-get install -y openjdk-17-jdk curl gnupg dos2unix

# Install sbt
RUN echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | tee /etc/apt/sources.list.d/sbt.list && \
    curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x99E82A75642AC823" | apt-key add && \
    apt-get update && \
    apt-get install -y sbt

# Copy necessary files
COPY . /app

# Ensure scripts have LF line endings
RUN dos2unix /app/bin/start-uc-server && \
    dos2unix /app/bin/uc

# Build project
RUN sbt package

# Make sure scripts are executable
RUN chmod +x /app/bin/start-uc-server /app/bin/uc

# Add /app/bin to PATH
ENV PATH="/app/bin:${PATH}"

# Expose port app runs on
EXPOSE 8080

# Run Unity Catalog server
CMD ["/bin/bash", "/app/bin/start-uc-server"]
```

## Phase 2: Running Unity Catalog in Docker

### Step 1: Build the Docker Image
Run the following command to build the Docker image:
```bash
docker build -t unitycatalog -f unitycatalog.dockerfile .
```

### Step 2: Run the Docker Container
Run the following command to start a new container:
```bash
docker run -d --name unitycatalog -p 8080:8080 unitycatalog
```

### Step 3: Verify the Container is Running
Check the logs to verify the container is running:
```bash
docker logs unitycatalog
```
*You should see the Unity Catalog logo and other startup messages.*

```
###################################################################
#  _    _       _ _            _____      _        _              #
# | |  | |     (_) |          / ____|    | |      | |             #
# | |  | |_ __  _| |_ _   _  | |     __ _| |_ __ _| | ___   __ _  #
# | |  | | '_ \| | __| | | | | |    / _` | __/ _` | |/ _ \ / _` | #
# | |__| | | | | | |_| |_| | | |___| (_| | || (_| | | (_) | (_| | #
#  \____/|_| |_|_|\__|\__, |  \_____\__,_|\__\__,_|_|\___/ \__, | #
#                      __/ |                                __/ | #
#                     |___/               v0.1.0-SNAPSHOT  |___/  #
###################################################################
```

## Phase 3: Initial Setup

### Step 1: Create and List Catalogs
Create a new catalog and list all catalogs:
```bash
docker exec -it unitycatalog uc catalog create --name my_local_catalog
docker exec -it unitycatalog uc catalog list
```

### Step 2: Create and List Schemas
Create a new schema within the catalog and list all schemas:
```bash
docker exec -it unitycatalog uc schema create --catalog my_local_catalog --name my_schema
docker exec -it unitycatalog uc schema list --catalog my_local_catalog
```

### Step 3: Create a Delta Table

#### Dependencies
Ensure you have the following Python packages installed:
- `deltalake`
- `duckdb`
- `mimesis`

#### Code to Generate and Write Data to Delta Table
```python
import duckdb
from deltalake import write_deltalake, DeltaTable
import os
from mimesis import Person
from mimesis.locales import Locale

# Generate 1000 records
person = Person(Locale.EN)
records = []
for index in range(1, 1001):
    record = {
        "Index": index,
        "User_Id": person.identifier(),
        "First_Name": person.first_name(),
        "Last_Name": person.last_name(),
        "Sex": person.gender(),
        "Email": person.email(),
        "Phone": person.telephone(),
        "Date_of_birth": person.birthdate().isoformat(),
        "Job_Title": person.occupation()
    }
    records.append(record)

# Create DuckDB table and insert records
con = duckdb.connect()
con.execute("""
    CREATE TABLE users (
        "Index" INTEGER,
        "User_Id" VARCHAR,
        "First_Name" VARCHAR,
        "Last_Name" VARCHAR,
        "Sex" VARCHAR,
        "Email" VARCHAR,
        "Phone" VARCHAR,
        "Date_of_birth" DATE,
        "Job_Title" VARCHAR
    )
""")

insert_query = "INSERT INTO users VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)"
con.executemany(insert_query, [(record["Index"], record["User_Id"], record["First_Name"], record["Last_Name"], record["Sex"], record["Email"], record["Phone"], record["Date_of_birth"], record["Job_Title"]) for record in records])

# Convert DuckDB table to DataFrame and write to Delta table
duck_df = con.execute("SELECT * FROM users").fetchdf()
delta_table_path = ".../test_delta_table/" # modify this to your desired directory
write_deltalake(delta_table_path, duck_df, mode='append')

# Verify Delta table directory contents and check metadata
print(os.listdir(delta_table_path))

result = con.execute(f"SELECT * FROM delta_scan('{delta_table_path}')").fetchdf()
print(result)

delta_table = DeltaTable(delta_table_path)
print(delta_table.history())
```

### Step 4: Register the Delta Table
Now that the Delta table is created, register it with Unity Catalog: 
*Note: Update storage location to the path of your Delta Table*
```bash
docker exec -it unitycatalog uc table create --full_name my_local_catalog.my_schema.sample_delta_table --columns "Index INT, User_Id STRING, First_Name STRING, Last_Name STRING, Sex STRING, Email STRING, Phone STRING, Date_of_birth DATE, Job_Title STRING" --format DELTA --storage_location file:///C:/.../.../.../test_delta_table
```

### Step 5: Read the Table *(Delta Format Only)*
Read the table to verify its contents:
```bash
docker exec -it unitycatalog uc table read --full_name my_local_catalog.my_schema.sample_delta_table
```

You have now successfully set up Unity Catalog to run on Docker, created a catalog, schema, generated sample Delta Table, registered the sample Delta table, and read it the Delta Table back using Unity Catalog native capabilities.
